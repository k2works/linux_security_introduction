Linuxセキュリティ入門
===
# 目的
# 前提
| ソフトウェア     | バージョン    | 備考         |
|:---------------|:-------------|:------------|
| OS X           |10.8.5        |             |
|           　　　|        |             |

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
## <a name="5">ネットワークのセキュリティ</a>
## <a name="6">SELinux</a>
## <a name="7">システムログの管理</a>
## <a name="8">セキュリティチェックと侵入探知</a>
## <a name="9">DNSサーバーのセキュリティ</a>
## <a name="10">Webサーバーのセキュリティ</a>
## <a name="11">メールサーバーのセキュリティ</a>
## <a name="12">FTPサーバーのセキュリティ</a>
## <a name="13">SSH</a>

# 参照
+ [Linuxサーバーセキュリティ徹底入門](http://www.amazon.co.jp/Linux%E3%82%B5%E3%83%BC%E3%83%90%E3%83%BC%E3%82%BB%E3%82%AD%E3%83%A5%E3%83%AA%E3%83%86%E3%82%A3%E5%BE%B9%E5%BA%95%E5%85%A5%E9%96%80-%E3%83%BC%E3%83%97%E3%83%B3%E3%82%BD%E3%83%BC%E3%82%B9%E3%81%AB%E3%82%88%E3%82%8B%E3%82%B5%E3%83%BC%E3%83%90%E3%83%BC%E9%98%B2%E8%A1%9B%E3%81%AE%E5%9F%BA%E6%9C%AC-%E4%B8%AD%E5%B3%B6-%E8%83%BD%E5%92%8C/dp/4798132381)
+ [http://www.ryuzee.com/contents/blog/6667](http://www.ryuzee.com/contents/blog/6667)
