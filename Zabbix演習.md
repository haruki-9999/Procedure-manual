# Zabbix演習
CVE

![説明文](aaaa)



## サーバの手順

タイムゾーンを日本にする
```bash
timedatectl set-timezone Asia/Tokyo
```
DBの準備
```bash 
dnf install mariadb105-server
 systemctl start mariadb
```
Zabbixリポジトリをインストールする
```bash
rpm -Uvh https://repo.zabbix.com/zabbix/7.4/release/amazonlinux/2023/noarch/zabbix-release-latest-7.4.amzn2023.noarch.rpm
dnf clean all
```
Zabbixサーバ、フロントエンド、エージェント2をインストールする
```bash
dnf install zabbix-server-mysql zabbix-web-mysql zabbix-apache-conf zabbix-sql-scripts zabbix-selinux-policy zabbix-agent2 zabbix-get zabbix-web-japanese
```
Zabbixエージェント2プラグインをインストールする
```bash
dnf install zabbix-agent2-plugin-mongodb zabbix-agent2-plugin-mssql zabbix-agent2-plugin-postgresql
```



設定を進めていく
```bash 
mysql -u root -p
```
DBにDB名やパスワードを付与してく
```bash 
CREATE DATABASE zabbix CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;
CREATE USER zabbix@localhost IDENTIFIED BY 'zabbix';
#'zabbix'部分は任意のパスワード
GRANT all privileges ON zabbix.* TO zabbix@localhost;
set global log_bin_trust_function_creators = 1;
FLUSH PRIVILEGES;
```
Zabbixサーバホストで初期スキーマとデータをインポートする
新しくパスワードが求められる
```bash
zcat /usr/share/zabbix/sql-scripts/mysql/server.sql.gz | mysql --default-character-set=utf8mb4 -uzabbix -p zabbix
```

データベーススキーマをインポートしたあとはlog_bin_trust_function_creators オプションを無効にする
```bash
mysql -uroot -p
set global log_bin_trust_function_creators = 0;
```
データベースを構築する
```bash
vi /etc/zabbix/zabbix_server.conf
```
/etc/zabbix/zabbix_server.confのバックラップを取る
```bash 
cp /etc/zabbix/zabbix_server.conf{,.org}
```
vi /etc/zabbix/zabbix_server.confを変更する
```bash
DBName=zabbix
DBUser=zabbix
DBPassword=zabbix(設定したDBパスワード)
```
vi /etc/zabbix/zabbix_agentd.confを保存する
```bash
vi /etc/zabbix/zabbix_agentd.conf
```
vi /etc/zabbix/zabbix_agentd.confをバックアップする
```bash
cp /etc/zabbix/zabbix_agentd.conf{,.org}
```
vi /etc/zabbix/zabbix_agentd.conf内を編集する
```bash
#ServerとActiveにはローカルホストのIPアドレスは必要ない
Server=<zabbix-server1 の IP>
ServerActive=<zabbix-server1 の IP>
Hostname=<自分自身のホスト名>
systemctl start zabbix-agent2
```
Apacheをインストールする
```bash
dnf install httpd
```
phpのインストール
```bash
dnf install php php-mysqlnd php-gd php-xml php-bcmath php-mbstring php-json
```
Apacheを起動する
```bash
systemctl start httpd
systemctl enable httpd
```
Zabbix Server 起動
```bash
systemctl start zabbix-server
systemctl enable zabbix-server
```

## エージェントの手順
プラットフォーム向けにZabbixをインストールを行う
```bash
rpm -Uvh https://repo.zabbix.com/zabbix/7.4/release/amazonlinux/2023/noarch/zabbix-release-latest-7.4.amzn2023.noarch.rpm
```
```bash
dnf clean all
```

Zabbixエージェント2をインストールする
```bash
dnf install zabbix-agent2
```
Zabbixエージェント２プラグインをインストールする
```bash 
dnf install zabbix-agent2-plugin-mongodb zabbix-agent2-plugin-mssql zabbix-agent2-plugin-postgresql
```
Zabbixエージェント2プロセスを開始する
プロセスを開始し、システムを起動時に開始させる
```bash 
systemctl restart zabbix-agent2
systemctl enable zabbix-agent2
```
エージェントの設定ファイルを編集する
```bash
vi /etc/zabbix/zabbix_agentd2.conf
#内容は秋のように変更する
Server=ZabbixServer の IP
ServerActive= ZabbixServer の IP
Hostname=監視対象の名前(分かりやすい名前を付ける)
Agent

#Server=127.0.0.1,44.243.252.82,172.31.60.140
#ServerActive=127.0.0.1,44.243.252.82,172.31.60.140
#上記のようにローカルホスト、プライベートIP、パブリックIPを入れる
```
Zabbixエージェントを起動する
```bash
systemctl start zabbix-agent2
systemctl enable zabbix-agent2
```
アクセスする
```bash
http://IPアドレス/zabbix
```

データベース設定ではパスワードを記入するだけで良い
↓
設定ではサーバ名を入力する
↓
ユーザ名とパスワードを入れる
ID Adimn
パス　zabbix

GUI操作
```bash
データ収集
↓
ホスト
↓
右上のホスト作成
↓
テンプレートとインターフェイスを設定する
↓
監視データからホスト、グラフ
↓
```
stressをインストールする
```bash
sudo dnf install stress -y
```
CPUに負荷をかける
```bash
stress --cpu 2 --timeout 60s
killall yes
```

CPU utilizationで負荷が上がっていることを確認する

