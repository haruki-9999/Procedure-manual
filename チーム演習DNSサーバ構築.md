# チーム演習　DNSサーバ（外部）

SSHしてからルートの権限を与えてインスタンスにbindをインストールする
```bash
sudo su -
dnf install bind
```

インストール後に起動させる
```bash
systemctl start named
```
起動していることを確認する
```bash
systemctl status named
```
namedファイルを作成して念のためにバックアップを作成する
下記コマンドで作成して保存する
```bash
vi /etc/named.conf
```
作成したらバックアップ先のディレクトリを作成する
```bash
mkdir backup
```
作成したディレクトリにコピーする
```bash
cp /etc/named.conf /root/backup/
```
再度設定ファイルを開いて編集する
```bash
vi /etc/named.conf 
```

ゾーンデータファイルを作成する
```bash
vi /var/named/teamc.entrycl.net.zone
```
作成したらbackupディレクトリにコピーする
```bash
cp /var/named/teamc.entrycl.net.zone /root/backup/teamc.entrycl.net.zone.bak
```
コピーができたらゾーンファイルの編集を行う
```bash
vi /var/named/teamc.entrycl.net.zone
```
下記は編集内容
※編集事項は必要に応じて作成する
```bash
$TTL 3600
@ IN SOA ns.teamc.entrycl.net. test.gmail.com. (
20220401 ; serial
3600 ; refresh
3600 ; retry
3600 ; expire
3600 ) ; minimum
IN NS ns.teamc.entrycl.net.
ns IN A
www IN A
```


zone "teamc.entrycl.net" IN {
    type slave;
    masters { 10.0.1.201; };
    file "slaves/teamc.entrycl.net";
};

