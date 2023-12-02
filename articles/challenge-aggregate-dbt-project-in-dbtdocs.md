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
2つのプロジェクトを作成して問題なくdbt docsが作成され、作成されたドキュメントが利用可能であるかを確認しました。
(実際には実プロジェクトのコードで確認しており、その検証結果の事象を再現するようにサンプルプロジェクトを作成しました。)

dbtではgitリポジトリを次のように記述することで、自身のプロジェクトを参照できます。
```yaml
packages:
  - git: "https://github.com/shin1103/dbtdocs-sample-system-a.git"
    subdirectory: "systemA"　# gitリポジトリ上のディレクトリを指定可能
    revision: "main" # gitのブランチ
  - git: "https://github.com/shin1103/dbtdocs-sample-system-a.git"
    subdirectory: "systemB"
    revision: "main"
```

上記設定をしてドキュメントを生成したのですが、概要にもある通り２つの点でうまくいきませんでした。
### dbt packageの依存関係解決
繰り返しになりますが、私のチームではシステムごとにdbtのプロジェクトを作成していました。`dbt_utils`に代表されるパッケージについては都度最新のパッケージを使うルールにしていました。そのためsystemAではバージョン1.0、systemBではバージョン1.1のようになっています。そのため、依存関係でエラーが起きます。
```log
08:49:45  Encountered an error:
Version error for package dbt-labs/dbt_utils: Could not find a satisfactory version from options: ['=1.0.0', '=1.1.0']
```
解消する案としてdbt_utilsのバージョン指定を"1.0.0"から">=1.0.0"にする方法はあるが、依存関係のあるパッケージの破壊的変更発生時にどこまで対応できるのか不明なので、将来的には予期せぬ不具合を起こしかねないと思いました。

### マスター系データの重複作成
個別システムのデータソースは基本的に重複することはないのですが、マスター系のデータについてはシステム横断で参照するため、複数のシステムから参照されています。
その場合、dbtでは同一のものをしてマージしてくれることはなく、別々のデータソースと認識するので`dbt docs`でドキュメントを作成した際に同じデータソースでも別々のものとして認識されてしまいます。

![](/images/challenge-aggregate-dbt-project-in-dbtdocs/duplicate_master.png)
図３：重複のスクリーンショット  

# 検証結果
`dbt package`のgitリポジトリ参照機能を用いて`dbt docs`を統合することは厳しそうです。`dbt Mesh`を導入するのがよいです