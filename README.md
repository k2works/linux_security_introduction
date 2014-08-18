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

## <a name="3">OSのセキュリティ</a>
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
