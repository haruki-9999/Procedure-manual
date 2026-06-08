# Web（nginx）構築〈リバースプロキシ不採用）

## nginxインストール

OSのアップデートする
```zsh
dnf update -y
```
OpenJDK 17インストールをインストールする
```zsh
sudo dnf install -y java-17-amazon-corretto java-17-amazon-corretto-devel
```
nginxインストールする
```zsh
sudo dnf install -y nginx
```

nginxを自動起動する
```zsh
sudo systemctl start nginx
sudo systemctl enable --now nginx
```
スタッツを確認する
```zsh 
systemctl status nginx
```
動作確認
```zsh
http://サーバIP
```

## Tomcatインストール

Tomcatユーザーを作成する、権限を変更する
```zsh
sudo useradd -r -m -U -d /opt/tomcat -s /bin/false tomcat
sudo chown -R tomcat:tomcat /opt/tomcat
```

Tomcatダウンロード
```zsh
cd /tmp

curl -O https://downloads.apache.org/tomcat/tomcat-10/v10.1.55/bin/apache-tomcat-10.1.55.tar.gz
```

ファイルを解凍する
```zsh
tar xzf apache-tomcat-10.1.55.tar.gz
```
ファイルの配置を変更する+実行権限を与える
```zsh 
sudo mv apache-tomcat-10.1.55 /opt/tomcat
sudo chmod +x /opt/tomcat/apache-tomcat-10.1.55/bin/*.sh
```
tomcatを起動する
```zsh
/opt/tomcat/apache-tomcat-10.1.55/bin/startup.sh
```
ポート確認する
```zsh
ss -tulnp | grep 8080
```
Tomcatの初期ページを出す
```zsh
http://サーバIP:8080
```

## nginxにTomecatを連携設定をする

連携設定を行う
```zsh
sudo vi /etc/nginx/conf.d/tomcat.conf
```
設定ファイルに下記を追加する（初期状態なのでファイル内は何もない）
```zsh
server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://127.0.0.1:8080;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

設定を確認する
```zsh
sudo nginx -t
```
成功していたら下記コマンド
```zsh
sudo systemctl restart nginx
```

ブラウザで確認する
```zsh
http://サーバIP
```

## Tomcatをsystemd化する

Tomcatをsystemd化
```zsh
systemctl start tomcat
systemctl enable tomcat
```

Java WEBアプリを配置する
```zsh
sudo mkdir -p /opt/tomcat/apache-tomcat-10.1.55/webapps/test
```

viに下記を書き込む
```zsh
<h1>Hello Tomcat</h1>
```

ブラウザで確認する
```zsh
http://ec2-userのIP/test
```
systendサービスファイル作成する
```zsh
sudo vi /etc/systemd/system/tomcat.service
```
下記はファイルに中身
```zsh
[Unit]
Description=Apache Tomcat
After=network.target

[Service]
Type=forking

User=tomcat
Group=tomcat

Environment=JAVA_HOME=/usr/lib/jvm/java-17-openjdk
Environment=CATALINA_HOME=/opt/tomcat/apache-tomcat-10.1.55
Environment=CATALINA_BASE=/opt/tomcat/apache-tomcat-10.1.55

ExecStart=/opt/tomcat/apache-tomcat-10.1.55/bin/startup.sh
ExecStop=/opt/tomcat/apache-tomcat-10.1.55/bin/shutdown.sh

Restart=always

[Install]
WantedBy=multi-user.target
```
systemdの反映、起動、自動起動設定
```zsh
sudo systemctl daemon-reload
sudo systemctl start tomcat
sudo systemctl enable tomcat
```
状態を確認する
```zsh
sudo systemctl status tomcat
```
問題が起きた時のログ確認方法
```zsh
journalctl -u tomcat -f
```
## PostgreSQL

PostgreSQLをインストールする
```zsh
sudo dnf install -y postgresql15-server
```
起動する
```zsh
sudo systemctl start postgresql
```
初期化されていなかったら下記コマンド後にスタートの工程を再度行う
```zsh
postgresql-setup --initdb
```

PostgreSQLに接続する
```zsh
sudo -u postgres psql
```
下記を入力したら終了
```zsh
SELECT version();
```
これで終了（下記）
```zsh
\q
```
PostgreSQLにロクインする
```zsh
sudo -u postgres psql
```
DBユーザーを作成する
```zsh
CREATE USER appuser WITH PASSWORD 'StrongPassword123!';
```
DBを作成する
```zsh
CREATE DATABASE appdb OWNER appuser;
```
権限確認する
```zsh
\l
```
終了
```zsh
\q
```
外部接続許可する
```zsh
vi /var/lib/pgsql/data/postgresql.conf
```
↓ファイル内に下記を探す
```zsh
#listen_addresses = 'localhost'
```
下記を追加する
```zsh
listen_addresses = '*'
```

認証設定する
```zsh
vi /var/lib/pgsql/data/pg_hba.conf
```
↓一番下に下記を追加する
```zsh
host    all             all             0.0.0.0/0               md5
```
↑
ローカルの認証も全てmd5に変換する必要がある

PostgreSQLを再起動する
```zsh
systemctl restart postgresql
```
ポート確認する
```zsh
ss -tulpn | grep 5432
```

AWSでセキュリティグループを解放する
- ポート番号5432を開ける
- インバウンドルールはマイIPに設定する

Tomcat JDBC ドライバ配置
※下記は配置先なのでコマンドではない
```zsh
/opt/tomcat/apache-tomcat-10.1.55/lib/
```
Javaアプリ接続設定
```zsh
jdbc:postgresql://localhost:5432/appdb
```
上記のコマンドでエラーが出たら下記コマンドでドライバーをインストールする
```zsh
curl -LO https://jdbc.postgresql.org/download/postgresql-42.7.11.jar
```
Tomcatのlibに配置
```zsh
sudo mv postgresql-42.7.3.jar /opt/tomcat/apache-tomcat-10.1.55/lib/
```
権限の確認
```zsh
ls -l /opt/tomcat/apache-tomcat-10.1.55/lib | grep postgres
```
Tomcat再起動を行う
```zsh
/opt/tomcat/apache-tomcat-10.1.55/bin/shutdown.sh
/opt/tomcat/apache-tomcat-10.1.55/bin/startup.sh
```

作成したDBにログインしてみる
もしエラーが出たらAIにエラーを送信して聞く
```zsh
psql -h localhost -U haruki -d harukidb -W
```
DB内でテーブルを作成する
```zsh
CREATE TABLE test_users (
    id SERIAL PRIMARY KEY,
    name TEXT
);
```
データを投入する
```zsh
INSERT INTO test_users (name) VALUES ('test1');
```

テーブルを確認する
```zsh
SELECT * FROM test_users;
```
Javaファイルを作る（Linux上）
```zsh
vi harukiDB.java
```
↓下記を書き込む
```zsh
import java.sql.*;

public class harukiDB {
    public static void main(String[] args) throws Exception {

        Class.forName("org.postgresql.Driver");

        Connection conn = DriverManager.getConnection(
            "jdbc:postgresql://localhost:5432/harukidb",
            "haruki",
            "20240515"
        );

        Statement stmt = conn.createStatement();

        ResultSet rs = stmt.executeQuery("SELECT * FROM test_users");

        while (rs.next()) {
            System.out.println(rs.getString("name"));
        }

        rs.close();
        stmt.close();
        conn.close();
    }
}
```
コンパイル
```zsh
javac -cp postgresql-42.7.11.jar TestDB.java
```
実行する
```zsh
java -cp .:postgresql-42.7.3.jar TestDB
```
期待結果として下記が出たら正解
```zsh
test1
```
※ここまでできたら連携成功


## WEBアプリを作成
appディレクトリ作成
```zsh
mkdir -p /opt/tomcat/apache-tomcat-10.1.55/webapps/app/WEB-INF/classes
mkdir -p /opt/tomcat/apache-tomcat-10.1.55/webapps/app/WEB-INF
```
web.xml作成する
```zsh
vi /opt/tomcat/apache-tomcat-10.1.55/webapps/app/WEB-INF/web.xml
```
↓中身にそのまま追加する（必要事項は変える）
```zsh
<web-app xmlns="https://jakarta.ee/xml/ns/jakartaee"
         version="5.0">

    <servlet>
        <servlet-name>test</servlet-name>
        <servlet-class>TestServlet</servlet-class>
    </servlet>

    <servlet-mapping>
        <servlet-name>test</servlet-name>
        <url-pattern>/test</url-pattern>
    </servlet-mapping>

</web-app>
```
コンパイルする
```zsh
javac -cp ".:/opt/tomcat/apache-tomcat-10.1.55/lib/*" harukiDB.java
```
↑
これが成功するとharukiDB.classができる

class配置（Tomcatに渡す）
```zsh
cp harukiDB.class /opt/tomcat/apache-tomcat-10.1.55/webapps/app/WEB-INF/classes/
```

web.xml確認
```zsh
cat /opt/tomcat/apache-tomcat-10.1.55/webapps/app/WEB-INF/web.xml
```
↓中身のように編集する
```zsh
<web-app xmlns="https://jakarta.ee/xml/ns/jakartaee" version="5.0">

  <servlet>
    <servlet-name>harukiDB</servlet-name>
    <servlet-class>harukiDB</servlet-class>
  </servlet>

  <servlet-mapping>
    <servlet-name>harukiDB</servlet-name>
    <url-pattern>/db</url-pattern>
  </servlet-mapping>

</web-app>
```

Tomcat再起動
```zsh
systemctl restart tomcat
```
ブラウザで確認する
```zsh
http://<サーバIP>:8080/app/db
```
下記のように見れたら成功
```zsh
2026-05-28 16:34:10.198536+00
```



