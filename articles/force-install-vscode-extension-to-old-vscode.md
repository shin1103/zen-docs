---
title: "古いバージョンのVSCodeに強引に最新のVSCode Extensionをインストールする方法"
emoji: "🐥"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["vscode"]
published: true
---
:::message alert
本記事は自己責任のもと実施ください
:::

# 概要
開発者間でVSCodeのバージョンを統一するため古いバージョンのVSCodeを使用中。その場合、最新のVSCode Extensionをインストールすることができず困っていたが、強引にインストールする方法がわかったので、よろしければ`自己責任のもと`試してみてください。

# 方法
1. VSCode Extensionsをローカルフォルダにダウンロード
Visual Studio Marketplaceでダウンロードができるのでダウンロードを行う。
![](/images/force-install-vscode-extension-to-old-vscode/vscode_marketplace.png)

2. ダウンロードしたファイルの拡張子をzipに変更
VSCode Extensionの拡張子はvsixだが、実態はzipファイルなので拡張子をzipに変更

3. zipファイルを解凍

4. extension/package.jsonを開く
解凍したzipファイルにはいり、extension/package.jsonを開く

5. vscodeのバージョンをお使いのVSCodeのバージョンに設定して保存
package.jsonにengines項目でVSCodeのバージョンが指定されているので、ここを現在使用中のVSCodeのバージョンに変更して保存
```json
  "engines": {
    "vscode": "^1.63.0"
  }
```

6. 解凍したファイルを再度zip形式で圧縮
解凍したファイルを再度zip形式で圧縮。この際、zipファイルのトップディレクトリが元の通りになるよう注意して作成すること
![](/images/force-install-vscode-extension-to-old-vscode/vsix_top_directory.png)

7. 拡張子をvsixに変更

8. VSCodeでvsixを読み込ませる
ローカルのVSCode Extensionsを読み込ませる場合はサイドバーで拡張機能を選択した後、メニューを表示し「VSIXからのインストール」を選択する
![](/images/force-install-vscode-extension-to-old-vscode/extension_menu1.png)
![](/images/force-install-vscode-extension-to-old-vscode/extension_menu2.png)


以上。
はじめにも記載しましたが、`自己責任のもと`お願いします。
:::message alert
本記事は自己責任のもと実施ください
:::
