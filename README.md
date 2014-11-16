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

### LOGIN
SSHなどのログインポートを設定。複数設定可。

### INTERVAL
IPの更新間隔。

### ACCEPT_COUNTRY_CODE
許可した国以外のIPからのパケットを破棄する。

国の設定を即座に更新するには既存のCOUNTRY_FILTERチェーンを初期化して再構築させる必要がある。

### DROP_COUNTRY_CODE
拒否した国のIPからのパケットを破棄する。性能が1/2から1/10程度に劣化するため注意が必要。

国の設定を即座に更新するには既存のCOUNTRY_FILTERチェーンを初期化して再構築させる必要がある。

### SECURE
国別IPフィルタの構築中このフィルタを使用するアクセスをすべて遮断するか、およびロールに設定されたファイルが存在しない場合にエラーを発生させるかを設定する。

### ROLE
各サービスへのアクセスフィルタをロールにより管理する。

### PREPROCESS
事前に実行するコマンドを設定する。

### POSTPROCESS
事後に実行するコマンドを設定する。

### NAME SERVER
自動設定。

### NTP SERVER
自動設定。

## チェーン

### 優先度

0. WHITELIST
0. BLACKLIST
0. COUNTRY_FILTER
0. FIREWALL(Firewall, IPS/IDS)

### ROLE
任意のロールを作成し任意のフィルタを設定する。フィルタはチェーンで指定およびファイルで生成する。

```
# /etc/iptables/systemlist
1.2.3.0/24
```

```
# /etc/iptables/adminlist
1.2.3.1
```

http://www.tcpiputils.com/

#### ROLES
任意のロールを作成する。ロール名はすべて大文字でなければならない。

#### LOCAL/KEEP/SYSTEM/NETWORK/AUTH/PRIVATE/CUSTOMER/PUBLICK
既定のロール。

### BLACKLIST/WHITELIST
BLACKLISTにより有害なIPを早期にフィルタすることでiptablesの負荷を軽減する。

#### BLACKLIST
一致するIPをDROPする。

#### WHITELIST
一致するIPをBLACKLISTから除外する。

```sh
BLACKLIST=/etc/iptables/blacklist
WHITELIST=/etc/iptables/whitelist
```

```
# BLACKLIST
1.2.3.0/24
```

```
# WHITELIST
1.2.3.1
```

### FIREWALL
Firewall機能を持つ。設定によりIPS/IDSへ処理を引き渡す。

## IPv6の無効化
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

### 0.4.0

* 仕様を刷新
* サービスへのフィルタ設定をロールレベルに変更
* PREPROCESS機能を追加
* POSTPROCESS機能を追加
* Ingress攻撃対策を削除
* ログ記録の制限を緩和

### 0.3.0

* GRAYLISTの挙動を改善

### 0.2.3

* Firewallの適用を通信方向に基づいて最適化
* CentOS7のNIC名に対応

### 0.2.2

* CIDRへの変換時の繰り上がり対応を削除(不要な対応)

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
