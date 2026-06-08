*******
# Mail演習
*******

## Postfixインストール
下記コマンドを打つ
```bash
dnf install postfix
```
インストールできたらスタートする
```bash
systemctl start postfix
```
動いているPostfixを全て表示する
```bash
ps -ef | grep postfix
```
ポート番号25が空いているか確認する
netstat -ln
netstat：ネットワーク状態を確認するコマンド（Linux標準）
-l：LISTEN（待ち受け中）のポートのみ表示
-n：ポート番号やIPを名前解決せず数値で表示
```bash
netstat -ln | grep 25
```

メールを送信するmailを送信する
```bash 
dnf install mailx
```

## Postfixの設定
バックアップを取って非常事態に備える
同じディレクトリに格納しないでバックアップ専用ディレクトリに格納する
```bash 
mkdir backup
cp /etc/postfix/main.cf /root/backup/
ls /root/backup
```
見ずらいコメントアウトの削除
```bash
grep -v ^# /etc/postfix/main.cf | cat -s > /tmp/main.cf
```
/empを/etcにコピーする
```bash
cp /tmp/main.cf /etc/postfix/main.cf
```

Postfix（サーバ）の設定ファイルを編集する
```bash
vi /etc/postfix/main.cf
```
編集するに当たって追記や編集を行う
追記をして２重で動いている場合、うまく作動しないので注意する
※コメントアウトしていれば重複しても大丈夫
```bash
- myhostname
myhostname = mail.haruki.local
- mydomain 
mydomain = haruki.local
- myorigin = $myhostname
- inet_interfaces
inet_interfaces = all
- mydestination
mydestination = $mydomain , $myhostname
- mynetworks 
mynetworks = 172.16.0.0/16, 127.0.0.1
- Maildir 形式にします
mail_spool_directory = /var/spool/mail/
- 完了したらPostfixを再起動
systemctl restart postfix
```
## DNSサーバの設定
インストールする
```bash
dnf install bind
```
インストール後はファイルを編集する
```bash
vi /etc/named.conf
```
変更する
```bash
listen-on port 53 { 127.0.0.1; }; 
listen-on-v6 port 53 { ::1; }; 
allow-query { localhost; }; 
allow-query { any; }; 
```
一番下に名前.localのゾーン情報を定義する
```bash
zone "自分の名前.local" IN {
type master;
file "/var/named/自分の名前.local.zone";
};
```

named.confに間違いはないか確認するコマンド
※何も表示されなければOK
```bash 
named-checkconf
```
正引きのゾーンデータベースファイルを新規作成する
```bash 
vi /var/named/haruki.local.conf.zone
```
作成したゾーンファイルを/backupにコピーする
root配下の/backupディレクトリにコピーされる
```bash
cp /var/named/haruki.local.conf.zone{,.bak} /root/backup/
```
再度vi /var/named/haruki.local.conf.zoneを起動させ、編集する
下記を打ちこむ
```bash 
$TTL 3600
@ IN SOA ns.自分の名前.local. test.gmail.com. (
20210401 ; serial
3600 ; refresh
3600 ; retry
3600 ; expire
3600 ) ; minimum
　IN NS ns.自分の名前.local.
　IN MX 10 mail.自分の名前.local.
ns IN A 172.16.***.*** ※自分のプライベートIPアドレス
mail IN A 172.16.***.*** ※送りたい人のiPアドレス
・
・
・
下には送りたい人がいれば複数追記することもできる
```
下記コマンドで構文に間違いがないか確認する
```bash
named-checkzone haruki.local /var/named/haruki.local.zone 
```
namedを起動させる
```bash
systemctl start named
```
DNSサーバの問い合わせ先を変更（送りたい方向へ変更）
```bash
vi /etc/systemd/resolved.conf
```
障害があったときようにバックアップを取る
```bash 
cp /etc/systemd/resolved.conf{,.bak} /root/backup/
```
再度vi /etc/systemd/resolved.confを開き、変更を行う

再起動を行う
```bash 
systemctl restart systemd-resolved.service
```
自分にmailを送ってみる
```bash 
mail -s こんにちは　root@haruki.local
```

mailコマンドで来ていることを確認する
```bash
mail
```

ログ確認するためにsyslogをインストールし、スタートする
```bash
dnf install rsyslog
systemctl start rsyslog
```
mailのログを確認する
sentの文字があればOK
```bash
tail /var/log/maillog
```




