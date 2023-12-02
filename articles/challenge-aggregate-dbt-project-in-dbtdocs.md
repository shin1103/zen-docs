---
title: "複数のdbtプロジェクトをdbt docsでまとめようとした話"
emoji: "📑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["dbt"]
published: false
---
# 概要
私が所属するチームでは導入の敷居を下げるために１データソースに１dbtプロジェクトという構成になっています。そのため、[dbt docs](https://docs.getdbt.com/docs/collaborate/documentation)でドキュメントを作成した場合、データソースごとに`dbt docs`でWebサイトを立ち上げる必要があり、とても業務に使えるものではありませんでした。
dbt Cloudであれば、[coalesce　2023](https://coalesce.getdbt.com/)で発表された[dbt Mesh](https://www.getdbt.com/product/dbt-mesh)を使って複数のプロジェクトを統合できるようですが、会社の都合上クラウドサービスを使うのは難易度が高いため、`dbt docs`でどうにかできないか検討し、以下の2点から実運用には耐えられないとの結論でした。
* dbt packageの依存関係解決
* マスター系データの重複作成

![](/images/challenge-aggregate-dbt-project-in-dbtdocs/architecture.drawio.png)
図１：アーキテクチャ構成

# 検証内容
## コンセプト
概要の内容ができないかと思ったのはdbtのパッケージで自分のプロジェクトを参照できるという点でした。
https://docs.getdbt.com/docs/build/packages#how-do-i-add-a-package-to-my-project

これらシステムごとに作成したdbtプロジェクトをまとめて一つの`dbt docs`を作成することで、dbtで作成したデータを一元管理できるのではないかと考えました。コンセプトは図２の通りです。

![](/images/challenge-aggregate-dbt-project-in-dbtdocs/concept.drawio.png)
図２：検証コンセプト

## 検証内容
2つのプロジェクトを作成して問題なくdbt docsが作成され、作成されたドキュメントが利用可能であるかを確認する。