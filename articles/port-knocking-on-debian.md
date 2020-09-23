---
title: "Debianでポートノッキング"
emoji: "🔑"
type: "tech"
topics: ["PortKnocking", "Security", "SSH", "ufw"]
published: true
---

Zenn での初記事です。

先日、[何故SSHのポートを22以外にするのは悪いことなのか](https://adayinthelifeof.nl/2012/03/12/why-putting-ssh-on-another-port-than-22-is-bad-idea/) という記事を見かけました。
この記事自体は2012年のものでそれほど新しいことは書かれていないのですが、ポートノッキングを選択肢として挙げられており、自鯖の最近の動向から併用するのも良いなと思って、自鯖のいくつかをポートノッキングを導入したので、その備忘録です。

タイトルにあるように対象はDebianで9と10での記録ですが、Ubuntuでも同様に設定できるはずです。
CentOS? 知らない子ですね。

さて、インストール自体は簡単で [knockd](https://zeroflux.org/projects/knock) というツールを使います。

```
$ sudo apt install knockd
```

インストールすると下記の場所にそれぞれファイルが配置されます。(ドキュメントは割愛)

```
/etc/default/knockd
/etc/init.d/knockd
/etc/knockd.conf
/lib/systemd/system/knockd.service
/usr/bin/knock
/usr/sbin/knockd
```

このうち、`/etc/knockd.conf` は下記のようになっています。

```
[options]
	UseSyslog

[openSSH]
	sequence    = 7000,8000,9000
	seq_timeout = 5
	command     = /sbin/iptables -A INPUT -s %IP% -p tcp --dport 22 -j ACCEPT
	tcpflags    = syn

[closeSSH]
	sequence    = 9000,8000,7000
	seq_timeout = 5
	command     = /sbin/iptables -D INPUT -s %IP% -p tcp --dport 22 -j ACCEPT
	tcpflags    = syn

```

これは 7000, 8000, 9000 のポートを(5秒以内に)順番に叩くことで叩いてきたIPアドレスに対してSSHのポートを開くという openSSH というサービスと、先ほどとは逆に 9000, 8000, 7000 の順番に叩くことで開いたSSHのポートを閉じるというcloseSSHというサービスが登録されています。

自鯖では iptables そのものを使っておらず、代わりに [ufw](https://help.ubuntu.com/community/UFW) というiptablesの管理ツールを使っているので、それに書き換えます。またこのままでは閉じるコマンドを忘れると開いたままになってしまうので、これも改めます。
その結果、次のようになりました。 

```
[options]
	UseSyslog

[openSSH]
	sequence    = 7000,8000,9000
	seq_timeout = 5
	tcpflags    = syn
	start_command = /usr/sbin/ufw allow from %IP% to any port 22 proto tcp
	cmd_timeout   = 30
	stop_command  = /usr/sbin/ufw delete allow from %IP% to any port 22 proto tcp
```

意味としては、7000,8000,9000のTCPポートを順番に5秒以内に叩くことで、叩いてきたIPアドレスに対してSSHポートを30秒のみ開き、30秒が経過すると追加したルールを削除するというものです。
実際に導入する際には `sequence` のポートは適宜変更してください。それぞれのポート番号はある程度開いたものにする方が良いかと思います。

少し前後しましたが、サービスとしての `knockd` を起動するため `/etc/default/knockd` の `START_KNOCKD` を `1` に書き換えます。

``` diff
--- /etc/default/knockd	2016-10-08 14:05:00.000000000 +0000
+++ /etc/default/knockd	2020-09-23 05:31:02.706257093 +0000
@@ -2,7 +2,7 @@
 # 1 = start
 # anything else = don't start
 # PLEASE EDIT /etc/knockd.conf BEFORE ENABLING
-START_KNOCKD=0
+START_KNOCKD=1

 # command line options
 #KNOCKD_OPTS="-i eth1"
```

続いてサービスを起動します。

```
$ sudo service knockd start
```

サービス起動を確認します。

```
$ sudo service knockd status
● knockd.service - Port-Knock Daemon
   Loaded: loaded (/lib/systemd/system/knockd.service; static; vendor preset: enabled)
   Active: active (running) since Wed 2020-09-23 05:42:42 UTC; 5s ago
     Docs: man:knockd(1)
 Main PID: 26335 (knockd)
    Tasks: 1 (limit: 4915)
   CGroup: /system.slice/knockd.service
           └─26335 /usr/sbin/knockd

Sep 23 05:42:42 vpn.deny.zone systemd[1]: Started Port-Knock Daemon.
Sep 23 05:42:42 vpn.deny.zone knockd[26335]: starting up, listening on eth0
```

動作確認のためログをターミナルに表示しておきます。

```
$ sudo journalctl -fu knockd.service
Sep 23 05:42:42 vpn.deny.zone systemd[1]: Started Port-Knock Daemon.
Sep 23 05:42:42 vpn.deny.zone knockd[26335]: starting up, listening on eth0
```

これから手元から動作確認をするのですが、Macでは brew にて同じ knockd がインストール可能です。
クライアントのみなせいか名前は `knock` ですが、インストールします。

```
$ brew install knock
```

Windowsな方は[こちら](https://zeroflux.org/projects/knock)からどうぞ。

動作確認は設定に従い、下記のように行います。ディレイのオプションを利用しないと動作しませんでしたので、適当に10ほど入れておきます。

```
$ knock -d 10 vpn.deny.zone 7000 8000 9000
```

先ほど開いたままのログを確認すると下記のようになりました。

```
Sep 23 05:48:01 vpn.deny.zone knockd[26335]: 11X.XX.XX.XX: openSSH: Stage 1
Sep 23 05:48:01 vpn.deny.zone knockd[26335]: 11X.XX.XX.XX: openSSH: Stage 2
Sep 23 05:48:01 vpn.deny.zone knockd[26335]: 11X.XX.XX.XX: openSSH: Stage 3
Sep 23 05:48:01 vpn.deny.zone knockd[26335]: 11X.XX.XX.XX: openSSH: OPEN SESAME
Sep 23 05:48:01 vpn.deny.zone knockd[26356]: openSSH: running command: /usr/sbin/ufw allow from 11X.XX.XX.XX to any port 22 proto tcp
Sep 23 05:48:01 vpn.deny.zone knockd[26335]: ERROR: '/etc/ufw/user.rules' is not writable
Sep 23 05:48:01 vpn.deny.zone knockd[26335]: openSSH: command returned non-zero status code (1)
Sep 23 05:48:01 vpn.deny.zone knockd[26356]: openSSH: command returned non-zero status code (1)
```

Stage 1, Stage 2, Stage 3と順調に確認できているのですが、`/etc/ufw/user.rules is not writable` というエラーが出ています。
これは提供されているdebianパッケージのバグで、詳細は[#1823051](https://bugs.launchpad.net/ubuntu/+source/knockd/+bug/1823051) にあるのですが、 `/lib/systemd/system/knockd.service` で `ProtectSystem=full` と指定されているため `/etc` 以下に対して書き込み権限がないことが原因です。 iptables の場合には影響を受けませんが、 `ufw` ではコマンドを実行する度に `/etc/ufw` 以下のファイルを書き換えるので、エラーとなるようです。
修正は上記のBTSにも記載されていますが、 `ProtectSystem=true` に修正します。ちなみに Ubuntu ではすでに修正されているので、この修正は必要ありません。

```diff
--- /lib/systemd/system/knockd.service	2016-10-08 14:05:00.000000000 +0000
+++ /lib/systemd/system/knockd.service	2020-09-23 05:33:04.706257093 +0000
@@ -9,5 +9,5 @@
 ExecReload=/bin/kill -HUP $MAINPID
 KillMode=mixed
 SuccessExitStatus=0 2 15
-ProtectSystem=full
+ProtectSystem=true
 CapabilityBoundingSet=CAP_NET_RAW CAP_NET_ADMIN
```

systemdのサービスを書き換えたのでリロードして、knockdを再起動します。

```
$ sudo systemctl daemon-reload && sudo service knockd restart
```

再度、サービスのログを表示して、動作確認に入ります。

```
$ sudo journalctl -fu knockd.service
Sep 23 06:28:12 vpn.deny.zone knockd[26335]: waiting for child processes...
Sep 23 06:28:12 vpn.deny.zone knockd[26335]: shutting down
Sep 23 06:28:12 vpn.deny.zone systemd[1]: Stopping Port-Knock Daemon...
Sep 23 06:28:12 vpn.deny.zone systemd[1]: Stopped Port-Knock Daemon.
Sep 23 06:28:12 vpn.deny.zone systemd[1]: Started Port-Knock Daemon.
Sep 23 06:28:12 vpn.deny.zone knockd[26682]: starting up, listening on eth0
```

Mac に戻り再度実行します。

```
$ knock -d 10 vpn.deny.zone 7000 8000 9000
```

表示していたログを確認します。

```
Sep 23 06:30:54 vpn.deny.zone knockd[26682]: 11X.XX.XX.XX: openSSH: Stage 1
Sep 23 06:30:54 vpn.deny.zone knockd[26682]: 11X.XX.XX.XX: openSSH: Stage 2
Sep 23 06:30:54 vpn.deny.zone knockd[26682]: 11X.XX.XX.XX: openSSH: Stage 3
Sep 23 06:30:54 vpn.deny.zone knockd[26682]: 11X.XX.XX.XX: openSSH: OPEN SESAME
Sep 23 06:30:54 vpn.deny.zone knockd[26684]: openSSH: running command: /usr/sbin/ufw allow from 11X.XX.XX.XX to any port 22 proto tcp
Sep 23 06:30:54 vpn.deny.zone knockd[26682]: Rule added
```

うまくいったようです。ログ表示を終了して ufw で確認します。

```
$ sudo ufw status
Status: active

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW       11X.XX.XX.XX
```

追加されていますね。30秒経過を待ってから、ログを確認します。

```
$ sudo service knockd status
<snip>
Sep 23 06:33:22 vpn.deny.zone knockd[26746]: 11X.XX.XX.XX: openSSH: command timeout
Sep 23 06:33:22 vpn.deny.zone knockd[26746]: openSSH: running command: /usr/sbin/ufw delete allow from 11X.XX.XX.XX to any port 22 proto tcp
Sep 23 06:33:22 vpn.deny.zone knockd[26682]: Rule deleted
```

ちゃんと30秒経過でルールの削除も実施できているようです。


参考
https://zeroflux.org/projects/knock
https://help.ubuntu.com/community/PortKnocking
