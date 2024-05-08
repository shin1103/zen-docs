---
title: "OpenMetadataとMicrosoft Active Directoryを連携する"
emoji: "👋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["openmetadata", "activedirectory"]
published: true
---
# はじめに
社内でデータカタログ導入に向けて調査を進めています。OpenMetadataについては[LDAP認証が行える](https://docs.open-metadata.org/v1.3.x/deployment/security/ldap)ことから、MicrosoftのActive Directoryと連携を行いましたので、その方法を記載します。

# 前提条件
1. Docker上でOpenMetadataを構築
上記LDAP認証はDocker上に構築したケースとBareMetalで構築したケースがありますが、Docker上に構築したケースです。

2. トラストストアは使用しない
LDAP認証でJVMのデフォルトストアや証明書ストアは自身で準備したトラストストア（カスタムトラストストア）を使用できますが、今回はトラストストアは使用しません。（トラストストアを使う方法は上記公式サイトに記載されています）

3. ホスト制限は行わない
公式サイトにはホスト名を制限する方法が記載されていますが、その設定は行いません。

# 設定方法
公式サイトに手順が記載されているので、その手順に従い実施します。様々な設定方法がごっちゃになっていますが、上記前提だとほとんど設定はいりません。

ENVファイルを使ってLDAP認証に必要な環境変数を設定します。トラストストアを使用せず、ホスト制限を使わない場合は設定例が公式サイトに記載されているので、その値を次の通り設定しました。

```:openmetadata_ldap.env
AUTHENTICATION_PROVIDER=ldap
AUTHENTICATION_LDAP_HOST=192.168.0.11
AUTHENTICATION_LDAP_PORT=389
#AUTHENTICATION_LDAP_PORT=636 #ssl
AUTHENTICATION_LOOKUP_ADMIN_DN="CN=Administrator, CN=Users, DC=mydomain, DC=local"
AUTHENTICATION_LOOKUP_ADMIN_PWD="xxxxxx"
AUTHENTICATION_USER_LOOKUP_BASEDN="OU=MY_OU, DC=mydomain, DC=local"
#OpenMetadata default is email. but Active Directory uses mail
AUTHENTICATION_USER_MAIL_ATTR=userPrincipal Name
AUTHENTICATION_LDAP_POOL_SIZE=3
AUTHENTICATION_LDAP_SSL_ENABLED=false
AUTHENTICATION_LDAP_TRUSTSTORE_TYPE=TrustAll
AUTHENTICATION_LDAP_EXAMINE_VALIDITY_DATES=true
```

上記ENVファイルでActive Directory固有の注意点は以下の通り
1. ポート番号
Active Directoryと接続するポートは`AUTHENTICATION_LDAP_PORT`で設定可能。SSL接続の場合は636、SSL接続を使わない場合は389を使用する
2. メールアドレス
`AUTHENTICATION_USER_MAIL_ATTR`はEMailを使用しますが、Active Directoryの場合は`userPrincipal Name`に記載されている
3. 接続ユーザ
Active Directoryに接続するユーザは`AUTHENTICATION_LOOKUP_ADMIN_DN`で設定できますが、検証目的での実施のため、ドメインのAdministratorを指定していますが、本番運用では適切な権限設定が必要と思います。（具体的な権限については調査していません）
4. ユーザ探索ディレクトリ
`AUTHENTICATION_USER_LOOKUP_BASEDN`で認証対象のユーザが存在するルートのディレクトリを設定可能。既存のActive Directoryの構成であるDNいかにすべてのユーザが存在しない場合、すべてのユーザを認証できないように思います。（検証未実施）

上記のENVファイルを設定したうえで、Dockerイメージを起動すればDLAP認証が行えます。
```
docker compose --env-file ~/openmetadata_ldap.env up -d
```

# 認証関連
認証関連で気になった点を記載します。
## ログイン時のドメイン名省略
OpenMetadataのログイン画面はユーザ名かE-Mailアドレスを認証時に使用しますが、ユーザ登録時にはユーザ名を入れる場所はありません。ユーザ名はE-Mailアドレスのアットマークの前の部分をユーザ名といっているようで、OpenMetadataの`AUTHORIZER_PRINCIPAL_DOMAIN`と一致するドメインの場合だけ、ユーザ名で認証が可能でした。なので、Active Directoryと認証する際にはActive Directoryのドメインと`AUTHORIZER_PRINCIPAL_DOMAIN`をそろえておくと、利用者の利便性が向上しそうです。
## 管理者ユーザの登録
OpenMetadataではデフォルトの認証はベーシック認証になっていて、`AUTHORIZER_ADMIN_PRINCIPALS`に従った管理者権限ユーザ（デフォルト`admin`）が作成される。この状態でLDAP認証を有効にするとこのユーザではログインできなくなります。
くわえてLDAP認証の場合、OpenMetadata上に事前にユーザ登録しておかなくても、初回LDAP認証時にユーザが作成されますが、その権限はデフォルト権限でした。
なので、事前に準備せずにLDAP認証にすると管理者権限ユーザが存在しなくなります。それを避けるために、ベーシック認証時に管理者権限ユーザでLDAP認証するドメインのユーザを管理者権限として作成しておくことで、LDAP認証に変更しても管理者権限ユーザが存在するようになります。

# 終わりに
本記事は検証結果であり、実運用開始後にTipsがあれば再度共有したいと思います。