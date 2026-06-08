**nginx ModSecurity 移行手順書**

nginx 1.16.1 → nginx 1.30.0

ModSecurity \+ nginx.conf 設定移行
Web通信の中身を見て攻撃っぽい内容を検知・ブロックする
**1\. 概要**

本手順書は、nginx 1.16.1 環境で稼働中の ModSecurity WAF を nginx 1.30.0 新環境へ移行するための作業手順をまとめたものです。

旧環境と新環境の構成の違いを踏まえ、ModSecurity モジュールのインストール・設定ファイルの移行・動作確認までを網羅します。

**1.1 環境概要**

| 項目 | 旧環境 | 新環境 |
| :---- | :---- | :---- |
| nginx バージョン | nginx 1.16.1 | nginx 1.30.0 |
| 設定ファイルパス | /usr/local/nginx-1.16.1/conf/nginx.conf | /etc/nginx/nginx.conf/etc/nginx/conf.d/proxy.conf |
| ModSecurity ルール | /usr/local/nginx/conf/modsec\_includes.conf | /usr/local/nginx/conf/modsec\_includes.conf（移行） |

**2\. 事前確認**

**2.1 旧環境の ModSecurity モジュール確認**

旧環境で使用中の ModSecurity バージョンと動的モジュール (.so) を確認します。

\# nginx バージョン確認  
/usr/local/nginx-1.16.1/sbin/nginx \-V 2\>&1 | grep \-i modsecurity  
   
\# ロード済みモジュール確認  
ls /usr/local/nginx-1.16.1/modules/  
ls /usr/local/nginx/modules/   \# 旧 nginx.conf の load\_module パスに合わせて確認

**2.2 旧環境の ModSecurity 設定ファイル一覧**

以下のファイルを新環境へ移行します。

* /usr/local/nginx/conf/modsec\_includes.conf  （インクルード定義）  
* /usr/local/nginx/conf/modsecurity.conf  （本体設定、存在する場合）  
* /usr/local/nginx/conf/crs/  または /usr/share/modsecurity-crs/  （OWASP CRS ルールセット）

**⚠ 注意:** 移行前にこれらのファイルをすべてバックアップしてください。

**2.3 新環境の nginx モジュール確認**

新環境（nginx 1.30.0）で ModSecurity 動的モジュールが利用可能か確認します。

\# nginx バージョン確認  
nginx \-V  
   
\# モジュールディレクトリ確認  
ls /usr/share/nginx/modules/ | grep modsecurity  
ls /etc/nginx/modules/ 2\>/dev/null

**★ 重要:** nginx 1.30.0 用にコンパイルされた ModSecurity モジュール（ngx\_http\_modsecurity\_module.so）が必要です。旧環境のモジュールはそのまま流用できません。

**3\. ModSecurity モジュールの準備**

**3.1 パッケージからのインストール（推奨）**

OS のパッケージマネージャーから nginx 1.30.0 対応の ModSecurity モジュールが提供されている場合は、パッケージインストールが最も簡単です。

**RHEL / AlmaLinux / Rocky Linux 系**

\# EPEL \+ nginx 公式リポジトリを有効化（未設定の場合）  
dnf install \-y epel-release  
dnf install \-y nginx-mod-modsecurity  
   
\# モジュールファイルの場所確認  
rpm \-ql nginx-mod-modsecurity | grep .so

**Ubuntu / Debian 系**

apt-get install \-y libnginx-mod-http-modsecurity  
   
\# モジュールファイルの場所確認  
dpkg \-L libnginx-mod-http-modsecurity | grep .so

**3.2 ソースビルドによるインストール（パッケージが存在しない場合）**

パッケージが提供されていない場合、ModSecurity v3 を nginx 1.30.0 のソースと合わせてビルドします。

\# 依存パッケージインストール（RHEL 系）  
dnf install \-y gcc gcc-c++ make git libxml2-devel pcre-devel \\  
    curl-devel yajl-devel lmdb-devel ssdeep-devel lua-devel \\  
    GeoIP-devel  
   
\# ModSecurity v3 のビルド  
git clone \--depth 1 \-b v3/master https://github.com/SpiderLabs/ModSecurity  
cd ModSecurity  
git submodule init && git submodule update  
./build.sh  
./configure  
make \-j$(nproc)  
make install  
cd ..  
   
\# nginx 1.30.0 ソース取得  
curl \-O http://nginx.org/download/nginx-1.30.0.tar.gz  
tar xzf nginx-1.30.0.tar.gz  
   
\# ModSecurity-nginx コネクタのクローン  
git clone \--depth 1 https://github.com/SpiderLabs/ModSecurity-nginx  
   
\# 動的モジュールとしてビルド  
cd nginx-1.30.0  
./configure \--with-compat \--add-dynamic-module=../ModSecurity-nginx  
make modules  
   
\# モジュールをコピー  
cp objs/ngx\_http\_modsecurity\_module.so /etc/nginx/modules/  
cd ..

**⚠ 注意:** ビルド時の nginx configure オプションは、新環境の nginx \-V で確認できる既存オプションに \--with-compat と \--add-dynamic-module を追加してください。

**4\. 設定ファイルの移行**

**4.1 ModSecurity 設定ファイルのコピー**

旧環境の ModSecurity 設定ファイルを新環境にコピーします。

\# 旧環境（コピー元）から設定ファイルを転送  
scp \-r \<旧サーバIP\>:/usr/local/nginx/conf/modsec\_includes.conf \\  
    /usr/local/nginx/conf/  
   
scp \-r \<旧サーバIP\>:/usr/local/nginx/conf/modsecurity.conf \\  
    /usr/local/nginx/conf/  
   
\# OWASP CRS ルールセットのコピー（例）  
scp \-r \<旧サーバIP\>:/usr/local/nginx/conf/crs/ \\  
    /usr/local/nginx/conf/  
   
\# パーミッション設定  
chown \-R root:root /usr/local/nginx/conf/  
chmod 644 /usr/local/nginx/conf/\*.conf

**4.2 新環境 nginx.conf の修正**

新環境の /etc/nginx/nginx.conf に ModSecurity モジュールのロード行を追加します。現在コメントアウトされている部分と、モジュールロードの設定を反映します。

**変更箇所: モジュールロードの追加**

nginx.conf の先頭（user 行より前）または include /usr/share/nginx/modules/\*.conf; の行の前後に追記します。

\# /etc/nginx/nginx.conf への追記  
\# ファイル冒頭（既存の include /usr/share/nginx/modules/\*.conf; の直後）に追加  
   
load\_module modules/ngx\_http\_modsecurity\_module.so;

**📌 補足:** /usr/share/nginx/modules/\*.conf 経由で自動ロードされる場合は、上記の load\_module 行は不要です。ls /usr/share/nginx/modules/ で ngx\_http\_modsecurity\_module.so の存在を確認してください。

**変更箇所: http ブロック内のチューニング設定**

旧環境 nginx.conf にあった以下のパフォーマンス・接続設定を新環境 nginx.conf の http ブロックに追加します（既存設定と重複する項目は上書きまたはマージしてください）。

http {  
    \# \--- 旧環境から移行する設定 \---  
    tcp\_nodelay             on;  
    client\_max\_body\_size    1000M;  
    server\_tokens           off;  
    server\_names\_hash\_bucket\_size  512;  
    client\_header\_buffer\_size      64k;  
    large\_client\_header\_buffers    4 64k;  
    gzip                    on;  
    gzip\_http\_version       1.0;  
    gzip\_disable            "msie6";  
    gzip\_proxied            any;  
    gzip\_min\_length         1024;  
    gzip\_comp\_level         9;  
    keepalive\_timeout       300;  
    keepalive\_requests      300;  
    proxy\_http\_version      1.1;  
    proxy\_set\_header        Connection "";  
    proxy\_ignore\_client\_abort  on;  
    proxy\_buffering         on;  
    proxy\_buffer\_size       8k;  
    proxy\_buffers           2048 8k;  
    send\_timeout            300;  
    \# \--- ここまで \---  
    ...  
}

**4.3 proxy.conf の修正**

新環境の /etc/nginx/conf.d/proxy.conf には既に modsecurity on; と modsecurity\_rules\_file が設定されています。モジュールのパスを確認し、必要に応じて修正します。

**現在の proxy.conf（抜粋）**

server {  
    listen       80;  
    server\_name  \_;  
   
    modsecurity on;  
    modsecurity\_rules\_file /usr/local/nginx/conf/modsec\_includes.conf;  
    ...  
}

**HTTPS（443）サーバブロックの追加**

旧環境には SSL/TLS の 443 番ポートの設定（hr-dash.tech 用）がありました。新環境でも HTTPS が必要な場合は proxy.conf に以下を追加します。

server {  
    listen       443 ssl http2;  
    server\_name  hr-dash.tech;  
    port\_in\_redirect  off;  
   
    ssl\_certificate     /etc/letsencrypt/live/hr-dash.tech/cert.pem;  
    ssl\_certificate\_key /etc/letsencrypt/live/hr-dash.tech/privkey.pem;  
    ssl\_session\_cache   shared:SSL:1m;  
    ssl\_session\_timeout 10m;  
    ssl\_ciphers         HIGH:\!aNULL:\!MD5;  
    ssl\_prefer\_server\_ciphers on;  
   
    proxy\_set\_header Host             $host;  
    proxy\_set\_header X-Real-IP        $remote\_addr;  
    proxy\_set\_header X-Forwarded-For  $proxy\_add\_x\_forwarded\_for;  
   
    modsecurity on;  
    modsecurity\_rules\_file /usr/local/nginx/conf/modsec\_includes.conf;  
   
    location / {  
        return 301 /hr-dash;  
    }  
   
    location /hr-dash {  
        proxy\_pass http://knowledge\_cluster/hr-dash;  
        proxy\_set\_header Host             $host;  
        proxy\_set\_header X-Real-IP        $remote\_addr;  
        proxy\_set\_header X-Forwarded-For  $proxy\_add\_x\_forwarded\_for;  
        proxy\_set\_header X-Forwarded-Proto $scheme;  
    }  
}

**4.4 modsec\_includes.conf のパス確認**

コピーした modsec\_includes.conf の中で参照しているルールファイルのパスが新環境でも有効か確認・修正します。

\# modsec\_includes.conf の内容確認  
cat /usr/local/nginx/conf/modsec\_includes.conf  
   
\# 参照先ファイルが存在するか確認（例）  
ls /usr/local/nginx/conf/modsecurity.conf  
ls /usr/share/modsecurity-crs/

**📌 補足:** OWASP CRS を使用している場合、新環境で改めて最新版をインストールすることを推奨します（https://github.com/coreruleset/coreruleset）。

**5\. 設定反映・動作確認**

**5.1 nginx 設定ファイルの文法チェック**

nginx \-t  
   
\# 期待される出力  
\# nginx: the configuration file /etc/nginx/nginx.conf syntax is ok  
\# nginx: configuration file /etc/nginx/nginx.conf test is successful

**⚠ 注意:** 文法エラーが出た場合は、エラーメッセージを確認してファイルを修正してください。ModSecurity モジュールがロードされていないとmodsecurity ディレクティブでエラーになります。

**5.2 nginx の再起動**

\# 設定リロード（無停止）  
systemctl reload nginx  
   
\# または再起動  
systemctl restart nginx  
   
\# 起動状態確認  
systemctl status nginx

**5.3 ModSecurity 動作確認**

ModSecurity が正常に機能しているか確認します。

\# ModSecurity が検知モード（DetectionOnly）または防御モード（On）か確認  
grep \-i "SecRuleEngine" /usr/local/nginx/conf/modsecurity.conf  
   
\# テスト: SQL インジェクション攻撃パターンを送信して 403 が返るか確認  
curl \-I "http://localhost/?id=1 UNION SELECT 1,2,3--"  
   
\# エラーログ確認  
tail \-f /var/log/nginx/error.log  
   
\# ModSecurity 監査ログ確認  
tail \-f /var/log/modsec\_audit.log  \# パスは modsecurity.conf の SecAuditLog による

**5.4 アプリケーション動作確認**

正常なリクエストがブロックされていないことを確認します。

* ブラウザで http(s)://hr-dash.tech/hr-dash にアクセスし、アプリケーションが正常表示されること  
* ログイン・画面遷移・ファイルアップロードなど主要機能が動作すること  
* /var/log/nginx/error.log に ModSecurity 由来の意図しない 403 エラーがないこと

**6\. トラブルシューティング**

| 事象 | 対処方法 |
| :---- | :---- |
| nginx \-t で "unknown directive modsecurity" エラー | load\_module が正しく記述されているか、.so ファイルが存在するパスか確認する。nginx 1.30.0 対応の .so を使用していること。 |
| nginx \-t で "cannot load" エラー（モジュールロード失敗） | ldd /etc/nginx/modules/ngx\_http\_modsecurity\_module.so で依存ライブラリの不足を確認し、必要なパッケージをインストールする。 |
| 正常なリクエストが 403 で弾かれる | modsecurity.conf の SecRuleEngine を DetectionOnly に変更してエラーログを確認。誤検知ルールを特定して SecRuleRemoveById で除外する。 |
| ModSecurity 監査ログが生成されない | modsecurity.conf の SecAuditEngine と SecAuditLog の設定を確認。ログディレクトリのパーミッションが nginx ユーザーで書き込み可能か確認。 |
| nginx が起動しない（設定は問題なし） | SELinux や AppArmor が有効な環境では、モジュール・設定ファイルのラベルや許可ポリシーを確認する。 |

**7\. 移行後の設定ファイル例**

**7.1 /etc/nginx/nginx.conf（変更箇所のみ抜粋）**

\# 既存の include /usr/share/nginx/modules/\*.conf; の後に追加  
load\_module modules/ngx\_http\_modsecurity\_module.so;  
   
\# ※ /usr/share/nginx/modules/ 経由で自動ロードされる場合は不要  
   
http {  
    \# \--- 旧環境から移行したチューニング設定 \---  
    tcp\_nodelay                    on;  
    client\_max\_body\_size           1000M;  
    server\_tokens                  off;  
    server\_names\_hash\_bucket\_size  512;  
    client\_header\_buffer\_size      64k;  
    large\_client\_header\_buffers    4 64k;  
    gzip on;  
    gzip\_http\_version  1.0;  
    gzip\_disable       "msie6";  
    gzip\_proxied       any;  
    gzip\_min\_length    1024;  
    gzip\_comp\_level    9;  
    keepalive\_timeout  300;  
    keepalive\_requests 300;  
    proxy\_http\_version 1.1;  
    proxy\_set\_header   Connection "";  
    proxy\_ignore\_client\_abort on;  
    proxy\_buffering    on;  
    proxy\_buffer\_size  8k;  
    proxy\_buffers      2048 8k;  
    send\_timeout       300;  
    \# \---  
   
    include /etc/nginx/conf.d/\*.conf;  
}

**7.2 /etc/nginx/conf.d/proxy.conf（完成形）**

upstream knowledge\_cluster {  
    server 10.0.171.244:8080;  
}  
   
server {  
    listen      80;  
    server\_name \_;  
   
    modsecurity on;  
    modsecurity\_rules\_file /usr/local/nginx/conf/modsec\_includes.conf;  
   
    location / {  
        return 301 /hr-dash;  
    }  
   
    location /hr-dash {  
        proxy\_pass http://knowledge\_cluster/hr-dash;  
        proxy\_set\_header Host             $host;  
        proxy\_set\_header X-Real-IP        $remote\_addr;  
        proxy\_set\_header X-Forwarded-For  $proxy\_add\_x\_forwarded\_for;  
        proxy\_set\_header X-Forwarded-Proto $scheme;  
    }  
}  
   
server {  
    listen      443 ssl http2;  
    server\_name hr-dash.tech;  
    port\_in\_redirect off;  
   
    ssl\_certificate     /etc/letsencrypt/live/hr-dash.tech/cert.pem;  
    ssl\_certificate\_key /etc/letsencrypt/live/hr-dash.tech/privkey.pem;  
    ssl\_session\_cache   shared:SSL:1m;  
    ssl\_session\_timeout 10m;  
    ssl\_ciphers         HIGH:\!aNULL:\!MD5;  
    ssl\_prefer\_server\_ciphers on;  
   
    proxy\_set\_header Host             $host;  
    proxy\_set\_header X-Real-IP        $remote\_addr;  
    proxy\_set\_header X-Forwarded-For  $proxy\_add\_x\_forwarded\_for;  
   
    modsecurity on;  
    modsecurity\_rules\_file /usr/local/nginx/conf/modsec\_includes.conf;  
   
    location / {  
        return 301 /hr-dash;  
    }  
   
    location /hr-dash {  
        proxy\_pass http://knowledge\_cluster/hr-dash;  
        proxy\_set\_header Host             $host;  
        proxy\_set\_header X-Real-IP        $remote\_addr;  
        proxy\_set\_header X-Forwarded-For  $proxy\_add\_x\_forwarded\_for;  
        proxy\_set\_header X-Forwarded-Proto $scheme;  
    }  
}

**8\. 作業チェックリスト**

☐  旧環境の ModSecurity バージョン・モジュールを確認した

☐  旧環境の ModSecurity 設定ファイル（modsec\_includes.conf、modsecurity.conf、CRS）をバックアップした

☐  新環境に nginx 1.30.0 対応の ngx\_http\_modsecurity\_module.so を準備した

☐  /etc/nginx/nginx.conf に load\_module を追記した（自動ロードの場合は不要）

☐  http ブロックに旧環境のチューニング設定を追記した

☐  ModSecurity 設定ファイルを新環境の /usr/local/nginx/conf/ にコピーした

☐  modsec\_includes.conf 内のルールファイルパスを新環境に合わせて修正した

☐  proxy.conf に HTTPS (443) サーバブロックを追加した（必要な場合）

☐  nginx \-t で文法エラーがないことを確認した

☐  systemctl reload nginx でサービスを再起動した

☐  ModSecurity テストリクエストで 403 が返ることを確認した

☐  正常なリクエストがブロックされていないことを確認した

☐  アクセスログ・エラーログに異常がないことを確認した