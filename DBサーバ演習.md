# DBサーバ演習手順書
*********
## LAMPサーバを準備する
*********
1.EC2にSSHさせる
```bash
ssh -i ~/CL_katsushima_h.pem ec2-user@18.237.157.113
```
2.LAMPサーバをインストールする
```bash
sudo dnf upgrade -y
```
↓
Dependencies resolved.
Nothing to do.
Complete!が出たらOK

3.Apacheウェブサーバの最新バージョンとAL2023用のPHPパッケージをインストールする
```bash
sudo dnf install -y httpd wget php-fpm php-mysqli php-json php php-devel
```
↓
Complete!でOK

4.MariaDBをインストールする
```bash
sudo dnf install mariadb105-server
```
↓
Complete!でOK

現在のパッケージのバージョンを確認する
```bash
sudo dnf info package_name
```
package_nameには調べたいものを入れる

5.6ApacheWebサーバを起動させ、自動で動くようにする
```bash
sudo systemctl start httpd
```
```bash
sudo systemctl enable httpd
```
このコマンドで状態を確認できる
```bash
sudo systemctl is-enabled httpd
```

7.AWS EC2のセキュリティグループからHTTP（80）を許可する
※0.0.0.0/0はセキュリティ的に危険なので避ける（テスト環境で短時間であればOK）

ec2userアカウントでこのディレクトリで複数の操作をすることを許可するにはディレクトおりに
所有権とアクセス許可を変更する必要がある
* ユーザをapacheグループに追加する
```bash
sudo usermod -a -G apache ec2-user
```

* exitで抜けてgroupsコマンドで確認してみる
→ec2-user adm wheel apache systemd-journalこれが出てきたらOK

* /var/wwwとコンテンツの所有権をapacheグループにする
```bash
sudo chown -R ec2-user:apache /var/www
```

* グループ書き込み許可をして/var/wwwサブディレクトリのディレクトリ許可をする
```bash
sudo chmod 2775 /var/www && find /var/www -type d -exec sudo chmod 2775 {} \;
```

* グループ書き込み許可をするには?/var/wwwとサブディレクトリの
ファイル許可を再帰的に処理する
※再帰的とは自分自身を使って処理を繰り返すことである
```bash
find /var/www -type f -exec sudo chmod 0664 {} \;
```
*********
## LAMPサーバをテストする
*********
1.Apacheドキュメントルートを作成する
```bash
echo "<?php phpinfo(); ?>" > /var/www/html/phpinfo.php
```

2.ウェブブラウザで確認する
*例）http://18.237.157.113/phpinfo.php*
→IPアドレスは自身のEC2インスタンスのパブリックIPアドレスである

ページがうまく表示されていない場合は全て必要なパッケージをインストールする必要がある
```bash
sudo dnf list installed httpd mariadb-server php-mysqlnd
```
3.phpinfo.php ファイルを削除する
セキュリティ上、PHPの情報が入っているファイルは削除しないと攻撃者に情報を取られる
```bash
rm /var/www/html/phpinfo.php
```
*********
## データベースサーバをででキュリティを保護する
***********


1.MariaDBを起動する
```bash
sudo systemctl start mariadb
```
2.mysql_secure_installation を実行する
```bash
sudo mysql_secure_installation
```
* 新しいパスワードを入する
* 匿名ユーザアカウント、リモートログイン、テストデータベース、権限テーブルを再リロードし、保存するを全て「Y」にする
**********
# WordPressブログでホストする
**********

## WordPressのインストール

1. 次のコマンドでパッケージをインストールする
```bash
dnf install wget php-mysqlnd httpd php-fpm php-mysqli mariadb105-server php-json php php-devel -y
```
↓
権限なしと出てきたら
```bash
sudo
```
コマンドを追加する

2. wgetコマンドを使って最新のWordpressをインストールする
下記のコマンドで最新リリースが必ずインストールされる
```bash
wget https://wordpress.org/latest.tar.gz
```

*解凍する
```bash
tar -xzf latest.tar.gz
```

## WordPressインストール用にデータベースユーザとデータベースを作成
1. データベースおよびウェブサーバを起動する
```bash
sudo systemctl start mariadb httpd
```
2. データベースサーバにrootユーザとしてログインする
```bash
mysql -u root -p
```

*パスワードを聞かれるが、これはSQLをインストールした時のパスワードである*

3.MySQLデータベースのユーザとパスワードを作成する
```bash
CREATE USER 'wordpress-user'@'localhost' IDENTIFIED BY 'your_strong_password';
```
'wordpress-user'と 'your_strong_password'は任意のものに変更する

4. データベースを作成する
```bash
CREATE DATABASE `wordpress-db`;
```
↓
'wordpress-db'の部分は任意で決める
*※権限を付与するときに使うので控えておく*

5. 作成したデータベースに完全な権限を付与する
```bash
GRANT ALL PRIVILEGES ON `wordpress-db`.* TO "wordpress-user"@"localhost";
```
↓
''中は決めた任意のデータベース名とウユーザー名にする

6. 全ての変更を有効にするために、データベースをフラッシュする
```bash
FLUSH PRIVILEGES;
```
7. mysqlクライアントを修了する
```bash
exit
```

### wp-config.phpファイル作成と編集を行うには

1. wp-config-sample.php ファイルを wp-config.php という名前でコピーする
元の構成ファイルが作成され、下のファイルがバックアップされる
```bash
cp wordpress/wp-config-sample.php wordpress/wp-config.php
```

2. テキストエディタを使ってvimファイルを編集し、インストール用の値を入力する
```bash
vi wordpress/wp-config.php
```
* DB_MANE
* DB_USER
* DB_PASSWORD
上記の項目を変更した内容に書き換える

自身のパブリックIPアドレスで検索をかけるとHPに飛べる

## Word PressファイルをApacheドキュメントの下にインストールできるには
* インストールファイルをウェブサーバのドキュメントルートのコピーを行う
```bash
cp -r wordpress/* /var/www/html/
```
* Wordpressをドキュメントルートに￥の下の別のディレクトリで実行する場合、別のディレクトリを作成してからそこにファイルをコピーする
```bash
mkdir /var/www/html/blog
cp -r wordpress/* /var/www/html/blog/
```
## 下記のように検索をかけてログイン画面が出れば成功！！

```bash
http://18.237.157.113/wp-admin/install.php
```


### ⚠️WordpressがApacheドキュメント下にインストールできていないうっかり忘れが多い




# データベースに入れない状況の障害の解消手順
MariaDB ログイン障害の解消手順書
1. 現象の確認

MySQL（MariaDB）クライアントから以下のコマンドを実行した際、エラーが発生した。

実行コマンド
```bash
mysql -u haruki -h 172.31.33.204 -p haruki_db
```
発生エラー: ERROR 2002 (HY000): Can't connect to MySQL server on '172.31.33.204' (115)

原因分析: エラーコード 115 (Connection timed out) は、サーバーに通信が届いていない、もしくはサーバー側で応答できる状態にないことを示す。

2. サービス状態の確認と復旧

サーバーにログインし、データベースサービスの状態を確認した。

確認コマンド
```bash
 systemctl status mariadb
```

状態: inactive (dead) （停止中）

対応: 以下のコマンドでサービスを起動した。

```bash
systemctl start mariadb
```
3. ユーザー権限（Host設定）の修正

サービス起動後、再度ログインを試みたが、アクセス拒否が発生した。

発生エラー: ERROR 1698 (28000): Access denied for user 'haruki'@'localhost'

原因分析: mysql.user テーブルを確認したところ、haruki ユーザーの許可ホストが 172.31.33.204（特定のIP）に限定されており、サーバー内部（localhost）からの接続が許可されていなかった。

4. 権限の変更と反映

どこからでも（サーバー内部・外部両方）接続できるように、ユーザーのホスト情報を変更した。

管理者（root）でログイン:

```bash
mysql -u root
```
ホスト情報の書き換え（RENAME USER）:

SQL
-- 172.31.33.204専用だったユーザーを「どこからでも（%）」に変更
```bash
RENAME USER 'haruki'@'172.31.33.204' TO 'haruki'@'%';
```
-- 権限を即時反映
```bash
FLUSH PRIVILEGES;
```
5. 最終確認

修正後、再度ログインを試行し、正常にデータベースへ接続できることを確認した。

コマンド
```bash
 mysql -u haruki -p haruki_db
```
パスワード
```bash
 haruki
```
結果: ログイン成功。
↓
権限が与えたれなくてError establishing a database connectionが出てくるなら下記のコマンドを参考に権限を解放しておく
```bash
 GRANT ALL PRIVILEGES ON `haruki_db`.* TO 'haruki'@'172.31.%' IDENTIFIED BY 'haruki';
 ```
 

