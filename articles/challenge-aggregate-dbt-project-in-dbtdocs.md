---
title: "複数のdbtプロジェクトをdbt docsでまとめようとした話"
emoji: "📑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["dbt"]
published: false
---
# 概要
私が所属するチームでは導入の敷居を下げるために１データソースに１dbtプロジェクトという構成になっています。そのため、[dbt docs](https://docs.getdbt.com/docs/collaborate/documentation)でドキュメントを作成した場合、データソースごとに`dbt docs`でWebサイトを立ち上げる必要があり、とても業務に使えるものではありませんでした。
dbt Cloudであれば、[coalesce　2023](https://coalesce.getdbt.com/)で発表された[dbt Mesh](https://www.getdbt.com/product/dbt-mesh)を使って複数のプロジェクトを統合できるようですが、会社の都合上クラウドサービスを使うのは難易度が高いため、`dbt docs`でどうにかできないか検討し、実運用には耐えられないとの結論でした。

# 検証内容
概要の内容ができないかと思ったのはdbtのパッケージで自分のプロジェクトを参照できるという点でした。[ドキュメント](https://docs.getdbt.com/docs/build/packages#how-do-i-add-a-package-to-my-project)

