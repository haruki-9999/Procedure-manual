## 初めに
---
解説とか、エラー対応の時間があまり取れなかったので、横山のAWSアカウントでNFSの演習をやってみました。これを見て、何がうまくいってなかったのか、なんでこの設定をするのか、自己学習する際に役立ててください。特にDNSの部分は使う現場は多いと思います。


- 構成はEC2が全３台
    - DNS兼NFSサーバが一台
    - メールサーバが一台ずつ

- アジェンダ
  - DNS構築
  - メールサーバ構築 1台目
  - メールサーバ構築 2台目
  - NFSサーバ構築　(今回はDNSサーバと同じサーバに構築)

---
## DNS 構築
---
bindインストール/有効化
```
dnf install -y bind
systemctl start named
systemctl status named
```
bind設定ファイル編集＆結果確認
```
・バックアップの作成
    ls -l /etc | grep named.conf
    cp -a /etc/named.conf{,.bak}
    ls -l /etc | grep named.conf
・バックアップの編集
    vi /etc/named.conf
        自サーバ以外から通信を受け付けるように変更を加える
        変更前：listen-on port 53 { 127.0.0.1; };
               listen-on-v6 port 53 { ::1; };
        変更後：# listen-on port 53 { 127.0.0.1; };
            　　# listen-on-v6 port 53 { ::1; };
        全てのサーバからのクエリを許可する(非推奨)
        変更前：allow-query { localhost; };
        変更後：allow-query { any; };
・構文チェック
    named-checkconf
    何かでるならファイルを確認して修正する。
・bindの再起動
    systemctl restart named
    systemctl status named
・ポート確認
    netstat -ln | grep 53
    tcpで自身のプライベートIPがオープンしていること
    tcp 0 0 プライベートIP:53 0.0.0.0:* LISTEN
```
ゾーン情報の定義(bind設定ファイルの編集)
```
・ファイルの編集
    vi /etc/named.conf
    以下を追記

    zone "サブドメ" IN {
        type master;
        file "/var/named/サブドメ.zone";
    };

・構文チェック
    named-checkconf
    何かでるならファイルを確認して修正する。
```
ゾーンデータベースファイルの作成
```
書き方は詰まりポイントかも。この辺のレコードの役割とか、順番とかはしっかり覚えておきましょう。
※優先度の部分はこのあと、mail01に飛ばしたり、mail02に飛ばしたり、の動きを操りたいので10と15にしています。最終的には10と10にします。
・vi /var/named/サブドメ.zone
    以下の追記

    $TTL 3600
    @ IN SOA ns.サブドメ. test.gmail.com. (
    20220401 ; serial
    3600 ; refresh
    3600 ; retry
    3600 ; expire
    3600 ) ; minimum
            IN NS ns.サブドメ.
            IN MX 10 mail01.サブドメ.
            IN MX 15 mail02.サブドメ.
    ns      IN A DNSサーバのプライベートIP(自身のため)
    mail01  IN A メールサーバ１のパブリックIP
    mail02  IN A メールサーバ２のパブリックIP

・構文チェック
    named-checkzone サブドメ /var/named/サブドメ.zone
・再起動
    systemctl restart named
    systemctl status named
```

名前解決テスト
```
ローカルのテスト　
※構文チェックがうまくいっていれば結果は帰ってくるはず
・dig @localhost ns.サブドメ
・dig @localhost mail01.サブドメ
パブリックテスト
・mail01.サブドメ

whoisというサイトでも情報検索ができる
ローカルから確認しているから結果が帰ってきているだけでは？という疑問を持っているときは外部のサイトでも確認してみる。
https://www.cman.jp/network/support/ip.html
```
---
## メールサーバ構築 1台目
Postfixのインストール/有効化
```
dnf install -y postfix
systemctl start postfix
systemctl status postfix
```

バックアップの作成
```
    ls -l /etc/postfix/ | grep main.cf
    cp -a /etc/postfix/main.cf{,.bak}
    ls -l /etc/postfix/ | grep main.cf
```
バックアップの編集
```
    vi /etc/postfix/main.cf

    以下の項目を追記

    myhostname = mail01.サブドメ
    mydomain = サブドメ
    myorigin = $mydomain
    mynetworks = 172.31.0.0/16
    mail_spool_directory = /var/spool/mail/

    以下の項目を変更
    inet_interfaces = all
    mydestination = $mydomain , $myhostname
```
Postfix再起動
```
systemctl restart postfix
systemctl status postfix
```

一つ目のメールアドレス作成
```
各サーバのユーザの作成順は詰まりポイントかも。idコマンドでUIDを確認しておきましょう。
useradd yokoyama01 -g mail -M -K MAIL_DIR=/dev/null -s /sbin/nologin
passwd yokoyama01
> New password: yokoyama01
> Retype new password: yokoyama01
```

Dovecot インストール
```
dnf install -y dovecot
```
Dovecot 設定変更
```
ls -l /etc/dovecot/ | grep dovecot.conf
cp -a /etc/dovecot/dovecot.conf{,.bak}
ls -l /etc/dovecot/ | grep dovecot.conf

vi /etc/dovecot/dovecot.conf
    以下の項目を変更
    protocols = pop3
   
    以下の項目を追記
    mail_location = maildir:/var/spool/mail/%u/


```
SSL無効化
```
ls -l /etc/dovecot/conf.d/ | grep 10-ssl
cp -a /etc/dovecot/conf.d/10-ssl.conf{,.bak}
ls -l /etc/dovecot/conf.d/ | grep 10-ssl

vi /etc/dovecot/conf.d/10-ssl.conf
    変更前：ssl = required
    変更後：#ssl = required

ls -l /etc/dovecot/conf.d/ | grep 10-a
cp -a /etc/dovecot/conf.d/10-auth.conf{,.bak}
ls -l /etc/dovecot/conf.d/ | grep 10-a

vi /etc/dovecot/conf.d/10-auth.conf
    変更前：#disable_plaintext_auth = yes
    変更後：disable_plaintext_auth = no

```
Dovecot 起動
```
systemctl start dovecot
systemctl enable dovecot
systemctl is-enabled dovecot
```

telnetインストール
```
dnf install -y telnet
```

メールログの有効化
```
dnf install -y rsyslog
systemctl start rsyslog
systemctl status rsyslog
```
いったんメールが届くか確認

```
yokoyama01@サブドメ宛てにメールを送る

確認1
telnet mail01.サブドメ 110
user yokoyama01
pass yokoyama01
list

確認2
cd /var/spool/mail/
ls

yokoyama01ディレクトリがあるか確認
/newに新規メール
/curに今までのメール？が届いている(あんまり確認はしていないので多分)

htmlにフォーマットされているがとりあえずよしとする。


```
二つ目のメールアドレス作成
```
useradd yokoyama02 -g mail -M -K MAIL_DIR=/dev/null -s /sbin/nologin
passwd yokoyama02
> New password: yokoyama02
> Retype new password: yokoyama02
```

### NFSクライアント設定
```
ls -l /etc/ | grep fstab
cp -a /etc/fstab{,.bak}
ls -l /etc/ | grep fstab
```

fstab編集
```
vi /etc/fstab
    以下を追記
    NFSのIPアドレス:/share-mail /var/spool/mail nfs4 defaults 0 0

マウントする
    mount /var/spool/mail
確認
    df
        以下のような結果が得られる。
        NFSのIPアドレス:/share-mail   8310784 1639680   6671104  20% /var/spool/mail
```
---
## メールサーバ構築 2台目
---
Postfixのインストール/有効化
```
dnf install -y postfix
systemctl start postfix
systemctl status postfix
```

バックアップの作成
```
    ls -l /etc/postfix/ | grep main.cf
    cp -a /etc/postfix/main.cf{,.bak}
    ls -l /etc/postfix/ | grep main.cf
```
バックアップの編集
```
    vi /etc/postfix/main.cf

    以下の項目を追記

    myhostname = mail02.サブドメ
    mydomain = サブドメ
    myorigin = $mydomain
    mynetworks = 0.0.0.0/0
    mail_spool_directory = /var/spool/mail/

    以下の項目を変更
    inet_interfaces = all
    mydestination = $mydomain , $myhostname
```
Postfix再起動
```
systemctl restart postfix
systemctl status postfix
```

一つ目のメールアドレス作成
```
useradd yokoyama01 -g mail -M -K MAIL_DIR=/dev/null -s /sbin/nologin
passwd yokoyama01
> New password: yokoyama01
> Retype new password: yokoyama01
```

Dovecot インストール
```
dnf install -y dovecot
```
Dovecot 設定変更
```
ls -l /etc/dovecot/ | grep dovecot.conf
cp -a /etc/dovecot/dovecot.conf{,.bak}
ls -l /etc/dovecot/ | grep dovecot.conf

vi /etc/dovecot/dovecot.conf
    以下の項目を変更
    protocols = pop3
   
    以下の項目を追記
    mail_location = maildir:/var/spool/mail/%u/


```
SSL無効化
```
ls -l /etc/dovecot/conf.d/ | grep 10-ssl
cp -a /etc/dovecot/conf.d/10-ssl.conf{,.bak}
ls -l /etc/dovecot/conf.d/ | grep 10-ssl

vi /etc/dovecot/conf.d/10-ssl.conf
    変更前：ssl = required
    変更後：#ssl = required

ls -l /etc/dovecot/conf.d/ | grep 10-a
cp -a /etc/dovecot/conf.d/10-auth.conf{,.bak}
ls -l /etc/dovecot/conf.d/ | grep 10-a

vi /etc/dovecot/conf.d/10-auth.conf
    変更前：#disable_plaintext_auth = yes
    変更後：disable_plaintext_auth = no

```
Dovecot 起動
```
systemctl start dovecot
systemctl enable dovecot
systemctl is-enabled dovecot
```

telnetインストール
```
dnf install -y telnet
```

メールログの有効化
```
dnf install -y rsyslog
systemctl start rsyslog
systemctl status rsyslog
```
いったんメールが届くか確認

```
yokoyama01@サブドメ宛てにメールを送る
注意：DNS設定で優先度を変えてあげないとこっちに飛んでこないので以下の動きを確認する際は変更が必要
確認1
telnet mail02.サブドメ 110
user yokoyama01
pass yokoyama01
list

確認2
cd /var/spool/mail/
ls

yokoyama01ディレクトリがあるか確認
/newに新規メール
/curに今までのメール？が届いている(あんまり確認はしていないので多分)

htmlにフォーマットされているがとりあえずよしとする。


```
二つ目のメールアドレス作成
```
useradd yokoyama02 -g mail -M -K MAIL_DIR=/dev/null -s /sbin/nologin
passwd yokoyama02
> New password: yokoyama02
> Retype new password: yokoyama02
```

### NFSクライアント設定
```
ls -l /etc/ | grep fstab
cp -a /etc/fstab{,.bak}
ls -l /etc/ | grep fstab
```

fstab編集
```
vi /etc/fstab
    以下を追記
    NFSのIPアドレス:/share-mail /var/spool/mail nfs4 defaults 0 0

マウントする
    mount /var/spool/mail

確認
    df
        以下のような結果が得られる。
        NFSのIPアドレス:/share-mail   8310784 1639680   6671104  20% /var/spool/mail
```
---

## NFSサーバ構築　(今回はDNSサーバと同じサーバに構築)
---
共有ディレクトリの作成

```
mkdir /share-mail
```
共有ディレクトリ設定
```
vi /etc/exports

    以下を追記
    /share-mail クライアントのIP範囲(rw,no_root_squash)

systemctl start nfs-server
systemctl status nfs-server
```

## ちょい詰まりポイント？

で/var/spool/mailの権限が以下のようになっていた。
この場合だと、rootユーザしか書き込めなくて、メールがきてもパーミッションがないのでエラーになる（タイムアウト？になりそう）
```
[root@ip-10-0-1-238 spool]# ls -la
total 16
drwxr-xr-x.  6 root root    54 Jan 26 09:34 .
drwxr-xr-x. 19 root root   266 Jan 26 07:10 ..
drwx------.  3 root root    31 Jan 24 16:56 at
drwxr-xr-x.  2 root root     6 Jan 30  2023 lpd
drwxr-xr-x.  2 root root    27 Jan 26 10:29 mail
drwxr-xr-x. 16 root root 16384 Jan 26 09:34 postfix
```

なのでmailディレクトリの権限を変更する必要がある。
※777はあんまりよくないので、本当ならグループの変更とか、色々する必要がある
この手順書書き始めてから５時間くらい経過しているので許してほしい。
```
[root@ip-10-0-1-238 spool]# chmod 777 mail/
[root@ip-10-0-1-238 spool]# ls -l
total 16
drwx------.  3 root root    31 Jan 24 16:56 at
drwxr-xr-x.  2 root root     6 Jan 30  2023 lpd
drwxrwxrwx.  2 root root    27 Jan 26 10:29 mail
drwxr-xr-x. 16 root root 16384 Jan 26 09:34 postfix
```
```
chown :mail mail
後で追記：本当はこれでやりたかった。
```
---

## 以上です
特に解説はしていないのは教科書に書いてある作業しかほぼしていないからです。
ほかに詰まりポイントになりそうなところはコメント書いておきました。
横山の環境では動作確認できたので、参考程度に。


