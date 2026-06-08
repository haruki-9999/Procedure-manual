# DBサーバ演習　※サーバ2台version
*******************
## LAMPサーバの準備
***********

1. AWS でEC2でインスタンスを起動させる

2. インストールされているソフトウェアを最新版にアップデートする（両方のサーバで実行する）
```bash
sudo dnf upgrade -y
```

3. Apacheウェブサーバの最新バージョンとAL2023ようにPHPパッケージをインストールする（1のサーバで実行する）
```bash
sudo dnf install -y httpd wget php-fpm php-mysqli php-json php php-devel
```


4. MariaDBソフトウェアパッケージをインストールする（2のサーバで実施する）
```bash
sudo dnf install mariadb105-server
```
パッケージの確認コマンド
```bash
sudo dnf info mariadb105
```
5. Apacheを起動する（1のサーバで行う）
```bash
sudo systemctl start httpd
```
6. システムがブートするたびにApacheウェブサーバが起動するようにする（1のサーバで実施する）
```bash
sudo systemctl enable httpd
```
httpdが有効であることを確認する
```bash
sudo systemctl is-enabled httpd
```

7. 両インスタンスでセキュリティーグループでHTTPを許可する（1のサーバで実施する）
※0.0.0.0/0は全ての通信を許可なので指定のIPアドレスを指定する
↓
下記のように検索をかける
```bash
http://44.255.100.40
```

## ファイル指定するには
1.ユーザをApacheグループに追加する
```bash
sudo usermod -a -G apache ec2-user
```
2. ログアウトして、再度接続して確認する
```bash
exit
groups
```
groupsで
*ec2-user adm wheel apache systemd-journal*
出てきたらOK！

3. /var/www とそのコンテンツのグループ所有権を apache グループに変更する（１のサーバで実施する）
```bash
sudo chown -R ec2-user:apache /var/www
```
4. グループに書き込みを許可を追加して、これからのサブディレクトリにグループIDを設定するには/var/wwwto
ディレクトリを許可する（1のサーバで実施する）
```bash
sudo chmod 2775 /var/www && find /var/www -type d -exec sudo chmod 2775 {} \;
```
5. グループ書き込み許可するには/var/wwwとサブディレクトリのファイル許可を再帰的に変更する（1のサーバで実施する）
```bash
find /var/www -type f -exec sudo chmod 0664 {} \;
```

## LAMPサーバをテストする
************

1. ApacheドキュメントルートでPHPファイルを作成する（1のサーバで実施する）
```bash
echo "<?php phpinfo(); ?>" > /var/www/html/phpinfo.php
```
2. ウェブブラウザで作製したファイルのURLを入力する
```bash
http://my.44.255.100.40/phpinfo.php
```
↓
PHP情報の確認ページが出てきたらOK

ページが表示されない場合は下記のコマンドで再度インスする
```bash
sudo dnf list installed httpd mariadb-server php-mysqlnd
```

3. phpinfo.php ファイルを削除する
削除しないとページの情報が危険に晒される危険性がある（1のサーバが実施する）
```bash
rm /var/www/html/phpinfo.php
```


## データベースサーバをセキュリティで保護する
**********

1. MariaDBサーバを起動する
```bash
sudo systemctl start mariadb
```

2. mysql_secure_installation を実行する
```bash
sudo mysql_secure_installation
```
⚠️ここで1のサーバにmariaDBを誤ってインストールしてしまった場合は確認＆削除をする方が良い
1のサーバでの確認コマンド
```bash
dnf list installed | grep mariadb
```
1のサーバで誤ってインストールしたDBを削除するコマンド
```bash
sudo dnf remove mariadb-server
```
プロンプトが表示されたらるーとアカウントのパスワードを入力する（2のサーバで実施する）

* 安全なパスワードを二つ入力
する
**下記は全て「Y」を入力する**
* 匿名ユーザーアカウントを削除する
* リモートルートログインを無効にする
* テストデータベースを削除する
* 権限テーブルを再ロードし、変更を保存する

## WordPressのインストール
***********

1. WordPressのパッケージをダウンロードしてインストールする（1のサーバで実施する）

2. 警告文が出てくる場合もある
下記に転載する
```bash
 WARNING:
  A newer release of "Amazon Linux" is available.

  Available Versions:
     
dnf upgrade --releasever=2023.0.20230202

    Release notes:
     https://aws.amazon.com

  Version 2023.0.20230204:
    Run the following command to update to 2023.0.20230204:

      dnf upgrade --releasever=2023.0.20230204 ... etc 
```
ベストプラクティスとして最新のOSにしておくことが環境内で競合が発生しない方法である

3. Wgetを使って最新のWordpressをダウンロードするコマンド（1のサーバで実行する）
```bash
wget https://wordpress.org/latest.tar.gz
```

4. インストールしたパッケージを解凍する（1のサーバで実施する）
```bash
tar -xzf latest.tar.gz
```

## WordPressインストール用にデータベースユーザとデータベースを作成する（1のサーバで実施する）

1. データベースおよびウェブサーバを起動する
* 1のサーバの場合
```bash
sudo systemctl start https
```
* 2のサーバの場合
```bash
sudo systemctl start mariadb
```
2. データベースサーバにrootユーザとしてログインする（2のサーバで実施する）
```bash
mysql -u root -p
```
3. MySQLデータベースのユーザとパスワードを作成する（2のサーバで実施する）
```bash
CREATE USER 'wordpress-user'@'uruhu9999' IDENTIFIED BY 'CL_katsushima';
```
4. データベースを作成する（2のサーバで実施する）
```bash
CREATE DATABASE `uruhu9999-db`;
```
5. WordPress ユーザーに対する完全な権限を付与する2のサーバで実施する）
```bash
GRANT ALL PRIVILEGES ON `uruhu9999-db`.* TO "uruhu9999"@"localhost";
```







