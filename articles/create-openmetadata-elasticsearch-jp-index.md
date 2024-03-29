---
title: "OpenMetadataでElasticSearchの日本語インデックスを作成する"
emoji: "🗂"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["elasticsearch", "openmetadata"]
published: true
---
# はじめに
社内でデータカタログ導入に向けて調査を進めています。DataHubやAmundsenなどいくつか触ってみましたが、OpenMetadataが日本語UIが提供されていたり、kuromojiを使ったElasticSearchの日本語インデックスを張れるなど、日本語フレンドリーなところがとても気に入っています。
ElasticSearchの日本語インデックス作成は簡単な設定でできますので、このブログでご紹介します。

# 前提条件
1. OpenMetadataのバージョンは1.3
当初1.2で検証を始めましたが、以下Issueにある通り、日本語インデックスの設定ファイルが正しくないため、うまく動きませんでした。
https://github.com/open-metadata/OpenMetadata/issues/14117
上記修正は1.3で取り込まれていますので、1.3を使うようにしてください。

2. Docker上でOpenMetadataを構築
一番簡単に構築可能なDockerベースの以下クイックスタートの手順をベースに説明をします。ですので、本記事ではクイックスタートの要件を満たしており、DockerやDocker Composeがインストールされているものとします。
https://docs.open-metadata.org/v1.3.x/quick-start/local-docker-deployment

# 設定方法
1. kuromojiをインストールしたElasticSearchのDockerイメージの作成
クイックスタートで使用するElasticSearchのDockerイメージはデフォルトのイメージであるため日本語解析用のプラグインkuromojiがインストールされていませんので、自分でイメージを作成します。
```dockerfile:Dockerfile
FROM docker.elastic.co/elasticsearch/elasticsearch:8.10.2
RUN bin/elasticsearch-plugin install analysis-kuromoji
```
```bash
 docker build -t elasticsearch-kuromoji:8.10.2 .
```

2. docker-compose.ymlファイルの修正
まずはデフォルト状態のdocker-compose.ymlをダウンロードします。
```bash
wget https://github.com/open-metadata/OpenMetadata/releases/download/1.3.0-release/docker-compose.yml
```
ダウンロードしたdocker-compose.ymlを3か所修正します。
    1. L42 docker.elastic.co/elasticsearch/elasticsearch:8.10.2 → elasticsearch-kuromoji:8.10.2
    ![](/images/create-openmetadata-elasticsearch-jp-index/compare_1.png)
    2. L167 -EN → -JP
    ![](/images/create-openmetadata-elasticsearch-jp-index/compare_2.png)
    2. L352 -EN → -JP
    ![](/images/create-openmetadata-elasticsearch-jp-index/compare_3.png)

3. docker-composeの実行
あとはクイックスタートに従い、修正したdocker-compose.ymlファイルで動かして完了です。
```bash
docker compose -f docker-compose.yml up --detach
```

すでに英語のインデックスがある場合うまく入れ替えができないので手動でインデックスを削除するのがおすすめです。参考までに全インデックスを削除するスクリプトを記載します
```python
import urllib3
import json
BASE_URL = 'http://localhost:9200/'

http = urllib3.PoolManager()

response = http.request('GET', BASE_URL + '_cat/indices?format=json')
es_index_list = json.loads(response.data.decode('utf-8'))
for es_index in es_index_list:
    http.request('DELETE', BASE_URL + es_index['index'])
```

# 感想
kuromojiで日本語を解析しているだけあって、あいまいな検索の制度が圧倒的に良いです。
例えば、tel_numberカラムに「電話番号」と備考が記述されていた場合、デフォルトのインデックスは「電話番号」で検索しないと引っかからないですが、日本語インデックスは「電話」でもしっかりヒットします。
また、前述した日本語UIもエンドユーザには受けがよく、アクティブに開発が継続していることから、フリーのデータカタログツールであればOpenMetadataが一番いいと感じました。
