# SRX演習
## 1日目

事前準備
- COMMITを各チームで共有する
- Webサーバを構築して見れるようにする
 Webサーバを構築する
 Apacheをインストール後、startとenableする
- AMIを作成
 構築したものを複製できる


＜最終確認をお願いします＞
インターフェース設定
set interfaces ge-0/0/0 unit 0 family inet address 172.31.0.222/24
set interfaces ge-0/0/1 unit 0 family inet address 172.31.1.222/24
★

トラフィック制御（内→外）
set security policies from-zone trust to-zone untrust policy trust_to_untrust_teamf_1 match source-address any
set security policies from-zone trust to-zone untrust policy trust_to_untrust_teamf_1 match destination-address any
set security policies from-zone trust to-zone untrust policy trust_to_untrust_teamf_1 match application any
set security policies from-zone trust to-zone untrust policy trust_to_untrust_teamf_1 then permit
★

トラフィック制御（外→内）
set security policies from-zone untrust to-zone trust policy untrust_to_trust_teamf_1 match source-address any
set security policies from-zone untrust to-zone trust policy untrust_to_trust_teamf_1 match destination-address any
set security policies from-zone untrust to-zone trust policy untrust_to_trust_teamf_1 match application any
set security policies from-zone untrust to-zone trust policy untrust_to_trust_teamf_1 then permit
★

プール作成
set security nat destination pool pool_teamf_1 address 172.31.1.23/32
★

ルール作成・定義
set security nat destination rule-set 1 rule rule_teamf match destination-address 172.31.0.222/32
set security nat destination rule-set 1 rule rule_teamf then destination-nat pool pool_teamf_1


## 2日目
