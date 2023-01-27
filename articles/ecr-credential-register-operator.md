---
title: "Amazon ECRに保存されたイメージでDockerOperatorを動かす"
emoji: "😺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['airflow', 'aws', 'ecr', 'python', 'docker']
published: true
---
# 概要
AirflowのDockerOperatorでAWS ECRをソースリポジトリとした際は単純にリポジトリを指定するだけではうまく行かなかったので、その解決策を記載しました。

# 検討事項
AWS ECRを利用する際には認証情報を渡しますが、以下の制約があり渡し方に苦労しました。
* ECRの認証情報は12時間で認証情報が変更されるので、事前にAirflowのConnectionsに保存しても、12時間が経過すると認証できなくなる
* DockerOperatorは引数にユーザ・パスワードを取っていない

# 解決方法
私が探した限りでは既存のOperatorやHookではAirflowのConnectionsに情報を書き込む機能が提供されておらず、途方に暮れていたところ以下のサイトを発見しました。
https://www.lucidchart.com/techblog/2019/03/22/using-apache-airflows-docker-operator-with-amazons-container-repository/

上記サイトでは、Airflow Cliを定期的なバッチで動かして、Airflow Connectionsを最新化するものですが、別途定期的なバッチを動かす仕組みを管理したくなかったので、DockerOperatorを動かす前にAirflow Connectionsを最新化する方法を取りました。
（今考えれば、AirlfowのPythonOperatorで定期実行すればよかったかも）

具体的にはAirflow Connectionsを最新化するCustom Operatorを作成し、チームのルールとしてDockerOperatorの呼び出し前にCustom Operatorを呼び出してAirflow Connectionsを最新化する方法を取りました。
（これも今考えれば、DockerOperatorを継承したCustom Operatorを作成して、透過的にAirflow Connectionsを最新化する方法をありだったかも）

最終的に作成したCustom Operatorは以下の通りです。
gistにも同じものを公開してます。
https://gist.github.com/shin1103/40e405aa5e8925b59b16d2ab110a3a70

```python
import traceback
from typing import TYPE_CHECKING, Any, List, Optional

from airflow.exceptions import AirflowException
from airflow.models import BaseOperator

if TYPE_CHECKING:
    from airflow.utils.context import Context


class EcrCredentialRegisterOperator(BaseOperator):
    """ register ECR credential to Airflow connections.
    """

    def __init__(self, *, conn_id_name: str, region_name: str, register_ids: List[str], **kwargs,) -> None:
        """ Constructor. conn_id_name, region_name, registry_ids, task_id(this is BaseOperator' arg) is mandatory.

        Args:
            conn_id_name (str): connection id name to register Airflow connections
            region_name (str): ECR's region name
            registry_ids List[str]: AWS accountID(12-figure number). The argument is of type list, but the list is assumed to have a single element.
        """

        super().__init__(**kwargs)
        self.conn_id_name = conn_id_name
        self.region_name = region_name
        self.registry_ids = register_ids

    def __set_ecr_credential_to_airflow(self) -> None:
        """ register ECR credential to Airflow connections.
            this methond refer the site below. In this site, some commandline options is obsolate, I edited some options.
            https://www.lucidchart.com/techblog/2019/03/22/using-apache-airflows-docker-operator-with-amazons-container-repository/
        """

        import base64
        import subprocess

        import boto3

        # Get ECR Credentials
        ecr = boto3.client('ecr', region_name=self.region_name)
        response = ecr.get_authorization_token(registryIds=self.registry_ids)
        username, password = base64.b64decode(
            response['authorizationData'][0]['authorizationToken']
        ).decode('UTF-8').split(':')
        registry_url = response['authorizationData'][0]['proxyEndpoint']

        # Delete existing docker connection
        airflow_del_cmd = 'airflow connections delete {}'.format(
            self.conn_id_name)
        process = subprocess.Popen(
            airflow_del_cmd.split(), stdout=subprocess.PIPE)
        del_output, del_error = process.communicate()

        # Add existing docker connection
        airflow_add_cmd = 'airflow connections add --conn-type docker --conn-host {} --conn-login {} --conn-password {} {}'.format(
            registry_url, username, password, self.conn_id_name)
        process = subprocess.Popen(
            airflow_add_cmd.split(), stdout=subprocess.PIPE)
        add_output, add_error = process.communicate()

    def execute(self, context: 'Context') -> Optional[List[Any]]:
        """ Override BaseOperator
        """

        try:
            self.__set_ecr_credential_to_airflow()
        except Exception as e:
            print(traceback.format_exc())
            raise AirflowException(
                'Fail to register ECR credential to Airflow connections.') from e
```