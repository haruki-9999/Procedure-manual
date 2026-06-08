# EBS演習
***********
## 既存のEC2にEBSを追加する
***********

1. Amazon EBSの作成
ボリューム作成の手順は[こちらのリンク](https://docs.aws.amazon.com/ja_jp/ebs/latest/userguide/ebs-creating-volume.html)をクリックしてください
- Amazon EC2コンソールで(https://us-west-2.console.aws.amazon.com/ec2/home?region=us-west-2#Overview:)を開く
- ナビゲーションペインで[ボリューム]を選択する
- ボリュームタイプは汎用SSD(gp3)にする
- 選択するスループットは125(MiB/秒)を選択する
- [アベイラビリティーゾーン] では、ボリュームを作成するアベイラビリティーゾーンを選択します
- 空のスナップショットを作るならデフォルトの値にする(スナップショットからボリュームを作成しない)
- ここでは暗号化は無し
- ボリュームにカスタムタグを割り当てるにはタグセッションでタグの追加を行う
- 異常無しならボリュームを追加を選択する




2. インスタンスへのAmazon EBSボリュームのアタッチ
EBSへのEBSへのアタッチには[こちらのリンク](https://docs.aws.amazon.com/ja_jp/ebs/latest/userguide/ebs-attaching-volume.html)へ
- ナビゲーションペインで[ボリューム]を選択する
- アタッチするボリュームを選択し、アクション、ボリュームアタッチの順に選択する
 - アベイラビリティーゾーンを一致させないとインスタンスが選択できなくなる
 - デバイス名はどこに選択したかを覚えとく（あとで使う）
- 上記が完了したら[ボリュームのアタッチ]を選択する













3. LinuxでAmazon EBSボリュームが使用できるようにするのは[こちらのリンク](https://docs.aws.amazon.com/ja_jp/ebs/latest/userguide/ebs-using-volumes.html)へ
- SSHを利用してインスタンスに接続する
- 下記のコマンドを使用して使用可能なディスクデバイスとマウントポイントを表示し、正しいデバイス名を決める
```bash
lsblk
```
下記のような結果が出てくる
まだボリュームがインスタンスにアタッチされていない
```bash
NAME          MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
nvme1n1       259:0    0  10G  0 disk
nvme0n1       259:1    0   8G  0 disk
-nvme0n1p1    259:2    0   8G  0 part /
-nvme0n1p128  259:3    0   1M  0 part
```
- ボリュームにファイルシステムがあるかどうかを確認する
新しいボリュームはファイルシステムが存在しないので作成する必要がある
※スナップショットから作成されたボリュームはファイルが存在している
- 下記のコマンドでファイルが存在しているか確認することができる
```bash 
sudo file -s /dev/sdd
```
コマンド結果
```bash
/dev/sdd : 
```

デバイスにファイルシステムが存在した場合はファイルシステムの存在が確認できる
```bash
sudo file -s /dev/sdd
```
コマンド結果
```bash
/dev/sdd: SGI XFS filesystem data (blksz 4096, inosz 512, v2 dirs)
```
- lsblkコマンドでインスタンスにアタッチされている全てのデバイスを確認することができる
```bash
sudo lsblk -f
```
コマンド結果
```bash
NAME FSTYPE FSVER LABEL UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
nvme0n1
│                                                                           
├─nvme0n1p1
│    xfs          /     1bc1d419-553e-4a88-91ca-fa9f67d175e5    6.4G    20% /
├─nvme0n1p127
│                                                                           
└─nvme0n1p128
     vfat   FAT16       BAA9-6338                               8.7M    13% /boot/efi
nvme1n1
```
- システムファイルが作成されていない場合は下記のコマンドを実行する
```bash
sudo mkfs -t xfs /dev/sdd
```
作成したら念の為に確認する
```bash
lsblk -f
```
FSTYPE欄にxfsと記載があれば完了
```bash
FSTYPE
nvme1n1
     xfs                            
```
- xkfs.xfsがないというエラーが出たら次のコマンドでXSFをインストールしてから
前述のコマンドを実行する
```bash
sudo yum install xfsprogs
```
- mkdirコマンでを使用してボリュームのマウントポイントディレクトリを作成する
マウントポイントとはボリュームをマウントした後にファイルシステムツリー内でボリュームないで配置されファイルの読み書きを行う場所である
```bash
sudo mkdir /data
```
確認コマンド
```bash
ls -ld /data
drwxr-xr-x. 2 root root 6 Apr 18 03:36 /data
```
- 前のステップで作成したマウントポイントディレクトリにボリュームまたはパーテーションをマウントする（今回はパーテーションが存在しない）
```bash
sudo mount /dev/sdd /data
```
- 新しいボリュームマウントのファイルのアクセス許可をレビューしてユーザーとアプリケーションがボリュームに書き込みができることを確認する
- インスタンスを再起動した後にマウントポイントが自動的に保存されることはない
再起動後に自動的にマウントするには下記に手順実行する

### 再起動した後に接続するボリュームを自動的にマウントする
- （オプション）/etc.fstabファイルのバックアップコピーを行うと編集中の誤作動で削除してしまった時用のバックアップを取る
```bash
sudo cp /etc/fstab /etc/fstab.bak
```
- blkidコマンドを使用してUUIDを見つける
再起動後にはマウントするデバイスのUUIDを書き溜める
```bash
/dev/nvme0n1p1: LABEL="/" UUID="1bc1d419-553e-4a88-91ca-fa9f67d175e5" BLOCK_SIZE="4096" TYPE="xfs" PARTLABEL="Linux" PARTUUID="d947e621-1922-4589-ae28-4f8b5d16efa3"
/dev/nvme0n1p128: SEC_TYPE="msdos" UUID="BAA9-6338" BLOCK_SIZE="512" TYPE="vfat" PARTLABEL="EFI System Partition" PARTUUID="bfedbd6a-2a20-4781-8200-7c5f2d0d5a0b"
/dev/nvme1n1: UUID="9ba8b017-1aea-40a2-8a44-4e1aaf5b44d8" BLOCK_SIZE="512" TYPE="xfs"
/dev/nvme0n1p127: PARTLABEL="BIOS Boot Partition" PARTUUID="f22ac85f-cadc-4b72-ad8a-bd647bb2f0b8"
```
- vimでテキストエディタを使用して/etc/fstabを開く
```bash
sudo vim /etc/fstab
```
この１行を一番下に追加する
```bash
UUID=9ba8b017-1aea-40a2-8a44-4e1aaf5b44d8 /data xfs defaults,nofail 0 2
```
↑
UUIDはマウントしている/dateのものを使う
blkidで確認すると下記の記載がマウントした/dateのUUIDである
```bash
/dev/nvme1n1: UUID="9ba8b017-1aea-40a2-8a44-4e1aaf5b44d8" BLOCK_SIZE="512" TYPE="xfs"
```
- 内容を正しく確認するにはデバイスをアンマウントし、全てのファイルシステムを/etc/fstabにマウントする
エラーがなければ問題なし
```bash
sudo umount /data
sudo mount -a
```
- 実際にマウントされているか再度確認する
```bash
mount | grep /data
```
コマンド結果
```bash
/dev/nvme1n1 on /data type xfs (rw,relatime,seclabel,attr2,inode64,logbufs=8,logbsize=32k,sunit=8,swidth=8,noquota)
```

- 再起動する
```bash
sudo reboot
```

- 再起動後もマウントされているのか確認する
```bash
mount | grep /data
```
コマンド結果
```bash
/dev/nvme1n1 on /data type xfs (rw,relatime,seclabel,attr2,inode64,logbufs=8,logbsize=32k,sunit=8,swidth=8,noquota)
```bash
lsblkでも確認してみる
``bash
lsblk
NAME          MAJ:MIN RM SIZE RO TYPE MOUNTPOINTS
nvme1n1       259:0    0   1G  0 disk /data
nvme0n1       259:1    0   8G  0 disk 
├─nvme0n1p1   259:2    0   8G  0 part /
├─nvme0n1p127 259:3    0   1M  0 part 
└─nvme0n1p128 259:4    0  10M  0 part /boot/efi
```
# 上記の確認が取れれば完了！！












![画像]()