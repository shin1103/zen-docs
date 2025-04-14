---
title: "embulk-input-icebergを作りました"
emoji: "🐙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["java", "embulk", "iceberg"]
published: true
---
# 初めに
データエンジニアをやっていると少なくないケースでバックエンドがJavaであることがあり、Javaを勉強したいと思っていました。最近[Apache Iceberg](https://iceberg.apache.org/)に興味を持ち勉強している中で業務で使用しているEmbulkのプラグインにIcebergがないと思い、Javaの勉強がてら作ってみようと決心し作ってみました。  
本当はoutputの方が需要はありそうだったのですが、実装の考慮点が少なそうなinputを選択してIcebergのプラグインを作成しました。
ソースコードは[Github](https://github.com/shin1103/embulk-input-iceberg)に公開するとともに、Mavenリポジトリに登録しています。

# 完成までの道のり

## Embulkのプロジェクト作成
### プラグイン形式の決定
EmbulkはJRubyにプラグインをリリースしていたようですが、[dmikurube](https://zenn.dev/dmikurube)さんの[2021年の記事](https://zenn.dev/dmikurube/articles/get-ready-for-embulk-v0-11-and-v1-0)の記事を見るとMavenリポジトリにプラグインをリリースする形式を推奨しているようでしたので、こちらのやり方でリリースすることにしました。

### 環境構築
Maven形式のプラグイン作成の環境構築は[この記事にインストール方法](https://www.embulk.org/articles/2024/06/13/installing-maven-style-embulk-plugins.html)が記載されていていました。

開発したいプラグインはIceberg関係のライブラリをインストールする必要があったので、[The Gradle org.embulk.runset plugin](https://github.com/embulk/gradle-embulk-runset) を使用して環境構築を試みたのですが、[リンクのissue](https://github.com/ben-manes/caffeine/issues/716)を実力不足で解消することができませんでした。

結局、[Emubulk Home](https://zenn.dev/dmikurube/articles/embulk-v0-11-is-coming-soon-ja)に関する記事を参照してm2_homeをMavenのローカルリポジトリに設定し、開発中のEmbulkプラグインを動かすのに必要なライブラリをMavenのローカルリポジトリに落して開発を行いました。

### Javaのバージョン
[Embulkの公式サイト](https://www.embulk.org/)によると公式サポートはJava8で11,17,21でもなんとなく動くとありました。そして、[IcebergはJava11以上でビルド](https://github.com/apache/iceberg/tree/apache-iceberg-1.8.1)されているので、Java11を利用することにしました。

## コーディング
コーディングの際に苦労した点は以下の通りです。

### インターフェースの理解
[Development Guild](https://docs.google.com/document/d/1oKpvgstKlgmgUUja8hYqTqWxtwsgIbONoUaEj8lO0FE/edit?tab=t.0)が出ていて大枠は理解できたのですが、具体的な実装になるとイメージができず既存のライブラリを参考にして作成しました。特に参照したのが以下２つでした。
- [embulk-input-jdbc](https://github.com/embulk/embulk-input-jdbc)
- [embulk-input-athena](https://github.com/shinji19/embulk-input-athena)

### Iceberg Java APIの理解
インターネット上には公式サイトの他ほとんど情報がなく、以下の３つのサイトくらいしか具体的な実装に触れられているものはありませんでしたが、それら情報でほぼ形にできました。一部足りないものについてはソースコードのコメントから理解するような形で何とか形にしました。
- [An Introduction to the Iceberg Java API Part 1](https://www.tabular.io/blog/java-api-part-1/)（パート２、３もあります）
- [TrinoとIcebergでログ基盤の構築](https://knowledge.sakura.ad.jp/36085/)
- [氷山を穿つ - Apache Icebergに大量データを投入するTopic -](https://caddi.tech/2025/03/31/114754)

### Classloaderの理解
サンプルのJavaプロジェクトでは動いていたのに、Embulkのプラグインにしたらうまく動かないことがありました。突き詰めていくとClassloaderがうまく設定されていないようでした。いろいろなEmbulkのプラグインのソースコードを見ると[embulk-output-parquet]( https://github.com/choplin/embulk-output-parquet)で私の抱える問題の対処になるのかと思い試したら見事解決しました。

## デプロイ
せっかくなのでMaven Centralにデプロイを試みました。

### Maven Central or OSSRH
最近まではOSSRHでリリースするのが一般的だったようなのですが、[このニュース](https://central.sonatype.org/news/20250326_ossrh_sunset/)によると2025/06/30にサポートが終了するようなので、Maven Centralへのリリースをすることにしました。
しかし、Gradleについては[公式のパブリッシュプラグインがない](https://central.sonatype.org/publish/publish-portal-gradle/)ため、同サイトにあるOSSを活用することが必要でした。
様々なパブリッシュプラグインを試しましたが、Javaのバージョンでダメになったり、署名がうまくいかないなど結局デプロイまでうまくいくものはなく、(このプラグイン)[https://github.com/yananhub/flying-gradle-plugin]は、ハッシュの作成と署名までうまくいったので、これを手作業でZIPに固めてMaven Centralにアップロードする方法で行いました。

### gradle-embulk-runsetの挙動
リリース後[The Gradle org.embulk.runset plugin](https://github.com/embulk/gradle-embulk-runset)で作成したembulk-input-icebergを読み込ませたのですが、[リンクのissue](https://github.com/ben-manes/caffeine/issues/716)は解消しなかったので、リンクに記載されていた該当の事象が発生するcaffeineというモジュールのバージョンをあげて再度リリース(v0.0.2)を行いました。
実際に動かしてみてcaffeineでエラーが出る場合はIcebergのモジュールが参照しているバージョンを参照しているv0.0.1を使ったほうがいいかもしれません。

# 終わりに
今回embulkのプラグインを作ることで、JavaだったりGradleだったりと少しだけ理解が深まった気がします。
今回対応したカタログはRESTのみかつストレージはMiniOのみ（というか試していない）なので、カタログタイプを増やしたり、outputについても作ってみたいと思いました。