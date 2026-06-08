# Webサーバ演習手順書（演習問題1から4）

## 問題1
Apacheをインストールと起動、どうさかくにんする
```zsh
dnf install httpd
systemctl start httpd
systemctl status httpd
```
http://グローバルIP/yakiniku/harami.html で検索をして見れるようにする
```zsh
vi /etc/httpd/conf/httpd.conf   
↓ここの部分を編集
<Directory "/var/www/yakiniku/harami.html">
    AllowOverride None
    Options None
    Require all granted
</Directory>
```

このままだと<Not Found
The requested URL was not found on this server.>が出てくる
```zsh
#下記で/yakinikkuを作る
mkdir /var/www/html/yakiniku
```

viでharami.htmlを作成する
中身には表示させたいものを記載する
```zsh
vi /hatami.html
```
/yakuniku配下に/harami.htmlをコピーする
```zsh
cp harami.html /var/www/html/yakiniku/
```
再度検索をかけると見れるようになっていいる

## 問題2
/home/sabaをドキュメントルートとして/home/saba配下に/miso/hoge.htmlをおいて表示させる
（/misoと/hoge.htmlを作成しておく）
```zsh
vi /etc/httpd/conf/httpd.conf
```
↓DocumentRootとDirectoryが一致しているようにする
```zsh
DocumentRoot "/home/saba"

#
# Relax access to content within /var/www.
#
<Directory "/home/saba">
    AllowOverride None
    # Allow open access:
    Require all granted
</Directory>
```
Apacheを再起動する
```zsh
systemctl restart httpd
```
再度確認してみて写っていたら成功

## 問題3はお預け


