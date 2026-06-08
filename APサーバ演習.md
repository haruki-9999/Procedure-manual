# APサーバ演習

## 1.サーバをEC2で作成し、ログインする
その際にrootにスイッチする
```bash
sudo su -
```
EC-2に移動する
```bash
cd /home/ec2-user
```

## 2.JDKインストールする
下記のコマンドを入力して実行する
```bash
wget https://corretto.aws/downloads/latest/amazon-corretto-8-x64-linux-jdk.tar.gz
```

* 解凍する
```bash
tar zxf amazon-corretto-8-x64-linux-jdk.tar.gz
```
* 解凍したディレクトリに移動する
```bash
mv amazon-corretto-8.282.08.1-linux-x64 /opt
```
```bash
 ls /opt/※これで確認する
```

* javaへのパスが通るようにする


下記のviを開いて追記する
```bash
 vi /root/.bash_profile
 ```

追記するコマンド
 ```bash 
 export PATH=$PATH:$HOME/bin:/opt/amazon-corretto-8.482.08.1-linux-x64/bin
```

確認コマンド
矢印の下の結果に変更されていれば成功
```bash echo $PATH
/root/.local/bin:/root/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/root/bin:/opt/
```
↓
```bash
/root/.local/bin:/root/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/root/bin:/opt/amazon-corretto-8.482.08.1-linux-x64/bin
```
ソースコマンドを再読み込みする
```zsh
. /root/.bash_profile
```


eixtでログアウトしてもPATHが通っていることが確認する
できなかったら追記コマンドへ戻り、再度やる

## 3.Tomcatインストール

* Tomcatのユーザを追加
```bash
useradd -s /sbin/nologin tomcat
```
* Tomcatをダウンロードする
旧手順書はバージョンが古くURLも古いものになっている。
```bash
wget https://downloads.apache.org/tomcat/tomcat-9/v9.0.118/bin/apache-tomcat-9.0.118.tar.gz
```


ファイルがあることを確認する
```zsh
ls -l apache-tomcat-9.0.118.tar.gz
```
・解凍する
```zsh
tar zxf apache-tomcat-9.0.118.tar.gz
```
その後localに移動する
```zsh
cd /usr/local
```
移動したのちにhome/ec2-userにあるTomcatをカレントディレクトリに動かす
```zsh
 mv /home/ec2-user/apache-tomcat-9.0.118./
 ```
※本当にインストールできているのかの確認する
```zsh
 find / -name "apache-tomcat-9.0.117" 2>/dev/null
```

・所有者とグループをTomcatにするchown -R tomcat:tomcat apache-tomcat-9.0.117

変更時の確認コマンド
ls -l apache-tomcat-9.0.118
↓
コマンド結果は以下のように出てくればよい
drwxr-x---. 2 tomcat tomcat 16384 Apr 14 05:19 bin

* シンボリックを貼る
 ln -s apache-tomcat-9.0.118 tomcat

* setenvを作成する
下記のコマンドを打つ
vi /usr/local/tomcat/bin/setenv.sh
↓
viが開いたら下記の設定を記載する
---ここから--------------------------------------
#!/bin/sh
CATALINA_HOME=/usr/local/tomcat
JAVA_HOME=/opt/amazon-corretto-8.282.08.1-linux-x64
JAVA_OPTS="-Xms128m -Xmx512m"
---ここまで--------------------------------------


server.xml を設定する（ここで設定する箇所は自動デプロイを無効にする）
**※server.xmlとはTomatの動作設定を定義しているファイルのこと*

vi /usr/local/tomcat/conf/server.xml
ここで各2つの設定を無効にする
①unpackWARs
.warファイルを自動で解凍してフォルダに展開する
②autoDeploy
Tomcatが実行していても自動でアプリをデプロイ（反映）する
```zsh
< Host name="localhost" appBase="webapps"
      unpackWARs="false" autoDeploy="false">
< /Host>
```
* 自動スクリプトを作成する
vi /etc/systemd/system/tomcat.service
↓
vi内に下記を入力する
```zsh
[Unit]
Description=Apache Tomcat 9
After=network.target
[Service]
User=tomcat
Group=tomcat
Type=oneshot
PIDFile=/usr/local/tomcat/tomcat.pid
RemainAfterExit=yes
ExecStart=/usr/local/tomcat/bin/startup.sh
ExecStop=/usr/local/tomcat/bin/shutdown.sh
ExecReStart=/usr/local/tomcat/bin/shutdown.sh;/usr/local/tomcat/bin/startup.sh
[Install]
WantedBy=multi-user.target
```
権限を誰にどう与えるかコマンドで指定する
```zsh
chmod 755 /etc/systemd/system/tomcat.service
```
自動化してtomcatを起動する
```zsh
systemctl enable tomcat
systemctl start tomcat
```

## 4.Apacheのインストール

* yumでインストールする
dnf install httpd

* proxy 設定を追記
 vi httpd.conf

設定ファイルに下記を記入する
---ここから--------------------------------------
ProxyRequests Off
ProxyPass /knowledge http://127.0.0.1:8080/knowledge
ProxyPassReverse /knowledge http://127.0.0.1:8080/knowledge
---ここまで--------------------------------------

* 自動有効化と起動
systemctl enable httpd
systemctl start httpd


## 5.javaアプリの配備
* 配備場所のディレクトリ作成する
mkdir /usr/local/tomcat/webapps/knowledge

そして移動する
mkdir /usr/local/tomcat/webapps/knowledge

* アプリケーションのダウンロード
wget https://github.com/support-
project/knowledge/releases/download/v1.13.1/knowledge.war

* ダウンロードしたファイルを解答
jar xf　knowledge.war

* 不要なファイルを削除
rm knowledge.war

* アプリケーションの権限変更
cd ../
chown tomcat:tomcat knowledge

## 6.動作確認
http://35.161.86.110:8080/knowledge
↑
IPアドレスはグローバルIPアドレスにする

