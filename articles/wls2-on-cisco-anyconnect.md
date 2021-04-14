---
title: "wsl2（Ubuntu20.04）とCisco AnyConnectの組み合わせでインターネット接続をする"
emoji: "✨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["wls2", "cisco", "anyconnect", ubuntu]
published: true
---

# 概要
会社のWindows10のバージョンが1909に上がったため、wsl2を入れられるようになった。
その際にインターネット接続に苦労したので、その設定方法を記載する。
（写経しているので、コマンドが間違っていたらごめんなさい）

問題となったのは２点で、
* resolv.confが社内のDNSサーバーが設定されていない。設定しても、起動の都度書き換えられてしまい、都度設定が必要。
* AnyConnectがルーティング情報を書き換えているので、社内のDNSサーバーに名前解決にいけない。また、プロキシサーバーにアクセスできない。

# 設定方法
## resolv.confの設定
ubuntuの場合DNSの設定はresolv.confに記述する。resolv.confの書き換えを自動or手動かの設定は`dpkg-reconfigure resolvconf`で設定できるようだが、wls2の場合はwsl.confに記載する必要がある。

```bash
sudo vi /etc/wsl.conf
```
次のとおり記載。
```ini
[network]
generateResolvConf = false
```

管理者権限でpowershellを立ち上げ、次のコマンドを実行してwslの再起動。
```
wsl --shutdown
```
サイドwslのシェルを起動すると自動的にwslが再起動するので、次のファイルを編集。ただし、事前にシンボリックリンクになっているか確認し、シンボリックリンクの場合は上記設定に失敗しているので設定内容を見直すこと。
```bash
ls -l /etc/resolv.conf
sudo vi /etc/resolv.conf
```

ここは個別の設定を記載する。
```ini
nameserver X.X.X.X
search mydomain.local
```

## ルーティングの設定
[Githubのissue](https://github.com/microsoft/WSL/issues/4277#issuecomment-779141250)の内容でOK.ただし、**AnyConnectに接続する前にwls2が起動していないと後述のスクリプトを実行してもうまくいかない。**

1. `%HOMEDRIVE%\%HOMEPATH%\scripts\UpdateAnyConnectInterfaceMetric.ps1`に次の内容を記載
```bash
Get-NetAdapter | Where-Object {$_.InterfaceDescription -Match "Cisco AnyConnect"} | Set-NetIPInterface -InterfaceMetric 6000
```

2. タスクスケジューラを設定
具体的な設定方法はサイトの内容を転記します。
>Open 'Task Scheduler'
>Click "Create Task" on Right Sidebar
>
>Name: Update Anyconnect Adapter Interface Metric for WSL2
>
>Set Security Options
>
>Check box: 'Run with highest priveleges'
>Select 'Triggers' Tab
>
>Click 'New' at bottom of Window
>
>Open 'Begin the task' drop-down
>
>Select 'On an Event'
>
>Configure Event:
>
>Log: 'Microsoft-Windows-NetworkProfile/Operational'
>Source: 'NetworkProfile'
>Event ID: '10000'
>Click 'OK'
>
>Select 'Actions' Tab
>
>Click 'New'
>
>Configure Action:
>
>Action: 'Start a Program'
>Program/script: 'Powershell.exe'
>Add arguments: '-ExecutionPolicy Bypass -File %HOMEDRIVE%\%HOMEPATH%\scripts\UpdateAnyConnectInterfaceMetric.>ps1'
>Click 'OK'
>
>Select 'Conditions' Tab
>
>Uncheck box:
>
>Power -> Start the task only if the computer is on AC Power
>Click 'OK'

以上

# 参考
* https://docs.microsoft.com/ja-jp/windows/wsl/wsl-config
* https://server.etutsplus.com/how-to-configure-systemd-resolved-and-resolvconf-on-ubuntu/
* https://aquasoftware.net/blog/?p=1472 
* https://github.com/microsoft/WSL/issues/4277#issuecomment-779141250