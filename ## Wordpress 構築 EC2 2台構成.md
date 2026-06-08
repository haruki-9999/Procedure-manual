## Wordpress　構築　EC2 2台構成
---

今回の構成では、EC2を2台立てます。
それぞれ、1台目にWeb,APを2台目にDBを入れて行います。

---

前提としてWordpressの構築にはWeb3層構造が必須です。

Web/AP/DBの3つが必要になります。
今回はWebとしてApache
APとしてPHP
DBとしてmariadbを使いますので、この3つはインストールが必須になります。

---
## 初めに
- まず、EC2を2台建てましょう
- 建てた後、それぞれのパブリックIPとプライベートIPを控えておきましょう
- WebbAP
  - パブリック：xxxxx
  - プライベート：xxxxx
- DB
  - パブリック：xxxxx
  - プライベート:xxxxx

## EC2 1台目 WEBとAPの設定
**WebAPの設定**
- まずややこしいのでホスト名を変えましょう
```hostnamectl set-hostname webap```
- Apacheのインストールと起動をします。
```dnf update -y```
```dnf install -y httpd```
```systemctl start httpd```
  -  これを実行した後はhttpdのstatusを確認して起動していることを確認しましょう。
- PHPをwebページに表示して、動作しているなぁと確認するために以下のコマンドを実行します。
```echo "<?php phpinfo(); ?>" > /var/www/html/phpinfo.php```
```chown apache:apache /var/www/html/phpinfo.php```
    - コマンドを少し解説すると、まず、/var/www/html/phpinfo.php。これはブラウザにページを表示するのに必要です。例えば、/var/www/html/yokoyama.htmlというファイルを作るとyokoyama.htmlの中身がブラウザで表示されます。というように/var/www/html/配下にファイルを入れるとブラウザに好きなページを表示させることができる。と覚えておきましょう。
    - で、今回やっているのは/var/www/html/phpinfo.phpに対して、php phpinfoを記載することでPHPのバージョン等をブラウザで表示させることができます。
    - chown チェンジオーナーはこのファイルをApacheが使えるように設定しています。ここがrootのままだと、Apacheがページにアクセス出来ず、エラーになります。

- wordpressをインストールします。
```wget https://wordpress.org/latest.tar.gz```
- 解凍します
```tar -xzf latest.tar.gz```

- wordpressの設定ファイルのバックアップをとります。
```cp wordpress/wp-config-sample.php wordpress/wp-config.php```
  - 一応lsコマンドでバックアップがとれているか確認しておきましょう。

- 設定ファイルを編集します。
```vi wordpress/wp-config.php```

- 下記に修正
  - define( 'DB_NAME', 'wordpress-database' );
  - define( 'DB_USER', 'wordpress-user' );
  - define( 'DB_PASSWORD', 'wordpress-password' );
  - define( 'DB_HOST', '(DBサーバーのプライベートIP)' );
それぞれ、データベース、ユーザー、パスワードはどこかに控えておきましょう。

- 管理しやすいように？wordpress配下のファイルを/var/www/html/に移動させておきます。/var/www/html/に移動させないと機能しない？かも
```mv wordpress/* /var/www/html/```

- PHP関連の必要なものをインストールします。また、mariadbもインストールしていますが、これはDBとして使いたいわけではなく、msqlコマンドを打てるようにしたい、という意味合いが強いです。
```dnf install mariadb105-server php php-fpm php-mysqli php-json php php-devel```

- PHPをスタートさせます。
```systemctl start php-fpm```
- 一応確認
```systemctl status php-fpm```

---
## EC2 2台目 DBの設定
- 1台目の時の同じ
```hostnamectl set-hostname db```
```dnf update -y```
- mariadbをインストールします。
```dnf install mariadb105-server```
- DBをスタートさせます。
```systemctl start mariadb```

- ```mysql_secure_installation```
  - 以下のことを設定するために行いますがrootのパスワード設定以外今回には関係ありません。rootのパスワード設定だけミスらないようにしましょう。
  - rootユーザーのパスワード設定
  - 匿名ユーザーの削除
  - 外部（ローカルホスト以外）からアクセス可能なrootユーザーの削除
  - testデータベースの削除
  - 「test_」から始まるデータベースへの接続権限の削除
  - 特権テーブルのリロード（更新内容の反映）
- rootユーザでログインします。
```mysql -u root -p```
↑にもあるようにまずmysqlコマンドで操作します。
-u root はログインするユーザを指定しています。
※-u yokoyama だったらyokoyamaってユーザでログイン。-uのuはuserのuです。
-p はパスワードを入力してログインするからパスワードを聞いてねってこと。コマンドを入力した後にパスワードが聞かれます。
- ここからDB内の操作。SQLを使って操作していきます。
  - ここではワードプレスが使う「データベースのユーザ」を作成します。rootユーザを使わせるわけにはいかないので、、。ワードプレス用のユーザを作ります。
    - CREATE USER →ユーザを作る時に使う。
    - wordpress-user　→ユーザの名前
    - @'APサーバーのプライベートIP'
      - なぜAPサーバのIPなのか。それはこのユーザを使うのがPHP側だからです。wordpressが動いているのはWEBAPサーバのほうです。このDBのサーバでは動いていません。WEBAPIサーバが何かしらのデータを使いたいときにこのユーザを使ってデータベースにアクセスし、必要な情報をとりに来ます。
    - IDENTIFIED BY wordpress-password　→このユーザのパスワードを設定します。
```MariaDB [(none)]> CREATE USER 'wordpress-user'@'APサーバーのプライベートIP' IDENTIFIED BY 'wordpress-password';```
  - データベースを作成します。
    - wordpress専用のデータベースです。
```MariaDB [(none)]> CREATE DATABASE `wordpress-database`;```
- データベースの権限を変更します。よく使っているコマンドchmodのイメージです。
  - GRANT ALL PRIVILEGES　→すべての権限を付与する。
  - ON `wordpress-database`　このデータベースに
  - "wordpress-user"@"APサーバーのプライベートIP";　→APサーバから接続するこのユーザに
```MariaDB [(none)]> GRANT ALL PRIVILEGES ON `wordpress-database`.* TO "wordpress-user"@"APサーバーのプライベートIP";```
- 設定を適応する。
MariaDB [(none)]> FLUSH PRIVILEGES;
---
## EC2 1台目　WebAP　での操作
- APからDBに接続できているか確認
  - APサーバ(Wordpress)からDBに接続しないといけないので、このコマンドで接続確認をします。
```mysql -u wordpress-user -h DBサーバのIP -p wordpress-database```
MariaDB [(wordpress-database)]>ってでてきたらOK


これでサーバの設定は完了なので、ブラウザで接続してみましょう。できない場合はセキュリティグループとか確認してみてください。

---
### まとめ

今回の構成でWordpressを構築するには以下のことが必要です。

- 1台目
  - Apacheのインストール
  - PHPのインストール
    - サービスの起動
  - mariadbのインストール
    - DBとしては使わないが、msqlコマンドを使いたい
  - wordpressのインストール
    - 設定ファイルを編集
      - ワードプレスが使うDBのユーザ名/パスワード/データベース名
      - DBサーバのIPアドレス(ここに接続してデータを持ってきたいから) 

- 2台目
  - mariadbのインストール
    - サービスの起動
  - DBの中にワードプレスが使う情報(ユーザ名パスワードデータベース)を作成し、データベースの権限を変更する

---
### 自習
+ 2台構成でやってみましたが、今度はこの手順書を見ながら1台構成をやってみてください。
+ 3台構成でやってみてください。
  + ApacheとPHPを別で入れる
    + その場合ApacheとPHPのサーバを繋げないといけません。
      + Apache側でPHPのサーバを許可する
      + PHP側でApacheのサーバを許可するという設定が必要そうですね。

