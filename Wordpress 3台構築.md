### WEBサーバでの作業
- 1台目のインスタンスを起動、分かりやすいようにホストネームやタブの色を変更（以後2台目3台目の手順には省略）
```
hostnamectl hostname webserver　など
```
- apacheのインストールおよび起動
```
dnf update -y

dnf install httpd -y

systemctl start httpd
systemctl status httpd
```
- apacheの起動確認→ブラウザにて***http://webサーバのグローバルIPアドレス***を検索→「It works!」の表示を確認。

- php-fpmをダウンロード（/etc/httpd/conf.d配下にphp.confを追加するため）
```
dnf install php-fpm -y
```
- php.confファイルの修正
vi /etc/httpd/conf.d/php.conf
下記に修正
```
    <FilesMatch \.(php|phar)$>
        SetHandler "proxy:fcgi://xxx.xxx.xxx.xxx(APサーバーのプライベートIP)"
    </FilesMatch>
```
※proxy:後はfcgi://の後にAPのプライベートIP。
- httpdの再起動
```
systemctl restart httpd 
```
- 念のためphp-fpmを停止
```
systemctl stop php-fpm
```

### APサーバでの作業
- phpのインストールおよび起動、mariadbサーバのインストール（sqlのコマンドを使用するため）
```
dnf install mariadb105-server  php-fpm php-mysqli php-json php-devel -y

systemctl start php-fpm
systemctl status php-fpm
```
- /etc/php-fpm.d/www.conf内の修正
下記に修正
```
;listen = /run/php-fpm/www.sock
→listen = xxx.xxx.xxx.xxx(APサーバーのプライベートIP):8000
;listen.allowed_clients = 127.0.0.1
→listen.allowed_clients = xxx.xxx.xxx.xxx(WebサーバーのプライベートIP)
```

- php-fpmの再起動
```
systemctl restart php-fpm
```
- phpinfoを表示するために、phpinfo.phpファイルを作成（APサーバに配置する必要あり。WEBサーバには不要）
```
echo "<?php phpinfo(); ?>" > /var/www/html/phpinfo.php
```
- apacheによりphpが表示できているか確認→ブラウザにて***http://webサーバのパブリックIPアドレス/phpinfo.php***　を検索→phpinfoページの表示を確認。

外観をインストールするために必要
chown -R apache:apache /var/www/html/

## APサーバとDBサーバの接続
### DBサーバでの作業
- mariadbのインストールおよび起動と本番環境の準備
```
dnf update -y
dnf install mariadb105-server -y
systemctl start mariadb 
systemctl status mariadb

※飛ばしてもよい
mysql_secure_installation
--------------------------------------------------
１.rootパスの入力、デフォルトは設定されていないためエンターキー
Enter current password for root (enter for none): Enter
２.接続をリモート可能にするか設定、ローカル接続のみにする場合はY
Switch to unix_socket authentication [Y/n] Y
３.rootパスワードを変更する場合Y
Change the root password? [Y/n] Y
４.匿名ユーザアカウントを削除する場合Y
Remove anonymous users? [Y/n] Y
５.リモートルートログインを無効にする場合Y
Disallow root login remotely? [Y/n] Y
６.テストデータベースを削除する場合Y
Remove test database and access to it? [Y/n] Y
７.権限テーブルをリロードし、変更を保存する場合Y
Reload privilege tables now? [Y/n] 
```
- sqlにrootユーザとしてログイン　ユーザとデータベースの作成および、権限の付与
```
mysql -u root -p
MariaDB [(none)]> CREATE USER 'wordpress-user'@'xxx.xxx.xxx.xxx(APサーバーのプライベートIP)' IDENTIFIED BY 'wordpress-password';
MariaDB [(none)]> CREATE DATABASE `wordpress-db`;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON `wordpress-db`.* TO "wordpress-user"@"APサーバーのプライベートIP";
MariaDB [(none)]> FLUSH PRIVILEGES;
```
※　誤字やコマンド終了のセミコロンに注意。

### APサーバでの作業
- Wordpressのダウンロードと解凍・バックアップの作成
```
cd /usr/local 
wget https://wordpress.org/latest.tar.gz
tar -xzf latest.tar.gz

cp wordpress/wp-config-sample.php wordpress/wp-config.php
```
- Wordpress配下を/var/www/html配下へ移動
```
mv wordpress/* /var/www/html/
```

- wp-config.phpの修正
```
vi /var/www/html/wp-config.php
===変更箇所==========================
define( 'DB_NAME', 'wordpress-db' );

/** Database username */
define( 'DB_USER', 'wordpress-user' );

/** Database password */
define( 'DB_PASSWORD', 'wordpress-password' );

/** Database hostname */
define( 'DB_HOST', 'xxx.xxx.xxx.xxx(DBサーバーのプライベートIP)' );
```
- APサーバとDBサーバが接続しているかを確認
```
mysql -u wordpress-user -h DBサーバのプライベートID -p wordpress-db
```
## ブラウザでのWordpressの表示準備
### WEBサーバ上での作業
- Wordpressのダウンロードと解凍・バックアップの作成
```
cd /usr/local 
wget https://wordpress.org/latest.tar.gz
tar -xzf latest.tar.gz

cp wordpress/wp-config-sample.php wordpress/wp-config.php
```
- Wordpress配下を/var/www/html配下へ移動
```
mv wordpress/* /var/www/html/
```
※WordpressはAPサーバとWEBサーバの両方のhtml配下に置く必要がある

- すべてのサーバを再起動

ブラウザにて***http://WEBサーバのIPアドレス***の検索
表示できていれば成功。

ユーザ：wordpress-user
パス：wordpress-password