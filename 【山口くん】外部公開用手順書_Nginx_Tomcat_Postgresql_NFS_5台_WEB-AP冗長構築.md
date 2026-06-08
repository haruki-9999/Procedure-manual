# Nginx，Tomcat，Postgresqlを用いたWeb/AP/DBサーバーの5台構築

------------------------------

## 1. ドキュメント情報

| 項目 | 内容 |
|------|------|
| 手順書名 | 外部公開用手順書_Nginx_Tomcat_Postgresql_NFS_5台_WEB-AP冗長構築 |
| 作成日 | 2026-05-31 |
| 最終更新日 | 2026-05-31 |
| 作成者 |  |
| バージョン | v1.0 |
| 対象環境 | AWS |

> **改訂履歴**
>
> | バージョン | 日付 | 変更内容 | 変更者 |
> |-----------|------|---------|--------|
> | v1.0 | 2026-05-31 | 初版作成 |  |

------------------------------

## 2. 目的・概要

### 2-1. 目的

> 本手順書では， それぞれ別のEC2でWebサーバーとして「Nginx」のサーバー2台，APサーバーとして「Tomcat」のサーバー2台，DBサーバーとして「Postgresql」のサーバー1台を用いて，情報共有OSSである「knowledge」を運用し，NFSシステムで「knowledge」のデータ共有を行うインフラ環境の構築手順について説明する．
> 構築後はブラウザで「`http://<WebサーバーのパブリックIP>/knowledge`」にアクセスし，NginxのサーバーとTomcatのサーバーで負荷分散されているログを確認できる状態を目指す．

### 2-2. 構成概要（アーキテクチャ）

![alt text](<Nginx_Tomcat_Postgresql_NFS_WEB-AP冗長.drawio.png>)

### 2-3. 完成イメージ（ゴール定義）

- [ ] ブラウザで「`http://<WebサーバーのパブリックIP>`」にアクセスし，「Welcome to Nginx!」と表示
- [ ] ブラウザで「`http://<WebサーバーのパブリックIP>/knowledge`」にアクセスし，Webページを閲覧できる
- [ ] ブラウザで「`http://<WebサーバーのパブリックIP>/knowledge`」にアクセスし，サインインして投稿した時に，Postgresqlと接続できている
- [ ] ブラウザで「`http://<WebサーバーのパブリックIP>/knowledge`」にアクセスし，Tomcatのサーバーに均等にログが確認できる
- [ ] 

------------------------------

## 3. 前提条件・準備


### 3-1. 環境要件

| 項目 | 要件 |
|------|------|
| OS | AWS（Amazon Linux 2023）,WSL（Ubuntu 24.04） |
| Webサーバー | Nginx |
| APサーバー | Tomcat |
| DBサーバー | Postgresql |

### 3-2. セキュリティグループ設定

### 3-2-1. Webサーバーのインバウンドルール

| タイプ | プロトコル | ポート範囲 | ソース | 説明 |
|-------|------------|----------|--------|------|
| SSH | TCP | 22 | マイIP | ローカルPCからSSHで接続 |
| HTTP | TCP | 80 | マイIP | ローカルPCのブラウザから接続 |

### 3-2-2. APサーバーのインバウンドルール

| タイプ | プロトコル | ポート範囲 | ソース | 説明 |
|-------|------------|----------|--------|------|
| SSH | TCP | 22 | マイIP | ローカルPCからSSHで接続 |
| カスタムTCP | TCP | 8080 | VPCのCIDR | VPC内のサーバーからのプロキシ転送許可 |

### 3-2-1. DBサーバーのインバウンドルール

| タイプ | プロトコル | ポート範囲 | ソース | 説明 |
|-------|------------|----------|--------|------|
| SSH | TCP | 22 | マイIP | ローカルPCからSSHで接続 |
| PostgreSQL | TCP | 5432 | APサーバーのプライベートIP | APサーバーからの接続許可 |
| NFS | TCP | 2049 | VPCのCIDR | VPC内のサーバーからのマウント許可 |

### 3-3. リンク一覧

| 項目名 | 目的 |
|-------|------|
| `https://corretto.aws/downloads/latest/amazon-corretto-8-x64-linux-jdk.tar.gz` | JDK（Amazon社のCorretto） |
| `https://downloads.apache.org/tomcat/tomcat-9/v9.0.118/bin/apache-tomcat-9.0.118.tar.gz` | Tomcatのサーバーソフト |
| `https://github.com/support-project/knowledge/releases/download/v1.13.1/knowledge.war` | knowledgeの本体 |
| `https://jdbc.postgresql.org/download/postgresql-42.6.2.jar` | JDBCドライバ |

------------------------------

## 4. 構築手順（詳細）

> **注意事項**
> - コマンド中の `<山カッコ>` は自分の環境の値に置き換えること

------------------------------

### Step 1 システム設定（全サーバー共通）

**目的：** システムの変更と更新を行う．

#### 操作手順

```bash
# ローカルPCからsshログイン
ssh -i <秘密鍵のファイルパス> ec2-user@<EC2のパブリックIP>

# rootにユーザーにスイッチ
sudo su -

# 最新パッケージの確認
dnf update -y

# 最新パッケージの更新
dnf upgrade -y

# システムの時間を日本時間に設定(これによりログを確認した時の時間が日本時間で表示されるようになる)
timedatectl set-timezone Asia/Tokyo

# ホスト名を変更
hostnamectl set-hostname <任意の名前>

# ec2-userにスイッチ
exit

# 再度rootユーザーにスイッチ
sudo su -
```

------------------------------

### Step 2 Nginxの設定（Webサーバー1）

**目的：** Nginx のシステム設定と転送設定の手順について説明する．

#### 操作手順

```bash
# Nginxのインストール
dnf install -y nginx

# Nginxのプロキシ設定ファイルの作成と記入
vi /etc/nginx/conf.d/proxy.conf

    #---以下を記入--------------------------------------------------------
    upstream knowledge_cluster {
        server <APサーバー1のプライベートIP>:8080;
    }
    server {
        listen       80;
        server_name  _;

        location /knowledge {
            proxy_pass http://knowledge_cluster/knowledge;

            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
    #---------------------------------------------------------------------


# Nginxの起動と自動起動設定
systemctl enable --now nginx

# Nginxの起動確認
systemctl status nginx | less

# Nginxの自動起動設定確認
systemctl is-enabled nginx
```

> **確認：** Nginxの起動と設定の確認
> 
> ブラウザで「`http://<Webサーバー1のパブリックIP>`」にアクセスし，「*Welcome to Nginx!*」と表示されれば成功

------------------------------

### Step 3 JDKの手動インストールと環境変数設定（APサーバー1）

**目的：** JDKを手動でインストールした時の設定手順と環境変数設定について説明する．
具体的に説明すると，`dnf install`でシステムが自動設定していることを手動で行うということである．

#### 操作手順

```bash
# 作業ディレクトリの移動
cd /home/ec2-user

# JDKのダウンロード
wget https://corretto.aws/downloads/latest/amazon-corretto-8-x64-linux-jdk.tar.gz

# ダウンロードできているかの確認
ll amazon-corretto-8-x64-linux-jdk.tar.gz

# JDKを解凍
tar zxf amazon-corretto-8-x64-linux-jdk.tar.gz

# JDKを移動
mv amazon-corretto-8.492.09.2-linux-x64 /opt

# シェル設定ファイルのバックアップ取得（原本保存）
cp /root/.bashrc{,.org}

# シェル設定ファイルの追記
vi /root/.bashrc

    #---設定ファイルの一番下に追記--------------------------------
    export JAVA_HOME=/opt/amazon-corretto-8.492.09.2-linux-x64
    export PATH=$JAVA_HOME/bin:$PATH
    #----------------------------------------------------------

# シェル設定ファイルの設定反映前のパス確認
echo $PATH

# シェル設定ファイルの設定を反映
source /root/.bashrc

# シェル設定ファイルの設定反映前のパス確認
echo $PATH

    #------------------------------
    追記したものが表示されていれば成功
    #------------------------------

```

------------------------------

### Step 4 Tomcatの設定（APサーバー1）

**目的：** Tomcatのダウンロードと手動設定

Amazon Linux 2023 の標準リポジトリに Tomcat のパッケージはないため，手動で設定を行う．

#### 操作手順

```bash
# Tomcatユーザーの作成
useradd -s /sbin/nologin tomcat

# TomcatユーザーのUIDを確認
id tomcat

# Tomcatをダウンロード
wget https://downloads.apache.org/tomcat/tomcat-9/v9.0.118/bin/apache-tomcat-9.0.118.tar.gz

# ダウンロードできているかの確認
ll apache-tomcat-9.0.118.tar.gz

# Tomcatの解凍
tar zxf apache-tomcat-9.0.118.tar.gz

# 作業ディレクトリの移動
cd /usr/local

# Tomcatの移動
mv /home/ec2-user/apache-tomcat-9.0.118 ./

# Tomcatの所有ユーザーと所有グループを変更
chown -R tomcat:tomcat apache-tomcat-9.0.118/

# シンボリックリンクの作成
ln -s /usr/local/apache-tomcat-9.0.118/ tomcat

# Tomcatの環境変数設定ファイルの作成と設定記入
vi /usr/local/tomcat/bin/setenv.sh

    #---以下を記入---------------------------------------------
    #!/bin/sh
    export CATALINA_HOME=/usr/local/tomcat
    export JAVA_HOME=/opt/amazon-corretto-8.492.09.2-linux-x64
    export JAVA_OPTS="-Xms128m -Xmx512m"
    export KNOWLEDGE_HOME=/var/lib/knowledge_data

    export PATH=$JAVA_HOME/bin:$PATH
    #---------------------------------------------------------

# Tomcatの環境変数設定ファイルの所有ユーザーと所有グループを変更
chown tomcat:tomcat /usr/local/tomcat/bin/setenv.sh

# Tomcatの環境変数設定ファイルの権限変更
chmod 750 /usr/local/tomcat/bin/setenv.sh

# knowledgeアプリケーションのデータ保存先を作成
mkdir /var/lib/knowledge_data

# knowledgeアプリケーションのデータ保存先の所有ユーザーと所有グループを変更
chown tomcat:tomcat /var/lib/knowledge_data

# Tomcatサーバーのネットワーク構成定義ファイルのバックアップ取得（原本保存）
cp /usr/local/tomcat/conf/server.xml{,.org}

# Tomcatサーバーのネットワーク構成定義ファイルの編集と追記
vi /usr/local/tomcat/conf/server.xml

    #---以下のように編集-----------------------
    #---変更前--------------------------------
    <Host name="localhost" appBase="webapps"
        unpackWARs="true" autoDeploy="true">
    #----------------------------------------
    <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
        prefix="localhost_access_log" suffix=".txt"
        pattern="%h %l %u %t &quot;%r&quot; %s %b" />
    #-----------------------------------------
    #---変更後--------------------------------
    <Host name="localhost" appBase="webapps"
        unpackWARs="false" autoDeploy="false">
    #-----------------------------------------
    <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
        prefix="localhost_access_log" suffix=".txt"
        pattern="%h %l %u %t &quot;%r&quot; %s %b"
        requestAttributesEnabled="true" />    
    #-----------------------------------------

    #---<Host name="localhost"タグの直下に以下を追記-------
    <Valve className="org.apache.catalina.valves.RemoteIpValve"
        internalProxies="<VPCのCIDR>"
        remoteIpHeader="X-Forwarded-For"
        proxiesHeader="X-Forwarded-By"
        protocolHeader="X-Forwarded-Proto" />
    #---------------------------------------------------

#---internalProxiesの記述方法-----------------------
172.31.0.0/16：172\.31\.\d{1,3}\.\d{1,3}
192.168.0.0/24：192\.168\.0\.\d{1,3}
#--------------------------------------------------

# TomcatをLinuxのサービスとして自動起動・管理するための設定ファイルの作成と設定記入
vi /etc/systemd/system/tomcat.service

    #---以下を記入------------------------------
    [Unit]
    Description=Apache Tomcat 9
    After=network.target

    [Service]
    User=tomcat
    Group=tomcat
    Type=forking

    ExecStart=/usr/local/tomcat/bin/startup.sh
    ExecStop=/usr/local/tomcat/bin/shutdown.sh

    [Install]
    WantedBy=multi-user.target
    #--------------------------------------------

# Tomcatサービスの権限変更
chmod 644 /etc/systemd/system/tomcat.service

# LinuxシステムにTomcatサービスの設定を再読み込み
systemctl daemon-reload

# Tomcatの起動と自動起動設定
systemctl enable --now tomcat

# Tomcatの起動確認
systemctl status tomcat | less

# Tomcatの自動起動設定確認
systemctl is-enabled tomcat
```

------------------------------

### Step 5 NFSサーバー側の公開ディレクトリ設定と同一UIDのユーザー作成（DBサーバー）

**目的：** NFSサーバーで公開するディレクトリの設定とAPサーバー1のTomcatユーザーと同じUIDのユーザーを作成する手順を説明する．

#### 操作手順

```bash
# NFSクライアントであるAPサーバー1のTomcatユーザーと同じUIDのTomcatユーザーを作成
useradd -u <確認したAPサーバー1でのTomcatユーザーのUID> -s /sbin/nologin tomcat

# TomcatユーザーのUIDがAPサーバー1と同一か確認
id tomcat

# NFSの公開ディレクトリを親ディレクトリも一緒に作成
mkdir -p /srv/nfs/knowledge_data

# 公開ディレクトリの所有ユーザーと所有グループをTomcatに変更
chown tomcat:tomcat /srv/nfs/knowledge_data/

# NFSの公開ディレクトリ設定ファイルのバックアップ取得（原本保存）
cp /etc/exports{,.org}

# NFSの公開ディレクトリ設定ファイルの追記
vi /etc/exports

    #---以下を追記--------------------------------------------------------------
    # NFSv4 疑似ルート（Pseudo Root）の定義
    /srv/nfs             <VPCのCIDR>(rw,sync,fsid=0,crossmnt,no_subtree_check)

    # 個別の公開ディレクトリ
    /srv/nfs/knowledge_data <VPCのCIDR>(rw,sync,no_subtree_check,root_squash)
    #--------------------------------------------------------------------------

# NFSの起動と自動起動設定
systemctl enable --now nfs-server

# NFSの起動確認
systemctl status nfs-server

# NFSの自動起動設定確認
systemctl is-enabled nfs-server
```

------------------------------

### Step 6 NFSクライアントのマウント設定（APサーバー1）

**目的：** NFSクライアント側のマウントポイントの作成とマウント設定

#### 操作手順

```bash
# マウント定義ファイルのバックアップ取得（原本保存）
cp /etc/fstab{,.org}

# マウント定義ファイルの追記
vi /etc/fstab

    #---以下を一番下に追記---------------------------------------------------------------------------------
    <NFSサーバーのプライベートIP>:<公開ディレクトリパス> <マウントポイントパス> nfs rw,nfsvers=4,soft,timeo=60,retrans=2,nofail,x-systemd.automount 0 0
    #----------------------------------------------------------------------------------------------------
    NFSのサーバー側で疑似ルートを設定しているため，上記の<公開ディレクトリパス>は疑似ルート以降のみの宣言でOK

# マウントポイントをマウント
mount <マウントポイントパス>

# マウントできているか確認
df
    #---以下のようになっていれば成功
    <NFSサーバーのプライベートIP>:<公開ディレクトリパス>    xxxxx   xxxxx   xxxxx   xx% <マウントポイントパス>
    #----------------------------
```

> **確認：** NFSサーバーとクライアントが共有できているか確認
>
> 今回の場合は、ログイン不可のTomcatユーザーが所有ユーザーとなっているので以下のコマンドでNFSクライアント側のマウントポイントにファイルを作成し、NFSサーバーの公開ディレクトリで確認する．
> 1. NFSクライアント側で「`sudo su -s /bin/bash -c "touch <マウントポイントパス>/test.txt" tomcat`」を実行
> 2. NFSサーバー側で「`ls <公開ディレクトリ>`」
> 作成したファイルが確認できれば成功

------------------------------

### Step 7 knowledgeアプリケーションの設定（APサーバー1）

**目的：** knowledgeアプリケーションの設定手順について説明する．

#### 操作手順

```bash
# knowledgeを配置するディレクトリの作成
mkdir /usr/local/tomcat/webapps/knowledge

# 作業ディレクトリの移動
cd /usr/local/tomcat/webapps/knowledge/

# knowledgeのダウンロード
wget https://github.com/support-project/knowledge/releases/download/v1.13.1/knowledge.war

# ダウンロードできているかの確認
ll knowledge.war

# knowledgeの解凍
jar xf knowledge.war

# 不要となったファイルの削除
rm knowledge.war

    #---下記のように表示されたら「yes」を入力してエンター
    rm: remove regular file 'knowledge.war'? yes
    #--------------------------------------------

# カレントディレクトリの親ディレクトリに移動
cd ..

# knowledgeディレクトリとその中のファイルとサブディレクトリの所有ユーザーと所有グループの変更
chown -R tomcat:tomcat knowledge/

# Tomcatの再起動
systemctl restart tomcat
```

> **確認：** NginxとTomcatの連携確認と今回のゴールの確認
>
> ブラウザで「`http://<Webサーバー1のパブリックIP>/knowledge`」にアクセスし，knowledgeのwebページが表示されれば成功

------------------------------

### Step 8 Postgresqlの設定（DBサーバー）

**目的：** Postgresqlの設定手順について説明する．

#### 操作手順

```bash
# Postgresqlをインストール
dnf install -y postgresql15-server

# Postgresqlの初期化
postgresql-setup --initdb

# Postgresqlの起動と自動起動設定
systemctl enable --now postgresql

# Postgresqlの状態確認
systemctl status postgresql | less

# Postgresqlの自動起動設定確認
systemctl is-enabled postgresql

# Postgresqlの特権ユーザー（postgres）でpostgresqlにログイン
sudo -u postgres psql

	# ユーザー作成とパスワード設定
	create user <ユーザー名> with password '<パスワード>';
	
	# データベース作成とその所有者の設定
	create database <データベース名> owner <ユーザー名>;

	# ログアウト
	\q

# Postgresqlの接続設定ファイルのバックアップ取得（原本保存）
cp /var/lib/pgsql/data/pg_hba.conf{,.org}

# Postgresqlの接続設定ファイルの編集
vi /var/lib/pgsql/data/pg_hba.conf

	#---一番下に追記--------------------------------------------------------------------------------
	host    <データベース名>       <ユーザー名>       <VPCのCIDR>            scram-sha-256
	#----------------------------------------------------------------------------------------------

# Postgresqlの待ち受けアドレス設定ファイルのバックアップ取得（原本保存）
cp /var/lib/pgsql/data/postgresql.conf{,.org}

# Postgresqlの待ち受けアドレス設定ファイルの編集
vi /var/lib/pgsql/data/postgresql.conf

    #---以下のように編集-----------------------------
	listen_addresses = '<自サーバーのプライベートIP>'
	#----------------------------------------------

# Postgresqlの再起動
systemctl restart postgresql.service
```

------------------------------

### Step 9 JDBCドライバの入れ替えと接続設定（APサーバー1）

**目的：** JDBCドライバの入れ替えと接続設定について説明する．

#### 操作手順

```bash
# knowledgeアプリケーションの古いJDBCドライバを削除
rm /usr/local/tomcat/webapps/knowledge/WEB-INF/lib/postgresql-42.1.4.jar

# 作業ディレクトリの移動
cd /usr/local/tomcat/lib/

# Postgresqlと互換性のあるJDBCドライバのダウンロード
wget https://jdbc.postgresql.org/download/postgresql-42.6.2.jar

# JDBCドライバの所有ユーザーと所有グループの変更
chown tomcat:tomcat postgresql-42.6.2.jar

# データベース接続先設定ファイルの作成と記入
sudo su -s /bin/bash -c "vi /var/lib/knowledge_data/custom_connection.xml" tomcat

    #---以下を記入
    <?xml version="1.0" encoding="UTF-8" standalone="yes"?>
    <connectionConfig>
        <name>custom</name>
        <driverClass>org.postgresql.Driver</driverClass>
        <URL>jdbc:postgresql://<DBサーバーのプライベートIP>:5432/<Postgresql設定時のデータベース名></URL>
        <user><Postgresql設定時のユーザー名></user>
        <password><Postgresql設定時のパスワード></password>
        <schema>public</schema>
        <maxConn>100</maxConn>
        <autocommit>false</autocommit>
    </connectionConfig>

# 設定反映のためTomcatを再起動
systemctl restart tomcat
```

> **確認：** TomcatとPostgresqlの接続確認と今回のゴールの確認
>
> **方法1：** knowledgeアプリケーションのGUI画面から確認
> 
> 1. ブラウザで「`http://<Webサーバー1のパブリックIP>/knowledge`」にアクセス
> 2. サインイン画面にリダイレクトされるので管理者でログイン
>   ユーザー名：admin パスワード：admin123
> 3. 画面右上のメニューからシステム設定を選択
> 4. データ管理のデータベースの接続先変更を選択
>   `custom_connection.xml`で設定したものが反映されていれば成功

> **方法2：** DBサーバーで作成したデータベースの中にテーブルが作成されているか確認
>
> 1. DBサーバーでpostgresqlにPostgresユーザーでログイン `sudo -u postgres psql`
> 2. 作成したユーザーにスイッチ `\c <作成したユーザー名>`
> 3. テーブル一覧を確認 `\dt`
>   テーブル一覧が表示されれば成功

------------------------------

### Step 10 APサーバーの追加

**目的：** 冗長化のためのAPサーバー追加構築の説明をする．

#### 操作手順

1. AWSコンソールですでに構築済みのAPサーバー1を停止
2. APサーバー1が「停止済み」になったことを確認
3. APサーバー1を選択 => アクション => イメージとテンプレート => イメージを作成

    | 設定項目 | 内容 |
    |----------|------|
    | イメージ名 | 任意の名前 |
    | イメージの説明 | 任意 |
    | インスタンスを再起動 | インスタンスが停止済みの状態になっていることを確認してからamiの作成に入っているのでどちらでもよい |
    | インスタンスボリューム | そのまま |
    | タグ | AMIとストレージに対してそれぞれ同じ名前を付けるか個別の名前を付けるかなのでどちらでもよい |

4. イメージを作成
5. AMIを使ってEC2を起動

**起動したAPサーバー2**
```bash
# ローカルPCからsshログイン
ssh -i <秘密鍵のファイルパス> ec2-user@<EC2のパブリックIP>

# rootにユーザーにスイッチ
sudo su -

# ホスト名を変更
hostnamectl set-hostname <任意の名前>

# ec2-userにスイッチ
exit

# 再度rootユーザーにスイッチ
sudo su -
```

**構築済みのWEBサーバー1**
```bash
# Nginxのプロキシ設定ファイルに追記
vi /etc/nginx/conf.d/proxy.conf

    #---upstreamセクションに以下を追記--------
    server <APサーバー2のプライベートIP>:8080;
    #---------------------------------------

# 設定反映のためNginxを再起動
systemctl restart nginx
```

> **確認：** TomcatとPostgresqlの接続確認と今回のゴールの確認
>
> 1. 2台のAPサーバーで「`tail -f /usr/local/tomcat/logs/localhost_access_log.YYYY-MM-DD.txt`」を実行し、リアルタイムでログを監視
> 2. ブラウザで「`http://<Webサーバー1のパブリックIP>/knowledge`」にアクセス
> 2台のAPサーバーに均等にログが記録されていれば成功

------------------------------

### Step 11 WEBサーバーの追加

**目的：** 冗長化のためのWEBサーバー追加構築の説明をする．

#### 操作手順

1. AWSコンソールですでに構築済みのWEBサーバー1を停止
2. WEBサーバー1が「停止済み」になったことを確認
3. WEBサーバー1を選択 => アクション => イメージとテンプレート => イメージを作成

    | 設定項目 | 内容 |
    |----------|------|
    | イメージ名 | 任意の名前 |
    | イメージの説明 | 任意 |
    | インスタンスを再起動 | インスタンスが停止済みの状態になっていることを確認してからamiの作成に入っているのでどちらでもよい |
    | インスタンスボリューム | そのまま |
    | タグ | AMIとストレージに対してそれぞれ同じ名前を付けるか個別の名前を付けるかなのでどちらでもよい |

4. イメージを作成
5. AMIを使ってEC2を起動

**起動したAPサーバー2**
```bash
# ローカルPCからsshログイン
ssh -i <秘密鍵のファイルパス> ec2-user@<EC2のパブリックIP>

# rootにユーザーにスイッチ
sudo su -

# ホスト名を変更
hostnamectl set-hostname <任意の名前>

# ec2-userにスイッチ
exit

# 再度rootユーザーにスイッチ
sudo su -
```

> **確認：** TomcatとPostgresqlの接続確認と今回のゴールの確認
>
> 1. 2台のAPサーバーで「`tail -f /usr/local/tomcat/logs/localhost_access_log.YYYY-MM-DD.txt`」を実行し、リアルタイムでログを監視
> 2. ブラウザで「`http://<Webサーバー1のパブリックIP>/knowledge`」にアクセス
> 2台のAPサーバーに均等にログが記録されていることを確認
> 3. ブラウザで「`http://<Webサーバー2のパブリックIP>/knowledge`」にアクセス
> 2台のAPサーバーに均等にログが記録されていることを確認
> 4. 2,3の確認が両方できていれば成功

------------------------------

## 付録

### A. コマンド解説

------------------------------

### B. 設定ファイル解説

------------------------------

### C. 用語解説

------------------------------

### D. 補足解説
