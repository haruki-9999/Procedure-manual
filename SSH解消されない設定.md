# SSH解消されないコマンド




```bash
vi /etc/ssh/sshd_config
#設定ファイル内
ClientAliveInterval 60
ClientAliveCountMax 120
#60秒間隔で生存確認を行い、120回やっても動かなかったらログアウト（2時間）
service sshd restart
```