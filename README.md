# iptables firewall

ルールを設定

```sh
$ sudo mkdir /var/cache/iptables
$ sudo touch /etc/cron.daily/iptables
$ sudo chmod 700 /etc/cron.daily/iptables
$ sudo vi /etc/cron.daily/iptables
$ sudo vi /etc/rsyslog.conf
kern.=debug /var/log/iptables.log
$ sudo sh /etc/cron.daily/iptables
```

BLACKLIST/WHITELIST

```sh
BLACKLIST=/etc/iptables/blacklist
WHITELIST=/etc/iptables/whitelist
RESTRICT=
```

```
# BLACKLIST
1.2.3.0/24
```

```
# WHITELIST
1.2.3.4
```

RESTRICT

```sh
BLACKLIST=
WHITELIST=/etc/iptables/whitelist
RESTRICT=true
```

ipv6を無効化
```sh
$ sudo vi /etc/sysctl.conf
# ipv6 disable
net.ipv6.conf.all.disable_ipv6=1
net.ipv6.conf.default.disable_ipv6=1
$ sudo vi /etc/modprobe.d/disable-ipv6.conf
options ipv6 disable=1
$ sudo vi /etc/hosts
#::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
$ sudo chkconfig ip6tables off

$ sudo /sbin/sysctl -p
$ sudo service network restart
$ sudo reboot

$ ifconfig
$ netstat -an -A inet6
$ lsmod | grep ipv6 # モジュール自体はロードさせる
```

ログローテート
```sh
$ sudo service rsyslog restart
$ sudo vi /etc/logrotate.d/iptables
/var/log/iptables.log {
  rotate 14
  daily
  compress
  missing ok
  notifempty
  
  postrotate
    service rsyslog restart
  endscript
}
```

## ChangeLog

### 0.0.2

* SSHポート自動設定機能が正常に動作しないバグを修正

### 0.0.1

* 公開
