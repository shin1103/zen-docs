---
title: "Airflowでサマータイムを考慮した話"
emoji: "🐷"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['airflow', 'python']
published: true
---

# 概要
Airflowでサマータイムを考慮する必要があり、Airflowのタイムゾーンの取り扱いについての調査を実施。調査結果は、以下の通り。

個別のDAGでサマータイムを意識する場合は、`Time zone aware DAGs`を使い、タイムゾーンの設定に`TZ database name`（例えばEurope/London、 America/New_York）を指定する。（標準時（例えばGMT、EST）ではだめ）

# 調査
## 方向性
Airflowすでにタイムゾーンが考慮されており、2つの方法で対応が可能。
https://airflow.apache.org/docs/apache-airflow/stable/authoring-and-scheduling/timezone.html  

1. airflow.cfgで設定
2. `Time zone aware DAGs`で設定

Airflowはすでに稼働中かつ一部のジョブのみサマータイムを加味したタイムゾーンを考慮する必要があったので後者の`Time zone aware DAGs`について調査を続行。

## Time zone aware DAGs
`Time zone aware DAGs`はDAG単位でタイムゾーンを設定できるもので、設定は極めて簡単。`start_date`にタイムゾーンを指定した`pendulum.datetime`オブジェクトを設定するだけでよい。
```python
import pendulum
dag = DAG("my_tz_dag", start_date=pendulum.datetime(2016, 1, 1, tz="Europe/Amsterdam"))
```

## 設定値
今回はロンドンでサマータイムを意識したかったので、ロンドンのサマータイムを考慮した場合どのような値にすればいいかを検討した。

[タイムゾーンの一覧](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones)を確認するとロンドンは以下の通りになっており、何を指定すればいいのかわからなかったので、それぞれを設定して実験をしてみた。（UTCも念のため）

| key                                            | value         |
| ---------------------------------------------- | ------------- |
| TZ database name（タイムゾーンデータベース名） | Europe/London |
| Standard Time（標準時刻）                      | GMT           |
| Daylight Saving Time（サマータイム）           | BST           |

```python
import pendulum

tzname_date: pendulum.datetime = pendulum.datetime(2023, 3, 25, tz="Europe/London")
gmt_date: pendulum.datetime = pendulum.datetime(2023, 3, 25, tz="GMT")
utc_date: pendulum.datetime = pendulum.datetime(2023, 3, 25, tz="UTC")
# bst_date: pendulum.datetime = pendulum.datetime(2023,3,25, tz="BST") 指定できずエラー

for i in range(4):
    print('-----------------')
    print(tzname_date.add(days=i))
    print(gmt_date.add(days=i))
    print(utc_date.add(days=i))
    # print(bst_date.add(days=i))
```
出力結果
```
-----------------
2023-03-25T00:00:00+00:00
2023-03-25T00:00:00+00:00
2023-03-25T00:00:00+00:00
-----------------
2023-03-26T00:00:00+00:00
2023-03-26T00:00:00+00:00
2023-03-26T00:00:00+00:00
-----------------
2023-03-27T00:00:00+01:00
2023-03-27T00:00:00+00:00
2023-03-27T00:00:00+00:00
-----------------
2023-03-28T00:00:00+01:00
2023-03-28T00:00:00+00:00
2023-03-28T00:00:00+00:00
```

以上より、サマータイムを意識するためには、標準時を指定せず`TZ database name`を指定する必要がある。