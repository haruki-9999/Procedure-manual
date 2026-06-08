# Postfix,Dovecot,Bindを用いたメールサーバーの構築

---

## 1. ドキュメント情報

| 項目 | 内容 |
|------|------|
| 手順書名 | 手順書_Postfix_Dovecot_Bind_基本構築 |
| 作成日 | 2026-05-18 |
| 最終更新日 | 2026-05-18 |
| 作成者 |  |
| バージョン | v1.0 |
| 対象環境 | AWS |

> **改訂履歴**
>
> | バージョン | 日付 | 変更内容 | 変更者 |
> |-----------|------|---------|--------|
> | v1.0 | 2026-05-18 | 初版作成 |  |

------------------------------

## 2. 目的・概要

### 2-1. 目的

> 本手順書では、2台EC2間でのメールのやり取りを行うメールサーバーの構築手順について説明する。
> 構築後は2台のEC2間でメールのやり取りが可能な状態を目指す。

### 2-2. 構成概要（アーキテクチャ）


### 2-3. 完成イメージ（ゴール定義）

- [ ] 自サーバーから自サーバーへメールを送ることができる
- [ ] 自サーバーから他サーバーへメールを送ることができる
- [ ] 自サーバーから他サーバーへユーザー名を指定してメールを送ることができる
- [ ] telnetを用いて他サーバーへ届いたメールを確認することができる

------------------------------

## 3. 前提条件・準備

### 3-1. 環境要件

| 項目 | 要件 |
|------|------|
| OS | AWS（Amazon Linux 2023）,WSL（Ubuntu 24.04） |
| SMTPサーバー | Postfix |
| POP3サーバー | Dovecot |
| DNSサーバー | Bind |
| メール管理ツール | Mailx |
| ログ管理サービス | Rsyslog |
| リモート操作コマンド | Telnet |

### 3-2. セキュリティグループ設定

| タイプ | プロトコル | ポート範囲 | ソース | 説明 |
|-------|------------|----------|--------|------|
| SSH | TCP | 22 | マイIP | ローカルPCからSSHで接続 |
| SMTP | TCP | 25 | 0.0.0.0/0 | メールがどこから転送されるか不明なため |
| POP3 | TCP | 110 | 172.31.0.0/16 | メールを受信する相手を許可するため |
| DNS | UDP | 53 | 0.0.0.0/0 | どこから名前解決依頼がくるか不明なため |

------------------------------

## 4. 構築手順（詳細）

> **注意事項**
> - コマンド中の `<山カッコ>` は自分の環境の値に置き換えること

------------------------------

### Step 1 システム設定

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
```

------------------------------

### Step 2 SMTPサーバーの設定

**目的：** Postfixをインストールし、SMTPサーバーの設定を行う

#### 操作手順

```bash
# PostfixとMailxをインストール
dnf install -y postfix mailx

# Postfixの設定ファイルのバックアップ取得（原本保存）
cp /etc/postfix/main.cf{,org}

# Postfixの設定ファイルを変更
vi /etc/postfix/main.cf

	#---以下のように変更-------------------
	myhostname =mail.<任意の名前>.local
	mydomain =<任意の名前>.local
	myorigin = $myhostname
	inet_interfaces = all
	mydestination = $mydomain , $myhostname
	mynetworks = 172.31.0.0/16, 127.0.0.1
	mail_spool_directory = /var/spool/mail/
	#--------------------------------------

# Postfixの起動と自動起動設定
systemctl enable --now postfix

# Postfixの起動確認
systemctl status postfix | less

# Postfixの自動起動設定確認
systemctl is-enabled postfix
```

------------------------------

### Step 2 DNSサーバーの設定

**目的：** Bindをインストールし、DNSサーバーの設定を行う

#### 操作手順

```bash
# Bindのインストール
dnf install -y bind

# Bindの設定ファイルのバックアップ取得（原本保存）
cp /etc/named.conf{,.org}

# Bindの設定ファイルの変更と追記
vi /etc/named.conf

	#---変更前------------------------
	listen-on port 53 { 127.0.0.1; };
	listen-on-v6 port 53 { ::1; };

	allow-query { localhost;};
	#---------------------------------
	#---変更後------------------------
	#listen-on port 53 { 127.0.0.1; };
	#listen-on-v6 port 53 { ::1; };

	allow-query { any;};
	#---------------------------------

	#---以下を設定ファイルの一番下に追記
	zone "<任意の名前>.local" IN {
		type master;
		file "/var/named/<任意の名前>.local.zone";
	}
	#---------------------------------

# 設定ファイルの構文チェック
named-checkconf

# ゾーンデータベースファイルの作成と記入
vi /var/named/sato.local.zone

	#---以下を記入-----------------------------------
	$TTL 3600
	@ IN SOA ns.<任意の名前>.local. test.gmail.com. (
	20210401 ; serial
	3600 ; refresh
	3600 ; retry
	3600 ; expire
	3600 ) ; minimum

		IN NS ns.<任意の名前>.local.
		IN MX 10 mail.<任意の名前>.local.

	ns IN A <DNSサーバーのプライベートIP>
	mail IN A <SMTPサーバーのプライベートIP>
	#-----------------------------------------------

# ゾーンデータベースファイルの構文チェック
named-checkzone <任意の名前>.local /var/named/<任意の名前>.local.zone

# Bindの起動と自動起動設定
systemctl enable --now named

# Bindの起動確認
systemctl status named | less

# Bindの自動起動設定確認
systemctl is-enabled named

# LinuxのDNSルールの設定ファイルのバックアップ取得（原本保存）
cp /etc/systemd/resolved.conf{,.org}

# LinuxのDNSルールの設定ファイルの変更
vi /etc/systemd/resolved.conf

	#---以下のように変更------------
	DNS=<DNSサーバーのプライベートIP>
	#------------------------------

# DNSルールの設定を反映させるため再起動
systemctl restart systemd-resolved.service
```

------------------------------

### Step 3 メール送受信確認（自サーバー）

**目的：** MailxとRsyslogを用いてメールの自サーバーでの送受信の確認を行う

#### 操作手順

```bash
# Rsyslogのインストール
dnf install -y rsyslog

# Rsyslogの起動と自動起動設定
systemctl enable --now rsyslog

# Rsyslogの起動確認
systemctl status rsyslog | less

# Rsyslogの自動起動設定確認
systemctl is-enabled rsyslog

# 自サーバーへメールの送信
mail -s <件名> root@<任意の名前>.local

	#---上記のコマンドを入力すると対話形式でメールを送る
	<内容>	内容を記入したらEnter
	.		「.」を入力してEnterを押してメールを送信
	#----------------------------------------------

# メールを送信できているか確認	
less /var/log/maillog

	#---以下が表示されているか確認
	status=sent
	#---------------------------

# mailコマンドでメールが届いているか確認
mail

	#---以下のように表示されていればメールが届いている--------------
	Heirloom Mail version 12.5 7/5/10.  Type ? for help.
	"/var/spool/mail/root": 1 message 1 new
	>N  1 root                  Mon May 18 17:44  17/523   "hi"
	& 
	rootの横の数字を入力してメールを閲覧可能
	#-----------------------------------------------------------

# メールの実ファイルを確認
ll /var/spool/mail/

	#---以下で確認----------------------------------------
	rootディレクトリが作成され、その中にメールが存在すれば成功
	#----------------------------------------------------
```

------------------------------

### Step 4 メール送受信確認（他サーバー）

**目的：** MailxとRsyslogを用いてメールの他サーバーでの送受信の確認を行う

#### 操作手順

```bash
# LinuxのDNSルールの設定ファイルの変更
vi /etc/systemd/resolved.conf

	#---以下のように変更------------
	DNS=<相手のDNSサーバーのプライベートIP>
	#------------------------------

# DNSルールの設定を反映させるため再起動
systemctl restart systemd-resolved.service

# mailコマンドで相手にメールが届くか確認
mail -s <件名> root@<相手の名前>.local

# Step 3と同じ手順でメールが届いているか確認
```

------------------------------

### Step 5 メールアドレスの作成

**目的：** メールアドレスの作成を行う

#### 操作手順

```bash
# メール用のユーザーを作成
useradd sato -g mail -M -K MAIL_DIR=/dev/null -s /sbin/nologin

# 作成したユーザーのパスワードを設定
passwd sato

	#---対話形式でパスワードを設定--------------------------
	New Password:
	Retry Password:
	passwd: all authentication tokens updated successfully.
	#------------------------------------------------------
```

### Step 6 POPサーバーの構築

**目的：** Dovecotの設定を行う

#### 操作手順

```bash
# Dovecotのインストール
dnf install -y dovecot

# Dovecotの設定ファイルのバックアップ取得（原本保存）
cp /etc/dovecot/dovecot.conf{,.org}

# Dovecotの設定ファイルの変更と追記
vi /etc/dovecot/dovecot.conf

	#---以下のように変更
	protocols = pop3
	#-----------------
	#---以下を設定ファイルの一番下に追記----------
	mail_location = maildir:/var/spool/mail/%u/
	#------------------------------------------

# Dovecotの暗号化設定ファイルのバックアップ取得（原本保存）
cp /etc/dovecot/conf.d/10-ssl.conf{,.org}

# Dovecotの暗号化設定ファイルの変更
vi /etc/dovecot/conf.d/10-ssl.conf

	#---以下のように変更
	#ssl = required
	#-----------------

# Dovecotのユーザー認証設定ファイルのバックアップ取得（原本保存）
cp /etc/dovecot/conf.d/10-auth.conf{,.org}

# Dovecotのユーザー認証設定ファイルの変更
vi /etc/dovecot/conf.d/10-auth.conf

	#---以下のように変更---------
	disable_plaintext_auth = no
	#--------------------------

# Dovecotの起動と自動起動設定
systemctl enable --now dovecot

# Dovecotの起動確認
systemctl status dovecot | less

# Dovecotの自動起動設定確認
systemctl is-enabled dovecot

# telnetのインストール
dnf install -y telnet

# メールが届いているか確認
telnet <SMTPサーバーのプライベートIP> 110

	#---作成したユーザーでログインしてメール確認
	user <作成したユーザー名>
	+OK
	pass <パスワード>
	+OK Logged in.
	#---------------------------------------
	#---メール確認--------------------------
	list
	+OK 1 messages;
	retr 1
	---メール閲覧---
	#--------------------------------------
```

------------------------------

## 付録（任意）

### A. コマンド解説

------------------------------

### B. 設定ファイル解説

------------------------------

### C. 用語解説

------------------------------

### D. 補足解説