# iptables firewall

## 使用方法

```sh
$ sudo mkdir /var/cache/iptables
$ sudo touch /etc/cron.daily/iptables
$ sudo chmod 700 /etc/cron.daily/iptables
$ sudo vi /etc/cron.daily/iptables
$ sudo vi /etc/rsyslog.conf
kern.=debug /var/log/iptables.log
$ sudo sh /etc/cron.daily/iptables
```

## 設定

### SECURE
有効にした場合、設定完了まですべての接続を切断、拒否する。無効でも新規の接続はすべて拒否する。初期値では無効。

```sh
SECURE=true
```

## チェーン

### 優先度

0. WHITELIST
0. GRAYLIST
0. BLACKLIST
0. BLACKLIST_COUNTRY
0. COUNTRY_FILTER
0. FIREWALL(Firewall, IPS/IDS)

### FIREWALL
Firewall機能を持つ。設定によりIPS/IDSへ処理を引き渡す。

### COUNTRY_FILTER
許可した国のIPからのパケットをFIREWALLへ送り、それ以外のパケットは破棄する。

国の設定を即座に更新するには既存のCOUNTRY_FILTERを初期化して再構築させる必要がある。

### BLACKLIST_COUNTRY
拒否した国のIPからのパケットを破棄する。性能が1/2から1/10程度に劣化するため注意が必要。

国の設定を即座に更新するには既存のCOUNTRY_FILTERを初期化して再構築させる必要がある。

### BLACKLIST
一致するIPをDROPする。

有害なIPを早期にフィルタすることでiptablesの負荷を軽減する。

```sh
BLACKLIST=/etc/iptables/blacklist
```

```
# BLACKLIST
1.2.3.0/24
```

### GRAYLIST
一致するIPをFIREWALLへ転送する。

BLACKLIST_COUNTRYとCOUTORY_FILTERによるフィルタを免除する。

```sh
GRAYLIST=/etc/iptables/graylist
```

```
# GRAYLIST
1.2.3.3
```

### WHITELIST
一致するIPをACCEPTする。

WHITELISTを設定した場合、WHITELISTに一致しないすべてのIPを遮断する。

```sh
WHITELIST=/etc/iptables/whitelist
```

```
# WHITELIST
1.2.3.4
```

## ipv6の無効化
無効化しない場合、v6のインターフェイスはノーガードで晒される。

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

## ログローテート

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

### 0.2.2

* CIDRへの変換時の繰り上がり対応を削除

### 0.2.1

* CIDRへの変換時に繰り上がり処理がされない潜在的バグを修正
* チェーン再構築の要否判定が正常に動作しないバグを修正

### 0.2.0

* OUTPUTチェーンのポリシーをDROPに変更
* CIDRへの変換時にサブネットマスクが分割されないバグを修正(初期設定では影響なし)
* IPリストの適用を高速化
* グレーリスト機能を追加
* SECUREモードを追加
* STRICTモードを削除

### 0.1.5

* WHITELISTが正常に動作しないバグを修正
* テーブル名が適切に設定されていないバグを修正
* ブルートフォース攻撃対策でNG処理がされていないバグを修正
* ログ記録のバースト制限を緩和

### 0.1.4

* IPS自動設定が正常に動作しないバグを修正

### 0.1.3

* IPS自動設定が正常に動作しないバグを修正

### 0.1.2

* 誤作動回避のため不正パケット発信元の追跡を取り消し

### 0.1.1

* アンチステルススキャンがkeep-aliveで誤作動する場合があるバグを修正

### 0.1.0

* IPS自動設定機能を追加
* 厳格モード時に国別IP制限を行わないよう修正

### 0.0.2

* SSHポート自動設定機能が正常に動作しないバグを修正

### 0.0.1

* 公開
