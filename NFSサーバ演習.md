********
# NFSサーバ演習
********

## Postfixのインストール

```bash
dnf install postfix
```
postfixを起動する
```bash
systemctl start postfix
```
下記のコマンドで正常にインストール、起動しているか確認することができる
```bash
ps -ef | grep postfix
netstat -ln | grep 25
```

その後にメールを送信するmailをインストールする
```bash
dnf install mailx
```
→outlookやGmailみたいなものでコマンドを打ってメール考査が可能になる

## Postfixの設定
Postfixの設定ファイルは「/etc/postfix/main.cf」である
削除しないようにバックアップを作成する
```bash
cp /etc/postfix/main.cf /etc/postfix/main.cf.
```
バックアップを確認するコマンド
```bash
ls -l /etc/postfix/main.cf.`date
```
編集前の設定はコメントアウトが多く見づらいのでコメントアウトを削除したファイルを削除する
```bash
grep -v ^# /etc/postfix/main.cf | cat -s > /tmp/main.cf
cp /tmp/main.cf /etc/postfix/main.cf
```
postfixのファイルを編集する
```bash
vi /etc/postfix/main.cf
```





