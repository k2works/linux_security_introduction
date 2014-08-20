Linuxセキュリティ入門
===
# 目的
# 前提
| ソフトウェア     | バージョン    | 備考         |
|:---------------|:-------------|:------------|
| OS X           |10.8.5        |             |
| vagrant   　　　|1.6.3         |             |
| centOS         |6.5           |             |

# 構成
+ [セットアップ](#1)
+ [セキュアサーバのクイックセットアップ](#2)
+ [OSのセキュリティ](#3)
+ [ファイルシステムのセキュリティ](#4)
+ [ネットワークのセキュリティ](#5)
+ [SELinux](#6)
+ [システムログの管理](#7)
+ [セキュリティチェックと侵入探知](#8)
+ [DNSサーバーのセキュリティ](#9)
+ [Webサーバーのセキュリティ](#10)
+ [メールサーバーのセキュリティ](#11)
+ [FTPサーバーのセキュリティ](#12)
+ [SSH](#13)

# 詳細
## <a name="1">セットアップ</a>
ネットワーク設定  
_Vagrantfile_
```ruby
config.vm.network "private_network", ip: "192.168.33.10"
```
ネームサーバ追加  
_Vagrantfile_
```ruby
script = <<SCRIPT
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf > /dev/null
SCRIPT
・・・
  config.vm.provision "shell", inline: script
```
```bash
$ vagrant init chef/centos-6.5
$ vagrant up
$ ssh root@192.168.33.10
$ root@192.168.33.10's password:vagrant
```

## <a name="2">セキュアサーバのクイックセットアップ</a>
### セキュリティのクイック対応
#### 作業ユーザーの作成
```bash
# useradd centuser
# passwd centuser
```
#### ホストのセキュリティ
システムのアップデート
```bash
# yum -y update
```
ランレベル３で動作しているサービスを確認
```bash
# chkconfig --list | grep 3:on
auditd          0:off   1:off   2:on    3:on    4:on    5:on    6:off
blk-availability        0:off   1:on    2:on    3:on    4:on    5:on    6:off
crond           0:off   1:off   2:on    3:on    4:on    5:on    6:off
ip6tables       0:off   1:off   2:on    3:on    4:on    5:on    6:off
iptables        0:off   1:off   2:on    3:on    4:on    5:on    6:off
lvm2-monitor    0:off   1:on    2:on    3:on    4:on    5:on    6:off
netfs           0:off   1:off   2:off   3:on    4:on    5:on    6:off
network         0:off   1:off   2:on    3:on    4:on    5:on    6:off
nfslock         0:off   1:off   2:off   3:on    4:on    5:on    6:off
postfix         0:off   1:off   2:on    3:on    4:on    5:on    6:off
rpcbind         0:off   1:off   2:on    3:on    4:on    5:on    6:off
rpcgssd         0:off   1:off   2:off   3:on    4:on    5:on    6:off
rsyslog         0:off   1:off   2:on    3:on    4:on    5:on    6:off
sshd            0:off   1:off   2:on    3:on    4:on    5:on    6:off
udev-post       0:off   1:on    2:on    3:on    4:on    5:on    6:off
vboxadd         0:off   1:off   2:on    3:on    4:on    5:on    6:off
vboxadd-service 0:off   1:off   2:on    3:on    4:on    5:on    6:off
vboxadd-x11     0:off   1:off   2:off   3:on    4:off   5:on    6:off
```
不要なサービスを終了
```bash
# service auditd stop
# service netfs stop
# service postfix stop
# chkconfig auditd off
# chkconfig netfs off
# chkconfig postfix off
```
システム再起動
```bash
# shutdown -r now
```
#### ユーザーのセキュリティ
コンソールからのrootログインを禁止  
```bash
# echo > /etc/securetty
```
_/etc/ssh/sshd_config_の編集
```
Port 20022
・・・
PermitRootLogin no
```
SSHサーバーを再起動
```bash
# service sshd restart
Stopping sshd:                                             [  OK  ]
Starting sshd:                                             [  OK  ]
```
#### ネットワークのセキュリティ
開いていているポートを表示
```bash
# netstat -atun
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address               Foreign Address             State
tcp        0      0 0.0.0.0:111                 0.0.0.0:*                   LISTEN
tcp        0      0 0.0.0.0:38129               0.0.0.0:*                   LISTEN
tcp        0      0 0.0.0.0:20022               0.0.0.0:*                   LISTEN
tcp        0      0 192.168.33.10:22            192.168.33.1:54220          ESTABLISHED
tcp        0      0 :::111                      :::*                        LISTEN
tcp        0      0 :::52528                    :::*                        LISTEN
tcp        0      0 :::20022                    :::*                        LISTEN
udp        0      0 0.0.0.0:111                 0.0.0.0:*
```
iptablesによるファイアウォールの確認
```
# iptables -L
Chain INPUT (policy ACCEPT)
target     prot opt source               destination

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
```
作業ユーザーでログインした後rootになる
```bash
# exit
$ ssh -p 20022 centuser@192.168.33.10
centuser@192.168.33.10's password:sysop
$ su
パスワード:vagrant
```

## <a name="3">OSのセキュリティ</a>
### ソフトウェアの更新
#### YUMを利用するための準備
デフォルトのGPG公開鍵のインポート
```bash
# rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-6
```
サードパーティのリポジトリ追加
```bash
# rpm --import http://ftp.riken.jp/Linux/fedora/epel/RPM-GPG-KEY-EPEL-6
# rpm -ivh  http://ftp-srv2.kddilabs.jp/Linux/distributions/fedora/epel/6/i386/epel-release-6-8.noarch.rpm
# rpm --import http://apt.sw.be/RPM-GPG-KEY.dag.txt
# rpm -ivh http://pkgs.repoforge.org/rpmforge-release/rpmforge-release-0.5.3-1.el6.rf.x86_64.rpm
```
#### YUMの設定
_/etc/yum.repos.d/CentOS-Base.repo_
#### YUMの基本操作
```bash
# yum update
# yum install httpd
# yum update httpd
# yum remove httpd
# yum info httpd
# yum groupinstall "Development tools"
```
#### システムの自動的な更新
```bash
# yum install yum-cron
# service yum-cron start
# chkconfig yum-cron on
```
設定ファイル：_/etc/sysconfig/yum-cron_

### ユーザーアカウントの管理
#### 一般ユーザーのログイン管理
ユーザーをロック
```bash
usermod -L fred
```
ユーザーロックを解除
```
usermod -U fred
```
#### rootログインの禁止
```bash
# echo > /etc/securetty
```
#### suコマンドの利用
suコマンドを使えるユーザーを制限する  
_etc/pam.d/su_
```
auth            required        pam_wheel.so use_uid
```
wheelグループへcentuserユーザーを登録
```bash
# usermod -G wheel centuser
```
#### root権限の利用
```bash
# visudo
```
_/etc/sudoers_
```
## Allows people in group wheel to run all commands
%wheel  ALL=(ALL)       ALL
```
#### パスワードの管理
パスワードに有効期限を設定する
```bash
# chage centuser
```
パスワード有効期限の確認
```bash
# chage -l centuser
```
### サービス管理
#### 不要なサービスの停止
開いているポートの確認
```bash
# netstat -antup
```
不要なサービスの停止
```bash
# service avahi-daemon stop
# service portresvere stop
```
不要なサービスのデフォルト起動を停止
```bash
# chkconfig avahi-daemon off
# chkconfig portreserve off
```
#### xinetd
xinetdのインストール
```bash
# yum install xinetd
```
rsyncサービスを有効化
```bash
# yum install rsync
# chkconfig rsync on
```
xinetdベースのサービス設定ファイル  
_/etc/xinetd.d_
```bash
# ls -al /etc/xinetd.d/
合計 56
drwxr-xr-x.  2 root root 4096  8月 18 03:41 2014 .
drwxr-xr-x. 62 root root 4096  8月 18 03:37 2014 ..
-rw-------.  1 root root 1157 10月  7 17:35 2013 chargen-dgram
-rw-------.  1 root root 1159 10月  7 17:35 2013 chargen-stream
-rw-------.  1 root root 1157 10月  7 17:35 2013 daytime-dgram
-rw-------.  1 root root 1159 10月  7 17:35 2013 daytime-stream
-rw-------.  1 root root 1157 10月  7 17:35 2013 discard-dgram
-rw-------.  1 root root 1159 10月  7 17:35 2013 discard-stream
-rw-------.  1 root root 1148 10月  7 17:35 2013 echo-dgram
-rw-------.  1 root root 1150 10月  7 17:35 2013 echo-stream
-rw-r--r--.  1 root root  332 10月 23 08:29 2013 rsync
-rw-------.  1 root root 1212 10月  7 17:35 2013 tcpmux-server
-rw-------.  1 root root 1149 10月  7 17:35 2013 time-dgram
-rw-------.  1 root root 1150 10月  7 17:35 2013 time-stream
```
### TCP Wrapper
#### TCP Wrapperの基本設定
libwrapライブラリにリンクしているかの確定
```bash
# ldd /usr/sbin/sshd | grep libwrap
        libwrap.so.0 => /lib64/libwrap.so.0 (0x00007fdb597d9000)
```
_/etc/hosts.allow_ファイルの記述例
```
sshd: .lpic.jp
ALL: 192.168.2.
```
_/etc/hosts.deny_ファイルの記述例
```
ALL:ALL
```
### プロセスの監視
#### psコマンドによるプロセス監視
プロセスを表示する
```bash
# ps
```
全てのプロセスを表示する
```bash
# ps aux
```
#### topコマンドによるプロセスとシステムの監視
終了は「Q」
```bash
# top
```
### ウイスル対策
#### Clam AntiVirusのインストール
インストール
```
# yum install clamav
```
ウィルスデータベースをアップデート
```
# freshclam
```
#### ウイルスのスキャン
ウイルススキャンの実行
```bash
# cd /home
# clamscan -r
```
ウイルス発見時のメッセージ
```bash
/tmp/eicar.com: Eicar-Test-Signature FOUND
```
ウイルスの自動削除
```bash
# clamscan --remove
```
## <a name="4">ファイルシステムのセキュリティ</a>
### パーミッション
#### 所有権と所有グループ
所有権と所有グループの確認
```bash
$ touch samplefile
$ ls -l samplefile
-rw-rw-r--. 1 centuser centuser 0  8月 18 04:26 2014 samplefile
```
#### アクセス権
アクセス権の確認
```bash
$ touch sample.sh
$ ls -l
合計 0
-rw-rw-r--. 1 centuser centuser 0  8月 18 04:27 2014 sample.sh
-rw-rw-r--. 1 centuser centuser 0  8月 18 04:26 2014 samplefile
```
#### アクセス権の変更
グループとその他ユーザーに書き込み権限を追加
```bash
$ chmode go+w samplefile
```
その他ユーザーの読み取り権と書き込み権を追加
```bash
$ chmod o-rw samplefile
```
アクセス権を644に設定
```bash
$ chmod 644 samplefile
```
#### SUID,SGID
samplefileにSUIDを設定
```bash
$ chmod u+s samplefile
```
samplefileにSGIDを設定
```bash
$ chmod g+s sampefile
```
#### スティッキービット
sampledirにスティッキービットを設定
```
$ mkdir sampledir
$ chmod o+t sampledir
```
#### デフォルトのアクセス権
umask値の確認
```bash
$ umask
0002
```
umask値の変更
```bash
$ umask 027
$ touch samplefile2
[centuser@localhost sampledir]$ ls -l
合計 0
-rw-rw-r--. 1 centuser centuser 0  8月 18 04:27 2014 sample.sh
-rwSr-Sr--. 1 centuser centuser 0  8月 18 04:48 2014 samplefile
-rw-r-----. 1 centuser centuser 0  8月 18 04:48 2014 samplefile2
```
### ACL
#### ACLの概要
ACLエントリの表示
```bash
$ touch testfile
$ getfacl testfile
# file: testfile
# owner: centuser
# group: centuser
user::rw-
group::r--
other::---
```
デフォルトACLを設定したディレクトリの例
```bash
$ mkdir testdir
$ getfacl testdir/
# file: testdir/
# owner: centuser
# group: centuser
user::rwx
group::r-x
other::---
```
#### ACLの設定
ユーザーcentuserのACLエントリを設定
```bash
$ setfacl -m user:centuser:rw testfile
```
マスク値の設定
```bash
$ setfacl -m mask::r testfile
```
マスク値の設定(省略形の記述)
```bash
$ setfacl -m mask::rx testfile
```
centuserユーザーのACLエントリの削除
```bash
$ setfacl -x user:centuser: testfile
```
すべてのACLエントリの削除
```bash
$ setfacl --remove-all testfile
```
### ファイルとファイルシステムの暗号化
#### GnuPGによるファイルの暗号化
##### 共通鍵暗号方式による暗号化
samplefileファイルを暗号化
```bash
$ gpg -c samplefile
```
samplefile.gpgを複合
```bash
$ gpg samplefile.gpg
```
##### 公開鍵暗号方式による暗号化
公開鍵と秘密鍵の鍵ペアを作成
```bash
$ gpg --gen-key
```
公開鍵の出力
```bash
$ gpg -o centuser.pub -a --export centuser@example.com
```
公開鍵anyuser.pubをGPGデータベースに追加
```bash
$ gpg --import anyuser.pub
```
フィンガープリントを表示
```bash
$ gpg --fingerprint
```
公開鍵を使って暗号化
```bash
$ gpg -a -e -r aka@example.com samplefile
```
暗号化ファイルを復号
```
$ gpg samplefile.asc
```
#### ファイルシステムの暗号化
cryptsetup-luksパッケージをインストール
```bash
$ su root
# yum -y install cryptsetup-luks
```
/dev/sdb1を暗号化
```bash
# cryptsetup create secret /dev/sdb1
```
暗号化ファイルシステムのデバイス名
```bash
# ls /dev/mapper
```
/dev/mapper/secretにext4ファイルシステムを作成
```bash
# mkfs -t ext4 /dev/mapper/secret
```
暗号化ファイルシステムをマウント
```bash
# mkdir /mnt/secret
# mount /dev/mapper/secret /mnt/secret
```
暗号化ファイルシステムの利用終了
```bash
# umount /dev/mapper/secret
# cryptsetup remove secret
```
暗号化ファイルシステムのマウント
```bash
# cryptsetup create secret /dev/sdb1
Enter passphrase:
# mount /dev/mapper/secret /mnt/secret
```
## <a name="5">ネットワークのセキュリティ</a>
### ネットワークの基本設定
NetworkManagerの無効化
```bash
# service NetworkManager stop
# chkconfig NetworkManager off
```
#### ネットワーク設定ファイル
+ _/etc/sysconfig/network_
+ _/etc/sysconfig/network-scripts/ifcfg-eth0_
+ _/etc/resolv.conf_

#### /etc/hosts
```bash
# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
```
#### 基本的なネットワーク管理コマンド
##### networkサービスの制御
```bash
# service network restart
Shutting down interface eth0:                              [  OK  ]
Shutting down interface eth1:                              [  OK  ]
Shutting down loopback interface:                          [  OK  ]
Bringing up loopback interface:                            [  OK  ]
Bringing up interface eth0:
Determining IP information for eth0... done.
                                                           [  OK  ]
Bringing up interface eth1:  Determining if ip address 192.168.33.10 is already in use for device eth1...
                                                           [  OK  ]
```
##### ifconfigコマンド
```bash
# ifconfig
eth0      Link encap:Ethernet  HWaddr 08:00:27:CE:08:3D
          inet addr:10.0.2.15  Bcast:10.0.2.255  Mask:255.255.255.0
          inet6 addr: fe80::a00:27ff:fece:83d/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:161484 errors:0 dropped:0 overruns:0 frame:0
          TX packets:48241 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:225858498 (215.3 MiB)  TX bytes:2657587 (2.5 MiB)

eth1      Link encap:Ethernet  HWaddr 08:00:27:3C:3C:85
          inet addr:192.168.33.10  Bcast:192.168.33.255  Mask:255.255.255.0
          inet6 addr: fe80::a00:27ff:fe3c:3c85/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:15690 errors:0 dropped:0 overruns:0 frame:0
          TX packets:11425 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:1257222 (1.1 MiB)  TX bytes:1885481 (1.7 MiB)

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:16436  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:0 (0.0 b)  TX bytes:0 (0.0 b)
```
##### netstatコマンド
```bash
# netstat
Active Internet connections (w/o servers)
Proto Recv-Q Send-Q Local Address               Foreign Address             State
tcp        0      0 192.168.33.10:20022         192.168.33.1:56265          ESTABLISHED
Active UNIX domain sockets (w/o servers)
Proto RefCnt Flags       Type       State         I-Node Path
unix  7      [ ]         DGRAM                    10077  /dev/log
unix  2      [ ]         DGRAM                    8476   @/org/kernel/udev/udevd
unix  2      [ ]         DGRAM                    58047
unix  2      [ ]         DGRAM                    56940
unix  3      [ ]         STREAM     CONNECTED     45780
unix  3      [ ]         STREAM     CONNECTED     45779
unix  2      [ ]         DGRAM                    45776
unix  2      [ ]         DGRAM                    33262
unix  2      [ ]         DGRAM                    11248
unix  3      [ ]         DGRAM                    8495
unix  3      [ ]         DGRAM                    8494
```
開いているTCPポートを確認
```bash
# netstat -at
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address               Foreign Address             State
tcp        0      0 *:sunrpc                    *:*                         LISTEN
tcp        0      0 *:20022                     *:*                         LISTEN
tcp        0      0 *:40095                     *:*                         LISTEN
tcp        0      0 192.168.33.10:20022         192.168.33.1:56265          ESTABLISHED
tcp        0      0 *:sunrpc                    *:*                         LISTEN
tcp        0      0 *:50162                     *:*                         LISTEN
tcp        0      0 *:20022                     *:*                         LISTEN
```
##### pigコマンド
pingコマンドの実行例
```bash
# ping 192.168.33.1
PING 192.168.33.1 (192.168.33.1) 56(84) bytes of data.
64 bytes from 192.168.33.1: icmp_seq=1 ttl=64 time=0.220 ms
64 bytes from 192.168.33.1: icmp_seq=2 ttl=64 time=0.243 ms
```
ICMPパケットを３回送信
```bash
# ping -c 3 192.168.33.1
PING 192.168.33.1 (192.168.33.1) 56(84) bytes of data.
64 bytes from 192.168.33.1: icmp_seq=1 ttl=64 time=0.155 ms
64 bytes from 192.168.33.1: icmp_seq=2 ttl=64 time=0.187 ms
64 bytes from 192.168.33.1: icmp_seq=3 ttl=64 time=0.507 ms

--- 192.168.33.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2002ms
rtt min/avg/max/mdev = 0.155/0.283/0.507/0.158 ms
```
192.168.0.0/24のホストにICMPパケットを送信
```bash
# ping -b 192.168.11.255
```
##### tracerouteコマンド
```
# yum -y install traceroute
# traceroute www.centos.org
```
##### hostコマンド
```bash
# yum -y install bind-utils
# host sv1.lpi.jp
sv1.lpi.jp has address 203.174.74.34
# host 203.174.74.34
34.74.174.203.in-addr.arpa domain name pointer sv1.lpi.jp.
```
#### ネットワーク探査対策
ICMPパケットを無視
```bash
# echo "1" > /proc/sys/net/ipv4/icmp_echo_ignore_all
```
プロードキャスト宛のICMPパケットのみ無視
```bash
# echo "1" > /proc/sys/net/ipv4/icmp_echo_ignore_broadcasts
```
永続的に設定する場合は以下を追加  
_/etc/sysctl.conf_
```bash
net.ipv4.icmp_echo_ignore_broadcasts="1"
```
### ファイアウォール
#### iptablesコマンド
INPUTチェインにルールを追加
```bash
# iptables -A INPUT -p tcp -s 192.168.0.0/24 --dport 22 -j ACCEPT
```
INPUTチェインの２番めのルールを削除
```bash
# iptables -D INPUT 2
```
INPUTチェインの全ルールを削除
```bash
# iptables -F INPUT
```
iptableの設定を保存
```bash
# service iptables save
iptables: Saving firewall rules to /etc/sysconfig/iptables:[  OK  ]
```
iptablesの設定を適用
```bash
# service iptables start
iptables: Applying firewall rules:                         [  OK  ]
```
iptables-saveコマンドで設定を保存
```bash
# iptables-save > /etc/iptables.tmp
```
/etc/iptables.tmpから設定をリストア
```bash
# iptables-restore < /etc/iptables.tmp
```
#### 基本的なパケットフィルタリングの設定
```bash
# iptables -A INPUT -p tcp -d 192.168.33.10 --dport 20022 -j ACCEPT
# iptables -A INPUT -p tcp -d 192.168.33.10 --dport 22 -j ACCEPT
# iptables -A INPUT -p tcp -d 192.168.33.10 --dport 25 -j ACCEPT
# iptables -A INPUT -p tcp -d 192.168.33.10 --dport 53 -j ACCEPT
# iptables -A INPUT -p udp -d 192.168.33.10 --dport 53 -j ACCEPT
# iptables -A INPUT -p tcp -d 192.168.33.10 --dport 80 -j ACCEPT
# iptables -A INPUT -p tcp -d 192.168.33.10 --dport 110 -j ACCEPT
# iptables -A INPUT -i lo -j ACCEPT
# iptables -A INPUT -p icmp -j ACCEPT
# iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
# iptables -P OUTPUT ACCEPT
# iptables -P FORWARD DROP
# iptables -P INPUT DROP
# service iptables save
iptables: Saving firewall rules to /etc/sysconfig/iptables:[  OK  ]
```
#### iptablesの応用
マッチするパケットをログに記録
```bash
# iptables -A INPUT -s 192.168.33.0/24 -j LOG
```
192.168.33.20からのパケットをログに記録して拒否
```bash
# iptables -A INPUT -s 192.168.33.20 -j LOG
# iptables -A INPUT -s 192.168.33.20 -j DROP
```
192.168.33.20からのパケットをメッセージ付きでログに記録して拒否
```bash
# iptables -A INPUT -s 192.168.33.20 -j LOG --log-prefix "Droped IP :"
# iptables -A INPUT -s 192.168.33.20 -j DROP
```
IP偽装対策
```bash
# iptables -A INPUT -s 10.0.0.0/8 -j DROP
# iptables -A INPUT -s 172.16.0.0/12 -j DROP
# iptables -A INPUT -s 192.168.0.0/16 -j DROP
# iptables -A INPUT -s 127.0.0.0/8 -j DROP
# iptables -A INPUT -s 169.254.0.0/16 -j DROP
# iptables -A INPUT -s 192.0.2.0/24 -j DROP
# iptables -A INPUT -s 224.0.0.0/4 -j DROP
# iptables -A INPUT -s 240.0.0.0/5 -j DROP
```
Smurf攻撃対策
```bash
# iptables -A INPUT -d 0.0.0.0/8 -j DROP
# iptables -A INPUT -d 255.255.255.255 -j DROP
```
#### CUIツールによるパケットフィルタリング設定
```bash
# yum install system-config-firewall
# system-config-firewall
```
## <a name="6">SELinux</a>
### SELinuxの概要
#### セキュリティコンテキスト
ファイルのセキュリティコンテキストを表示
```bash
# ls -lZ /etc/inittab
-rw-r--r--. root root system_u:object_r:etc_t:s0       /etc/inittab
```
プロセスのセキュリティコンテキストを表示
```bash
# ps -Z
LABEL                             PID TTY          TIME CMD
unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 12671 pts/1 00:00:00 su
unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 12675 pts/1 00:00:00 bash
unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 12963 pts/1 00:00:00 ps
```
ユーザーのセキュリティコンテキストを表示
```bash
# id -Z
unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
```
#### 動作モード
SELinuxの動作モードを表示
```
# getenforce
```
SELinuxをPermissiveモードに変更
```bash
# setenforce permissive
```
#### ポリシー
SELinuxの状況を確認
```bash
# sestatus
SELinux status:                 enabled
SELinuxfs mount:                /selinux
Current mode:                   permissive
Mode from config file:          permissive
Policy version:                 24
Policy from config file:        targeted
```
### SELinuxの設定
#### 論理パラメータの設定
論理パラメータの確認
```bash
# getsebool -a
```
httpd_enable_homedirsの確認
```bash
# getsebool httpd_enable_homedirs
httpd_enable_homedirs --> off
```
httpd_enable_homedirsをonにする
```bash
# setsebool httpd_enable_homedirs on
```
httpd_enable_homedirsを永続的にonにする
```bash
# setsebool -P httpd_enable_homedirs on
```
#### ファイルのセキュリティコンテキストの変更
#### ファイルのコピーとバックアップ

## <a name="7">システムログの管理</a>
### システムログの概要
#### rsyslogの設定
_/etc/rsyslog.conf_
```bash
# cat /etc/rsyslog.conf
# rsyslog v5 configuration file

# For more information see /usr/share/doc/rsyslog-*/rsyslog_conf.html
# If you experience problems, see http://www.rsyslog.com/doc/troubleshoot.html

#### MODULES ####

$ModLoad imuxsock # provides support for local system logging (e.g. via logger command)
$ModLoad imklog   # provides kernel logging support (previously done by rklogd)
#$ModLoad immark  # provides --MARK-- message capability

# Provides UDP syslog reception
#$ModLoad imudp
#$UDPServerRun 514

# Provides TCP syslog reception
#$ModLoad imtcp
#$InputTCPServerRun 514


#### GLOBAL DIRECTIVES ####

# Use default timestamp format
$ActionFileDefaultTemplate RSYSLOG_TraditionalFileFormat

# File syncing capability is disabled by default. This feature is usually not required,
# not useful and an extreme performance hit
#$ActionFileEnableSync on

# Include all config files in /etc/rsyslog.d/
$IncludeConfig /etc/rsyslog.d/*.conf


#### RULES ####

# Log all kernel messages to the console.
# Logging much else clutters up the screen.
#kern.*                                                 /dev/console

# Log anything (except mail) of level info or higher.
# Don't log private authentication messages!
*.info;mail.none;authpriv.none;cron.none                /var/log/messages

# The authpriv file has restricted access.
authpriv.*                                              /var/log/secure

# Log all the mail messages in one place.
mail.*                                                  -/var/log/maillog


# Log cron stuff
cron.*                                                  /var/log/cron

# Everybody gets emergency messages
*.emerg                                                 *

# Save news errors of level crit and higher in a special file.
uucp,news.crit                                          /var/log/spooler

# Save boot messages also to boot.log
local7.*                                                /var/log/boot.log


# ### begin forwarding rule ###
# The statement between the begin ... end define a SINGLE forwarding
# rule. They belong together, do NOT split them. If you create multiple
# forwarding rules, duplicate the whole block!
# Remote Logging (we use TCP for reliable delivery)
#
# An on-disk queue is created for this action. If the remote host is
# down, messages are spooled to disk and sent when it is up again.
#$WorkDirectory /var/lib/rsyslog # where to place spool files
#$ActionQueueFileName fwdRule1 # unique name prefix for spool files
#$ActionQueueMaxDiskSpace 1g   # 1gb space limit (use as much as possible)
#$ActionQueueSaveOnShutdown on # save messages to disk on shutdown
#$ActionQueueType LinkedList   # run asynchronously
#$ActionResumeRetryCount -1    # infinite retries if host is down
# remote host is: name/ip:port, e.g. 192.168.0.1:514, port optional
#*.* @@remote-host:514
# ### end of the forwarding rule ###
```
#### ログサーバーの設定
ログを192.168.11.2のホストに送る(UDP)  
```
*.warning        @192.168.11.2
```
ログサーバーの/etc/rsyslog.confの設定(UDP)  
```
$ModLoad imudp.so
$UDPServerRun 514
```
ログを192.168.11.2のホストに送る(TCP)  
```
*.warning        @192.168.11.2
```
ログサーバーの/etc/rsyslog.confの設定(TCP)  
```
$ModLoad imtcp.so
$TCPServerRun 514
```
ログサーバーの/etc/rsyslog.confの設定(TCP 10514番ポート)  
```
$ModLoad imtcp.so
$TCPServerRun 10514
```
ログを192.168.11.2のホストに送る(TCP 10514番ポート)
```
*.warning        @192.168.11.2:10514
```
#### ログのローテーション
```
# cat /etc/logrotate.conf
# see "man logrotate" for details
# rotate log files weekly
weekly

# keep 4 weeks worth of backlogs
rotate 4

# create new (empty) log files after rotating old ones
create

# use date as a suffix of the rotated file
dateext

# uncomment this if you want your log files compressed
#compress

# RPM packages drop log rotation information into this directory
include /etc/logrotate.d

# no packages own wtmp and btmp -- we'll rotate them here
/var/log/wtmp {
    monthly
    create 0664 root utmp
        minsize 1M
    rotate 1
}

/var/log/btmp {
    missingok
    monthly
    create 0600 root utmp
    rotate 1
}
```
### ログの監視
#### ログファイルの監視
|  ログファイル    | 説明         |
|:---------------|:-------------|
| var/log/message|システムの汎用ログファイル  |
| var/log/secure |認証関連の記録 |
| var/log/boot.log|サービスの起動／停止の記録 |
| var/log/cron   |cronジョブの実行結果 |
| var/log/dmesg  |カーネルが出力するメッセージの記録 |
| var/log/lastlog|ログインの記録 |
| var/log/maillog|メールサブシステムの記録 |
| var/log/wtmp   |ログインの記録 |
| var/log/yum.log|YUMによるパッケージ情報操作記録 |

#### logwatch
logwatchの確認  
```bash
$ rpm -q logwatch
```
logwatchのインストール  
```bash
$ yum install logwatch
```
設定ファイルは_/usr/share/logwatch/default.conf/logwatch.conf_

#### swatch
swatchのインストール  
```bash
# yum install swatch perl-File-Tail
```
_/root/.swatchrc_  
```
watchfor /Accepted password/
  echo
```
swatchの起動
```
# swatch -c .swatchrc -t /var/log/secure

*** swatch version 3.2.3 (pid:1200) started at 2014年  8月 19日 火曜日 01:06:24 UTC
```
swatchの設定
```
watchfor /パターン/
      アクション
```
|  アクションの例  | 説明         |
|:---------------|:-------------|
| echo           |標準出力に出力する |
| execコマンド    |指定したコマンドを実行する |
| bell           |ベルを鳴らす(回数指定可能) |
| mail addresses=メールアドレス,subject=件名 |指定したメールアドレスにメッセージを送信する |

swatchの設定例
```
watchfor /Failed \(login|password\)/i
  echo
  mail root@localhost,subject="Failed Login"
```
バックグラウンドでswatchを実行
```
# swatch -c .swatchrc -t /var/log/secure --daemon
```
## <a name="8">セキュリティチェックと侵入探知</a>
### ポートスキャンとパケットキャプチャ
#### nmap
nmapのインストール
```bash
# yum -y install nmap
```
192.168.33.20をポートスキャン
```bash
$ nmap 192.168.33.20

Starting Nmap 5.51 ( http://nmap.org ) at 2014-08-19 01:43 UTC
Nmap scan report for 192.168.33.20
Host is up (0.0040s latency).
Not shown: 998 closed ports
PORT    STATE SERVICE
22/tcp  open  ssh
111/tcp open  rpcbind

Nmap done: 1 IP address (1 host up) scanned in 0.25 seconds
```
サーバーソフトウェアの詳細を調べる
```bash
$ nmap -A 192.168.33.20

Starting Nmap 5.51 ( http://nmap.org ) at 2014-08-19 01:45 UTC
Nmap scan report for 192.168.33.20
Host is up (0.0031s latency).
Not shown: 998 closed ports
PORT    STATE SERVICE VERSION
22/tcp  open  ssh     OpenSSH 5.3 (protocol 2.0)
| ssh-hostkey: 1024 5e:98:a6:5a:e4:1d:61:9a:31:97:1e:8b:68:d5:92:82 (DSA)
|_2048 c4:4d:f9:05:09:31:33:05:cd:99:52:5b:fc:e0:10:b5 (RSA)
111/tcp open  rpcbind

Service detection performed. Please report any incorrect results at http://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 6.64 seconds
```
Xmasツリースキャン
```bash
$ sudo nmap -sX 192.168.33.20

Starting Nmap 5.51 ( http://nmap.org ) at 2014-08-19 01:55 UTC
Nmap scan report for 192.168.33.20
Host is up (0.0010s latency).
Not shown: 998 closed ports
PORT    STATE         SERVICE
22/tcp  open|filtered ssh
111/tcp open|filtered rpcbind
MAC Address: 08:00:27:2F:7D:D6 (Cadmus Computer Systems)

Nmap done: 1 IP address (1 host up) scanned in 2.44 seconds
```
全てのポートをスキャン
```
$ nmap -p 0-65535 192.168.33.20

Starting Nmap 5.51 ( http://nmap.org ) at 2014-08-19 01:57 UTC
Nmap scan report for 192.168.33.20
Host is up (0.0022s latency).
Not shown: 65533 closed ports
PORT      STATE SERVICE
22/tcp    open  ssh
111/tcp   open  rpcbind
35186/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 10.84 seconds
```
1023番までと60000番以降のポートをスキャン
```bash
$ nmap -p -01023,60000- 192.168.33.20

Starting Nmap 5.51 ( http://nmap.org ) at 2014-08-19 01:58 UTC
Nmap scan report for 192.168.33.20
Host is up (0.0037s latency).
Not shown: 6557 closed ports
PORT    STATE SERVICE
22/tcp  open  ssh
111/tcp open  rpcbind

Nmap done: 1 IP address (1 host up) scanned in 1.15 seconds
```
#### tcpdump
53番ポートへのアクセスの監視  
```bash
$ sudo tcpdump -nli eth0 port 53
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 65535 bytes
02:16:00.932490 IP 10.0.2.15.51321 > 8.8.8.8.domain: 37346+ A? www.yahoo.com. (31)
02:16:00.932765 IP 10.0.2.15.51321 > 8.8.8.8.domain: 62438+ AAAA? www.yahoo.com. (31)
```
53番ポートの通信内容を表示
```bash
$ sudo tcpdump -X -i eth0 port 53
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 65535 bytes
02:23:59.481956 IP 10.0.2.15.42014 > google-public-dns-a.google.com.domain: 36744+ A? www.yahoo.co.jp. (33)
        0x0000:  4500 003d 2aa6 4000 4011 f3eb 0a00 020f  E..=*.@.@.......
        0x0010:  0808 0808 a41e 0035 0029 1c59 8f88 0100  .......5.).Y....
        0x0020:  0001 0000 0000 0000 0377 7777 0579 6168  .........www.yah
        0x0030:  6f6f 0263 6f02 6a70 0000 0100 01         oo.co.jp.....
```
ICMPパケットを監視
```bash
$ sudo tcpdump -nli eth0 proto \\icmp
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 65535 bytes
```
telnet接続を監視
```bash
$ sudo tcpdump -nli eth0 port 23
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 65535 bytes
```
### Tripwire
#### Tripwireの設定
Tripwireのインストール
```bash
# yum install tripwire
```
サイトキーとローカルキーの生成
```
# tripwire-setup-keyfiles
```
#### Tripwireの設定
Tripwireの設定ファイル等(_/etc/tripwire/_)

|  ファイル名  | 説明         |
|:---------------|:-------------|
| tw.cfg         | Tripwireの全体設定ファイル|
| twcfg.txt      | 全体設定ファイルのひな形|
| tw.pl          | ポリシーファイル|
| twpol.txt      | ポリシーファイルのひな形|
| site.key       | サイトキーファイル|
| ホスト名.local.key   | ホストキーファイル|

tw.cfgファイルの生成
```bash
# cd /etc/tripwire/
# twadmin -m F -c tw.cfg -S site.key twcfg.txt
Please enter your site passphrase:
Wrote configuration file: /etc/tripwire/tw.cfg
```
tw.polファイルの生成
```bash
# twadmin -m P -S site.key twpol.txt
Please enter your site passphrase:
Wrote policy file: /etc/tripwire/tw.pol
```
#### Tripwireの運用
ベースラインデータベースの初期化
```bash
# tripwire --init
Please enter your local passphrase:
Parsing policy file: /etc/tripwire/tw.pol
Generating the database...
```
tw.polファイルの再生性とベースラインデータベースの再作成
```bash
# twadmin -m P -S site.key twpol.txt
Please enter your site passphrase:
Wrote policy file: /etc/tripwire/tw.pol
# tripwire --init
Please enter your local passphrase:
Parsing policy file: /etc/tripwire/tw.pol
Generating the database...
```
改ざんチェック
```bash
# tripwire --check
```
レポートの表示
```bash
# twprint -m r -r /var/lib/tripwire/report/host1-20140819-031031.twr
```
ベースラインデータベースのアップデート
```bash
# tripwire -m u -r /var/lib/tripwire/report/host1-20140819-031031.twr
```
### Rootkit Humterとfail2banの利用
#### Rootkit Hunter
Rootkit Hunterのインストール
```bash
# yum install rkhunter
```
rootkitデータベースのアップデート
```bash
# rkhunter --update
[ Rootkit Hunter version 1.4.2 ]

Checking rkhunter data files...
  Checking file mirrors.dat                                  [ No update ]
  Checking file programs_bad.dat                             [ No update ]
  Checking file backdoorports.dat                            [ No update ]
  Checking file suspscan.dat                                 [ Updated ]
  Checking file i18n/cn                                      [ No update ]
  Checking file i18n/de                                      [ Updated ]
  Checking file i18n/en                                      [ No update ]
  Checking file i18n/tr                                      [ Updated ]
  Checking file i18n/tr.utf8                                 [ Updated ]
  Checking file i18n/zh                                      [ Updated ]
  Checking file i18n/zh.utf8                                 [ Updated ]
```
Rootkit Hunterの実行
```bash
# rkhunter --check
```
#### fail2ban
fail2banのインストール
```bash
# yum install fail2ban
```
+ 設定ファイル_/etc/fail2ban/fail2ban.conf_  
+ jailの設定_/etc/fail2ban/jail.conf_  

fail2banの起動
```bash
# service fail2ban start
```
iptablesによるfail2banの確認
```bash
# iptables -L
Chain INPUT (policy ACCEPT)
target     prot opt source               destination
fail2ban-SSH  tcp  --  anywhere             anywhere            tcp dpt:ssh

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination

Chain fail2ban-SSH (1 references)
target     prot opt source               destination
REJECT     all  --  192.168.33.20        anywhere            reject-with icmp-port-unreachable
RETURN     all  --  anywhere             anywhere
```
ssh-iptablesの状況を表示
```bash
# fail2ban-client status ssh-iptables
Status for the jail: ssh-iptables
|- filter
|  |- File list:        /var/log/secure
|  |- Currently failed: 1
|  `- Total failed:     6
`- action
   |- Currently banned: 1
   |  `- IP list:       192.168.33.20
   `- Total banned:     1
```

## <a name="9">DNSサーバーのセキュリティ</a>
### BINDの基本設定
#### BINDのインストール
```bash
# yum install bin bind-chroot
```
起動
```bash
# rndc-confgen -a -r /dev/urandom -t /var/named/chroot
# service named start
```
#### rndcコマンド
```bash
# chmod o-rwx /etc/rndc.key
```
example.comゾーンを再度読み込み
```bash
# rndc reload example.com
```
設定ファイルと新規ゾーンのみ再読込
```bash
rndc reconfig
```
BINDの再起動
```bash
service named restart
```
## <a name="10">Webサーバーのセキュリティ</a>
### Apacheの基本
#### Apacheのインストールと基本
httpdパッケージの確認
```bash
# rpm -q httpd
```
Apacheと関連パッケージのインストール
```bash
# yum groupinstall "Web Server"
```
Apacheの設定ファイル

|  ファイル名  | 説明         |
|:---------------|:-------------|
| /etc/httpd/conf/httpd.conf | メイン設定ファイル|
| /etc/httpd/conf/magic | MIMEタイプの設定|
| /etc/httpd/conf.d/manual.conf | オンラインマニュアルの設定|
| /etc/httpd/conf.d/perl.conf| Perlの設定(mod_perlをインストールした場合)|
| /etc/httpd/conf.d/php.conf| PHPの設定(mod_phpをインストールした場合)|
| /etc/httpd/conf.d/ssl.conf| SSL/TLSの設定(mod_sslをインストールした場合)|

Apacheを起動する
```bash
# service httpd start
```
設定の再読み込み
```bash
# service httpd reload
```
Apacheの再起動
```bash
# service httpd restart
```
Apacheの安全な再起動
```bash
# service httpd graceful
```

## <a name="11">メールサーバーのセキュリティ</a>
### メールサーバーの基礎
#### Postfixのインストールと基本
postfixパッケージの確認
```bash
# rpm -q postfix
```
#### Dovecotのインストールと基本
dovecotパッケージの確認
```bash
# rpm -q dovecot
```
Dovecotのインストール
```bash
# yum install dovecot
```
## <a name="12">FTPサーバーのセキュリティ</a>
### FTPの基本
lftpインストール
```bash
# yum install lftp
```
vsftpdのインストール
```bash
# yum install vsftpd
# service vsftpd start
```
設定ファイルは_/etc/vsftpd/vsftpd.conf_  

## <a name="13">SSH</a>
前提としてhost1とhost2にcentuserが存在すること

### SSHの基本
鍵のフィンガープリントを確認
```bash
# ssh-keygen -i
```
### 公開鍵の作成
```bash
$ ssh-keygen -t rsa
Generating public/private rsa key pair.
Enter file in which to save the key (/home/centuser/.ssh/id_rsa):
Created directory '/home/centuser/.ssh'.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /home/centuser/.ssh/id_rsa.
Your public key has been saved in /home/centuser/.ssh/id_rsa.pub.
The key fingerprint is:
ce:0b:3f:79:82:9f:aa:ad:e2:4e:e8:2d:4b:38:44:ce centuser@host1
The key's randomart image is:
+--[ RSA 2048]----+
|                 |
|                 |
| .               |
|+                |
| E      S        |
|o.     o         |
|+..   ..o.       |
|o+o  ..o+o.      |
| ==ooooo=+       |
+-----------------+
```
公開鍵をサーバーに登録
```bash
$ ssh-copy-id centuser@192.168.33.20
Could not open a connection to your authentication agent.
centuser@192.168.33.20's password:
Now try logging into the machine, with "ssh 'centuser@192.168.33.20'", and check in:

  .ssh/authorized_keys

to make sure we haven't added extra keys that you weren't expecting.
```
公開鍵認証を使ったSSHログイン
```bash
$ ssh 192.168.33.20
```
公開鍵ファイルをコピー
```bash
$ scp ~/.ssh/id_rsa.pub 192.168.33.20:publickey
centuser@192.168.33.20's password:
id_rsa.pub                                                                                                                 100%  396     0.4KB/s   00:00
```
192.168.33.20にSSHでログイン
```
$ ssh 192.168.33.20
centuser@192.168.33.20's password:
Last login: Wed Aug 20 08:22:04 2014 from 192.168.33.10
```
authorized_keysに公開鍵を登録
```
$ cat publickey >> ~/.ssh/authorized_keys
```
authorized_keysファイルのパーミッションを変更
```bash
$ chmod 600 ~/.ssh/authorized_keys
```
publickeyファイルを削除
```bash
$ rm publickey
```
### OpenSSHサーバー
#### SSHサーバーの基本設定
|  設定項目 | 説明         |
|:---------------|:-------------|
| Port | SSHで使うポート番号 |
| Protocol | SSHのバージョン(１または２または両方) |
| HostKey | ホストの秘密鍵ファイル |
| PermitRootLogin | rootでもログインを許可するかどうか(yes,no,without-password,forced-commands-onlyのいずれか)|
| RSAAuthentication | SSHバージョン１での公開鍵認証を使用するかどうか(yes,no)|
| PubkeyAuthentication | SSHバージョン２での公開鍵認証を使用するかどうか(yes,no)|
| AuthorizedKeysFile | 公開鍵が格納されるファイル名 |
| PermitEmptyPasswords | 空のパスワードを許可するかどうか (yes,no) |
| PasswordAuthentication | パスワード認証を許可するかどうか(yes,no) |
| AllowUsers | 接続を許可するユーザーのリスト |
| DenyUsers | 接続を拒否するユーザーのリスト |
| LoginGraceTime | ログイン認証の待ち時間(デフォルトは120秒)|
| MaxAuthTries | ログイン認証の最大再試行回数(デフォルトは6秒)|
| UsePAM | PAM認証を利用するかどうか(yes,no)|

設定変更後はSSHサーバーを再起動する
```bash
# service sshd restart
```
TCP Wrapperによるアクセス制御  
/etc/hosts.denyの設定例  
```
ALL: ALL
```
/etc/hosts.allowの設定例  
```
sshd: 192.168.11.2 *.example.com
```
### SSHクライアント
#### SSHリモートログイン
指定したホストにSSHで接続
```bash
$ ssh 192.168.33.20
```
ユーザー名を指定して接続
```
$ ssh centuser@192.168.33.20
```
SSHでコマンドのみ実行
```bash
$ ssh 192.168.33.20 df
```
#### SSHリモートコピー
リモートホストからローカルホストへのファイルコピー（１）
```bash
$ scp /etc/hosts 192.168.33.20:/tmp
```
リモートホストからローカルホストへのファイルコピー（２）
```bash
$ scp 192.168.33.20:/etc/hosts .
```
ユーザー名を指定したファイルコピー
```bash
$ touch data.txt
$ scp data.txt centuser@192.168.33.20:
```
ポート番号を指定したファイルコピー
```bash
$ scp -P 22 /etc/hosts 192.168.33.20:/tmp
```
#### sftp
```bash
$ sftp 192.168.33.20
```
#### ポート転送
SSHポート転送
```bash
$ ssh -f -N centuser@192.168.33.20 -L 10110:192.168.33.20:110
```
#### SSH Agent
ssh-agentの利用を開始
```bash
$ ssh-agent bash
```
パスフレーズを登録
```bash
$ ssh-add
Identity added: /home/centuser/.ssh/id_rsa (/home/centuser/.ssh/id_rsa)
```
秘密鍵の一覧を表示
```bash
$ ssh-add -l
2048 ce:0b:3f:79:82:9f:aa:ad:e2:4e:e8:2d:4b:38:44:ce /home/centuser/.ssh/id_rsa (RSA)
```
#### SSHクライアントの設定
_/etc/ssh/ssh_config_  

|  設定項目 | 説明         |
|:---------------|:-------------|
| Port | ポート番号を指定する |
| Protocol | 利用するSSHプロトコルバージョン |
| PasswordAuthentication |  パスワード認証を使用するかどうか(yes,no)|
| RSAAuthentication | 公開鍵認証を使用するかどうか(yes,no) |
| IdentityFile | 秘密鍵ファイルを指定する |

SSHログイン後に自動実行したいコマンドがある場合は_/etc/ssh/sshrc_または_~/.ssh/rc_ファイルに設定。

# 参照
+ [Linuxサーバーセキュリティ徹底入門](http://www.amazon.co.jp/Linux%E3%82%B5%E3%83%BC%E3%83%90%E3%83%BC%E3%82%BB%E3%82%AD%E3%83%A5%E3%83%AA%E3%83%86%E3%82%A3%E5%BE%B9%E5%BA%95%E5%85%A5%E9%96%80-%E3%83%BC%E3%83%97%E3%83%B3%E3%82%BD%E3%83%BC%E3%82%B9%E3%81%AB%E3%82%88%E3%82%8B%E3%82%B5%E3%83%BC%E3%83%90%E3%83%BC%E9%98%B2%E8%A1%9B%E3%81%AE%E5%9F%BA%E6%9C%AC-%E4%B8%AD%E5%B3%B6-%E8%83%BD%E5%92%8C/dp/4798132381)
+ [Vagrant + シェルスクリプトで簡単プロビジョニング](http://www.ryuzee.com/contents/blog/6667)
+ [Vagrant で複数のVM を立ち上げて、お互いに通信できるようにするには](http://qiita.com/sho7650/items/cf5a586713f0aec86dc0)
