---
title: "Airflowでdbt Coreを動かした話"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["dbt","airflow"]
published: true
published_at: 2022-12-21 01:00 # 未来の日時を指定する---
---

この記事は[dbt Advent Calendar 2022](https://qiita.com/advent-calendar/2022/dbt)の12月21日の記事です。

# 概要
私の所属しているチームではAirflowでジョブを動かしています。新たにdbtのジョブをAirflow実行する際にチームの前提事項をもとにどのように動かしたらいいのか検討した結果を記載します。

# 前提
## サマリ
* Airflowはオンプレミスで実行しており、柔軟なリソース増減はできない
* チームは複数のプロジェクトで構成されており、Airflowを共用
* チームでAirflowの基本機能は提供するが、実際のコンピューティングリソースはプロジェクトが用意する
* dbt Cloudは使えない（もし、使えれば[dbt Cloud Operator](https://airflow.apache.org/docs/apache-airflow-providers-dbt-cloud/stable/operators.html)が存在するため、以下の議論すべて不要になったかも）

## Airflow
私の所属するチームではAirflowのSaaSサービスを使用できないため、オンプレミスにAirflowを構築しています。

このAirflowを複数のプロジェクトで共有しているため、あるプロジェクトが高負荷の処理を行った場合に他プロジェクトに影響がないようにする必要がありました。
dbt Cloudが使えれば実際の処理はすべてdbt Cloudでやってくれるため、特段の考慮は不要でしたが、dbt Coreを使う前提なのでワーカーリソースは各プロジェクトで準備してもらうことにしました。
その方法として、[DockerOperator](https://airflow.apache.org/docs/apache-airflow-providers-docker/stable/_api/airflow/providers/docker/operators/docker/index.html)を使うことにし、各プロジェクトが準備したサーバーにてDockerイメージを実行してもらうことにしました。

雑になりますが以下がイメージ図です。
![](/images/airflow-with-dbt-core/airflow-architecute.drawio.png)

# 検討事項
## ベースのDockerイメージの作成
DockerOperatorで実行するDockerイメージはチームで統一したいため、共通のDockerイメージを探しました。
[Docker Hub](https://hub.docker.com/r/fishtownanalytics/dbt)に公式のイメージがありましたが、DEPRECATEDとなっており、[ghcr.io](https://github.com/orgs/dbt-labs/packages?visibility=public)に最新版があるとのことでした。
ここからダウンロードしてもよいのですが、ソースコード上に[Dockerfile](https://github.com/dbt-labs/dbt-core/blob/main/docker/Dockerfile)も公開されていたため、このDockerfileをチームの環境に合わせて編集したものを使用しました。
ビルド時はBUILD KITを使っているのでBUILD KITを指定しないとエラーになりました。
```bash
DOCKER_BUILDKIT=1 docker build -t 198.168.100.50/my_project/dbt_base:v1.2.3 --target dbt-postgres .
```
## 実行用Dockerイメージの作成

公式のDockerfileをみるとWORK_DIRが`/usr/app/dbt/`に設定されているため、WORK_DIR以下に保存するように記述。my_project以下に実際のdbtのコード、後述に記載したprofiles.ymlをベースイメージに追加しています。
![](/images/airflow-with-dbt-core/directory.png)  

```dockerfile
FROM 198.168.100.50/my_project/dbt_base:v1.2.3
ARG APP_PASS="/usr/app/dbt/"

ADD ./my_project $APP_PASS
COPY ./profiles.yml $APP_PASS
```

## 接続情報の受け渡し
接続情報はソースコードには記載せず、AirflowのConnectionsに記載し、DockerOperator実行時にVariableから情報を取得し環境変数に設定します。
### profiles.ymlの設定
前述したprofiles.ymlのprodにはユーザ名とパスワードは環境変数を参照するようにしました。ローカルの開発ではprofiles.ymlは使用せずデフォルトのprofiles.ymlを使って開発者が自由に設定してもらっています。
```yml
config:
  send_anonymous_usage_stats: false
my_project:
  outputs:
    prod:
      type: postgres
      threads: 1
      host: 192.168.100.150
      port: 5432
      user: "{{ env_var('MY_PROJECT_USERNAME') }}"
      password: "{{ env_var('MY_PROJECT_PASSWORD') }}"
      dbname: my_app
      schema: my_function
```
### DockerOperatorの設定
以下のことを注意して記載しました。
* AirflowのConnectionsを取得しDockerイメージ実行時に環境変数に設定
* targetをprodに設定
* profileをDockerイメージに入れたものファイルを参照するように変更

```python
from airflow.hooks.base_hook import BaseHook
conn = BaseHook.get_connection('MY_PROJECT_CONNECTION') 

DockerOperator(
        task_id="execute_dbt_docker",
        container_name=f"dbt_test",
        image="198.168.100.50/my_project/my_app:v1.0",
        force_pull=True,
        auto_remove=True,
        environment=dict(
            MY_PROJECT_USERNAME=conn.login,
            MY_PROJECT__PASSWORD=conn.password
        ),
        command=" run --profiles-dir=/usr/app/dbt/ --target=prod ",
        docker_url=f"tcp://198.168.100.100:2375",
        cpus=1,
        mem_limit="1g",
        mount_tmp_dir=False
```

# 課題
DockerOperatorで指定した`command`の実行中にエラーが発生した場合は`on_failure_callback`でコールバックを受け取り、Slack等への通知ができるのですが、`docker_url`で指定したサーバとの通信が失敗したり、`image`で指定したコンテナリポジトリからのイメージ取得に失敗した場合は`on_failure_callback`でコールバックを拾えないため、どのような形でエラーを検知するのかが課題となっています。