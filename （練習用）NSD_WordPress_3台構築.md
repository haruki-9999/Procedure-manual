DNS / Apache / PHP / MariaDB / WordPress

目的：NSDで内部DNS名を解決し、Webサーバー上のWordPressからDBサーバー上のMariaDBへ接続する構成を構築する。

対象OS：Amazon Linux 2023を想定。別OSの場合はパッケージ名や設定ファイルの場所が異なる場合があります。

| **重要：**WordPress公式の推奨要件は PHP 8.3以上、MariaDB 10.6以上またはMySQL 8.0以上、Apache/Nginxとmod_rewrite、HTTPS対応です。本手順では検証環境向けにHTTPで開始し、本番ではHTTPS化を必須とします。 |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

# 1. 構成概要

3台構成は以下の通りです。NSDは権威DNSサーバーとして team.local ゾーンを持ちます。

| **サーバー** | **役割**      | **ホスト名**     | **IP例**  | **主なミドルウェア**   |
|--------------|---------------|------------------|-----------|------------------------|
| dns01        | 名前解決      | dns01.team.local | 10.0.1.10 | NSD                    |
| web01        | Web/WordPress | www.team.local   | 10.0.1.20 | Apache, PHP, WordPress |
| db01         | DB            | db.team.local    | 10.0.1.30 | MariaDB                |

インターネット/管理PC  
↓ HTTP/SSH  
web01 Apache + PHP + WordPress  
↓ DB接続: db.team.local:3306  
db01 MariaDB  
  
名前解決:  
web01 / db01 / 管理PC → dns01(NSD) → www.team.local, db.team.local を解決

# 2. 事前準備

## 2.1 セキュリティグループ / FW設計

| **宛先** | **ポート** | **許可元**             | **用途**      |
|----------|------------|------------------------|---------------|
| dns01    | UDP/TCP 53 | VPC内または検証端末    | DNS問い合わせ |
| web01    | TCP 80     | 検証端末または必要範囲 | WordPress表示 |
| web01    | TCP 443    | 本番では必須           | HTTPS         |
| db01     | TCP 3306   | web01のみ              | MariaDB接続   |
| 全台     | TCP 22     | 管理端末のみ           | SSH管理       |

## 2.2 全サーバー共通作業

sudo dnf update -y  
sudo hostnamectl set-hostname \<各サーバー名\>  
  
\# 例  
a hostnamectl set-hostname dns01  
\# 実際は a を付けずに sudo hostnamectl set-hostname dns01 のように実行してください。

| **補足：**手順中のIPアドレスは例です。実環境のプライベートIPに置き換えてください。 |
|------------------------------------------------------------------------------------|

# 3. dns01：NSD構築

NSDは権威DNSサーバーです。自分が管理するゾーンの問い合わせに回答します。ここでは team.local のAレコードを返す構成にします。

## 3.1 NSDインストール

sudo dnf search nsd  
sudo dnf install -y nsd bind-utils

## 3.2 ゾーンファイル作成

sudo mkdir -p /etc/nsd/zones  
sudo vi /etc/nsd/zones/team.local.zone

\$ORIGIN team.local.  
\$TTL 300  
  
@ IN SOA dns01.team.local. admin.team.local. (  
2026060401 ; serial  
3600 ; refresh  
900 ; retry  
604800 ; expire  
300 ; minimum  
)  
  
IN NS dns01.team.local.  
  
dns01 IN A 10.0.1.10  
www IN A 10.0.1.20  
db IN A 10.0.1.30

## 3.3 nsd.conf設定

sudo cp -a /etc/nsd/nsd.conf /etc/nsd/nsd.conf.org  
sudo vi /etc/nsd/nsd.conf

server:  
ip-address: 0.0.0.0  
port: 53  
  
zone:  
name: "team.local"  
zonefile: "/etc/nsd/zones/team.local.zone"

## 3.4 文法チェックと起動

sudo nsd-checkzone team.local /etc/nsd/zones/team.local.zone  
sudo systemctl enable --now nsd  
sudo systemctl status nsd  
sudo ss -lntup \| grep ':53'

## 3.5 DNS動作確認

dig @10.0.1.10 www.team.local +short  
dig @10.0.1.10 db.team.local +short

期待値：それぞれ 10.0.1.20、10.0.1.30 が返ること。

# 4. db01：MariaDB構築

WordPress用のデータベースと接続ユーザーを作成します。

## 4.1 MariaDBインストール

| **バージョン確認：**WordPress公式推奨はMariaDB 10.6以上です。Amazon Linux 2023で利用可能なMariaDBパッケージを確認し、mariadb1011-server等が使える場合はそれを優先します。 |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

sudo dnf list available 'mariadb\*' \| grep -E 'server\|10\\11\|10\\6'  
  
\# MariaDB 10.11系が利用できる場合の例  
sudo dnf install -y mariadb1011-server  
  
\# もし上記が見つからず検証目的で進める場合のみ  
\# sudo dnf install -y mariadb105-server

sudo systemctl enable --now mariadb  
sudo systemctl status mariadb  
mariadb --version

## 4.2 初期セキュリティ設定

sudo mariadb-secure-installation

推奨：anonymous user削除、remote root login禁止、test database削除、privilege table reloadを実施します。

## 4.3 DBとユーザー作成

sudo mariadb

CREATE DATABASE wordpress CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;  
CREATE USER 'wpuser'@'10.0.1.20' IDENTIFIED BY 'StrongPassword123!';  
GRANT ALL PRIVILEGES ON wordpress.\* TO 'wpuser'@'10.0.1.20';  
FLUSH PRIVILEGES;  
EXIT;

| **注意：**パスワードは必ず本番用の強固なものへ変更してください。手順書内の値は検証用サンプルです。 |
|----------------------------------------------------------------------------------------------------|

## 4.4 MariaDB待受設定

sudo ss -lntup \| grep 3306  
sudo grep -Rni "bind-address" /etc/my.cnf /etc/my.cnf.d/ 2\>/dev/null

sudo vi /etc/my.cnf.d/mariadb-server.cnf  
  
\# \[mysqld\] に以下を設定  
bind-address=0.0.0.0

sudo systemctl restart mariadb  
sudo ss -lntup \| grep 3306

# 5. web01：Apache / PHP / WordPress構築

Apache、PHP、WordPress本体をインストールし、db.team.local のMariaDBへ接続します。

## 5.1 Apache / PHPインストール

sudo dnf update -y  
sudo dnf install -y httpd php php-mysqli php-json php-gd php-xml php-mbstring php-curl tar wget  
php -v  
httpd -v

sudo systemctl enable --now httpd  
sudo systemctl status httpd

## 5.2 Apacheのmod_rewrite確認

httpd -M \| grep rewrite

出力に rewrite_module があればOKです。WordPressのパーマリンク利用時に必要になります。

## 5.3 DNS参照先をdns01に変更

nmcli con show  
  
\# 接続名が "System eth0" の場合  
sudo nmcli con mod "System eth0" ipv4.dns "10.0.1.10"  
sudo nmcli con up "System eth0"  
  
cat /etc/resolv.conf  
getent hosts db.team.local  
getent hosts www.team.local

## 5.4 WordPressダウンロードと配置

cd /tmp  
wget https://wordpress.org/latest.tar.gz  
tar zxf latest.tar.gz  
sudo rsync -av /tmp/wordpress/ /var/www/html/  
sudo chown -R apache:apache /var/www/html  
sudo find /var/www/html -type d -exec chmod 755 {} \\  
sudo find /var/www/html -type f -exec chmod 644 {} \\

## 5.5 wp-config.php作成

cd /var/www/html  
sudo cp wp-config-sample.php wp-config.php  
sudo vi wp-config.php

define( 'DB_NAME', 'wordpress' );  
define( 'DB_USER', 'wpuser' );  
define( 'DB_PASSWORD', 'StrongPassword123!' );  
define( 'DB_HOST', 'db.team.local' );  
define( 'DB_CHARSET', 'utf8mb4' );  
define( 'DB_COLLATE', '' );

| **推奨：**AUTH_KEY等の認証用ユニークキーはWordPress公式のsecret-key serviceで生成した値へ置き換えてください。 |
|---------------------------------------------------------------------------------------------------------------|

## 5.6 Apache設定

sudo vi /etc/httpd/conf.d/wordpress.conf

\<VirtualHost \*:80\>  
ServerName www.team.local  
DocumentRoot /var/www/html  
  
\<Directory /var/www/html\>  
AllowOverride All  
Require all granted  
\</Directory\>  
  
ErrorLog /var/log/httpd/wordpress_error.log  
CustomLog /var/log/httpd/wordpress_access.log combined  
\</VirtualHost\>

sudo apachectl configtest  
sudo systemctl restart httpd

## 5.7 DB接続テスト

sudo dnf install -y mariadb105  
mysql -h db.team.local -u wpuser -p wordpress

ログインできれば、web01 から db.team.local を名前解決し、MariaDBへ接続できています。

# 6. 管理PCまたは検証端末からの確認

ブラウザで www.team.local にアクセスするには、検証端末も dns01 をDNSサーバーとして参照できる必要があります。

\# DNS確認例  
nslookup www.team.local 10.0.1.10  
  
\# ブラウザ確認  
http://www.team.local/

| **一時確認：**端末のDNS設定を変更できない場合は、検証端末のhostsファイルに 10.0.1.20 www.team.local を一時追加して確認できます。ただしNSDの名前解決確認にはならないため、最終確認はDNS問い合わせで実施してください。 |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

# 7. 最終確認チェックリスト

| **対象** | **確認コマンド**                              | **OK基準**                  |
|----------|-----------------------------------------------|-----------------------------|
| dns01    | sudo systemctl status nsd                     | NSDがactiveであること       |
| dns01    | dig @10.0.1.10 www.team.local +short          | 10.0.1.20が返ること         |
| dns01    | dig @10.0.1.10 db.team.local +short           | 10.0.1.30が返ること         |
| web01    | systemctl status httpd                        | Apacheがactiveであること    |
| web01    | getent hosts db.team.local                    | DBサーバーのIPが返ること    |
| web01    | mysql -h db.team.local -u wpuser -p wordpress | MariaDBへログインできること |
| db01     | systemctl status mariadb                      | MariaDBがactiveであること   |
| db01     | sudo ss -lntup \| grep 3306                   | 3306番で待受していること    |

# 8. よくあるエラーと確認ポイント

### www.team.local が開けない

端末がdns01をDNS参照していない、またはFWで53/80が閉じている可能性があります。nslookup www.team.local 10.0.1.10 と curl http://10.0.1.20/ を確認します。

### Error establishing a database connection

DB名、DBユーザー、パスワード、DB_HOST、MariaDBのbind-address、FWの3306許可を確認します。

### WordPressのパーマリンクが404になる

Apacheのrewrite_moduleとAllowOverride All、.htaccessの存在を確認します。

### NSDが起動しない

nsd-checkzoneとjournalctl -u nsd -xeでゾーンファイルやnsd.confの文法エラーを確認します。

# 9. 本番化する場合の追加対応

- HTTPS化：Let’s EncryptやACM/ALBなどを利用し、HTTPのみの運用は避けます。

- DB接続制限：db01の3306番はweb01からのみ許可します。

- WordPress管理画面保護：強固なパスワード、MFA、IP制限、不要プラグイン削除を検討します。

- バックアップ：WordPressファイルとMariaDBデータを定期バックアップします。

- 監視：httpd、mariadb、nsdのプロセス監視、ポート監視、ディスク監視を設定します。

- DNS：team.localは検証向けです。本番では正式に管理しているドメインを使用します。

# 10. 参考資料

1.  Amazon Linux 2023 User Guide: Tutorial - Install a LAMP server on AL2023. https://docs.aws.amazon.com/linux/al2023/ug/ec2-lamp-amazon-linux-2023.html

2.  Amazon Linux 2023 User Guide: PHP in AL2023. https://docs.aws.amazon.com/linux/al2023/ug/php.html

3.  WordPress.org: Requirements. https://wordpress.org/about/requirements/

4.  NLnet Labs NSD Documentation. https://nsd.docs.nlnetlabs.nl/

5.  NLnet Labs nsd.conf manual. https://www.nlnetlabs.nl/documentation/nsd/nsd.conf/

6.  MariaDB Documentation: Installing MariaDB with yum/dnf. https://mariadb.com/docs/server/server-management/install-and-upgrade-mariadb/installing-mariadb/binary-packages/rpm/yum
