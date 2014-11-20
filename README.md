# iptables firewall
高機能iptables設定スクリプト。

```
===================
  SSH PACKET FLOW
===================

== config ==
  ROLES=(SSH)
  SSH=("file{1,2}|TRACK_PROWLER|DROP" COUNTRY_FILTER FIERWALL FW_INTRUDER "IPS|LOG...|DROP")
  ...
  $IPTABLES -A INPUT -p tcp --dport 60022 -j SSH


== implement ==
            INTERNET
        ______ V ______________________________________________________
INPUT  |               | TCP UDP ICMP                                  |
       |   TCP 60022   |       TRAP_PORTSCAN  ( --> TRACK_PROWLER )   --->
       |               |                                               |
       |====== | ======|===============================================|
       |====== V ======|===============================================|   _______
Layer1 | Rule1         | Rule101       | Rule201       | Rule202       |  |       |
       |     file1    -->    file2    -->TRACK_PROWLER-->    DROP     --->|       |
       |______ V ______|______ V ______|______ V ______|_______________|  |       |
Layer2 |                                                               |  |       |
       |                         COUNTRY_FILTER                       --->|       |
       |______________________________ V ______________________________|  | BLOCK |
Layer3 |                                                               |  |       |
       |                            FIERWALL                          --->|       |
       |______________________________ V ______________________________|  |       |
Layer4 |                                                               |  |       |
       |                          FW_INTRUDER  ( -->  TRACK_PROWLER ) --->|       |
       |______________ V ______________________________________________|  |       |
Layer5 |                               |               |               |  |       |
       |            IDS/IPS           -->     LOG     -->     DROP    --->|       |
       |                               |               |               |  |       |
       |============== | ==============================================|  |_______|
       |============== V ==============================================|
SERVICE|                                                               |
       |                      === SSH SERVICE ===                      |
       |_______________________________________________________________|
```

## Feature

* ロールベースコントロール
* マルチレイヤフィルタリング
* ファイアーウォール
* ポートスキャントラップ
* 国別IPフィルタリング
* プリプロセス/ポストプロセスコマンド実行
* 地域レジストリIP割り当ての自動取得/更新/適用

## Usage

```sh
$ mkdir /var/cache/iptables
$ touch /etc/cron.daily/iptables
$ chmod 700 /etc/cron.daily/iptables
$ vi /etc/cron.daily/iptables
> # Paste script.
$ vi /etc/rsyslog.conf
> kern.=debug /var/log/iptables.log
$ bash /etc/cron.daily/iptables
```

## Config

### LOGIN
SSHなどのログインポートを設定。複数設定可。

### INTERVAL
地域レジストリから取得するIP割り当ての更新間隔。

### IDSIPS
IDSまたはIPSを使用する場合に設定する。

### ACCEPT_COUNTRY_CODE
許可した国以外のIPからのパケットを破棄する。

国の設定を即座に更新するには既存のCOUNTRY_FILTERチェーンを初期化して再構築させる必要がある。

### DROP_COUNTRY_CODE
拒否した国のIPからのパケットを破棄する。性能が1/2から1/10程度に劣化するため注意が必要。

国の設定を即座に更新するには既存のCOUNTRY_FILTERチェーンを初期化して再構築させる必要がある。

### TRAP
ポートスキャントラップの使用を切り替える。

### SECURE
国別IPフィルタの構築中このフィルタを使用するアクセスをすべて破棄するか、およびロールに設定されたファイルが存在しない場合にエラーを発生させるかを設定する。

### ROLES
任意のロールを作成する。

#### Example

```sh
# TESTロールを作成
ROLES=(TEST)
```

```sh
# TESTロールを適用
$IPTABLES -A INPUT -p tcp --dport 8080 -j TEST
```

### RULES(LOCAL/CONNECTION/SYSTEM/NETWORK/AUTH/PRIVATE/CUSTOMER/PUBLIC)
既定のロールルール設定。ルールは左から順に適用される。

ファイル、ユーザー定義チェーン、ジャンプターゲットおよびこれらをあらかじめ結合するフォーマットを組み合わせてルールを構築する。

Compositeタイプによる複数のルールを結合した単一(単層)フィルタと、これを含む複数のルールを多段適用する積層構造の両方向でのフィルタ構築が可能。これにより多数のソースファイルとチェーンを組み合わせたフィルタを動的に構築可能。

最後のルールは通過させずパケットの処理を決定しなければならないため最後のルールにはジャンプターゲットを設定することが望ましい。

#### Type

Type|Definition
----|----------
Chain|大文字とアンダースコアの組み合わせによるチェーンの予約または定義済みチェーン。
Target|ジャンプターゲット(ACCEPT/DROP/RETURN/REJECT/LOG/NFQUEUE)。
File|/etc/iptables/からの相対パスまたは絶対パス。WL_FILENAMEを生成。
Composite|他のタイプの組み合わせ。ROLENAME_FILENAMEを生成。

#### Chain

Name|Description
----|-----------
COUNTRY_FILTER|ACCEPT_COUNTRY_CODEで指定した国のIPのみ通過させる。
FIREWALL|不審なパケットを破棄し、そうでないパケットのみ通過させる。
FW_INTRUDER|不審なIPを遮断するオプションファイアウォールフィルタ。既知のポート(0-1023)は保護しない。
IPS/IDS|IPS/IDSが設定されている場合にパケットを転送する。設定がない場合はすべて通過する。
WL_FILENAME|ファイルから生成されるホワイトリストフィルタ。遮断したIPは不審なIPとして登録される。名前の重複に注意。
ROLENAME_FILENAME|複数のチェーンを展開して結合する。名前の重複に注意。

#### Example

```sh
# TESTロールにルールを設定
TEST=(whitelist/private COUNTRY_FILTER FIREWALL FW_INTRUDER IPS ACCEPT)
# 1. whitelist/private
# ファイルに記述されたIPのみ通過させ、ほかは遮断する。
#
# 2. COUNTRY_FILTER
# 許可した国のIPのみ通過させ、ほかは遮断する。
#
# 3. FIREWALL
# Firewallを適用し接続を検疫する。
#
# 4. FW_INTRUDER
# 攻撃行為または不審行為のあったIPを遮断する。
#
# 5. IPS
# 指定のパケットをIPSへ渡し処理を終える。
#
# 6. ACCEPT
# 渡されなかった残りのパケットをすべて許可し処理を終える。
#
```

ルールにファイルを指定した場合、ファイルに記述されたIPからホワイトリストフィルタを生成し適用する。
フィルタはファイル名で識別されるためファイル名を重複させない必要がある。

```
# whitelist/auth
# プロバイダなどで制限
# http://www.tcpiputils.com/
1.2.3.0/24
```

外部で作成されたチェーンを使用する場合はプリプロセスでチェーンを作成する必要がある。

```
TEST=(CUSTOM_FILTER)
...
......
PREPROCESS="sh /etc/iptables/script/preprocess.sh"
```

```
#!/bin/sh
# /etc/iptables/preprocess.sh
iptables -N CUSTOM_FILTER
```

### FORMAT
ファイルのデータを選択、整形する。

### PREPROCESS
事前に実行するコマンドを設定する。

### POSTPROCESS
事後に実行するコマンドを設定する。

### NAME SERVER
自動設定。

### NTP SERVER
自動設定。

### BLACKLIST
グローバルブラックリスト。一致するIPを破棄する。/etc/iptables/からの相対パスまたは絶対パスで指定。

```sh
BLACKLIST=blacklist/global
```

```
# BLACKLIST
1.2.3.0/24
```

### WHITELIST
グローバルホワイトリスト。一致するIPをBLACKLISTから除外する。/etc/iptables/からの相対パスまたは絶対パスで指定。

```sh
WHITELIST=whitelist/global
```

```
# WHITELIST
1.2.3.1
```

## IPv6 disable
無効化せず放置した場合、IPv6のインターフェイスが攻撃し放題となる。

```sh
$ vi /etc/sysctl.conf
# ipv6 disable
net.ipv6.conf.all.disable_ipv6=1
net.ipv6.conf.default.disable_ipv6=1
$ vi /etc/modprobe.d/disable-ipv6.conf
options ipv6 disable=1
$ vi /etc/hosts
#::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
$ chkconfig ip6tables off

$ /sbin/sysctl -p
$ service network restart
$ reboot

$ ifconfig
$ netstat -an -A inet6
$ lsmod | grep ipv6 # モジュール自体はロードさせる
```

## Logrotate

```sh
$ service rsyslog restart
$ vi /etc/logrotate.d/iptables
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

## License
MIT License

## ChangeLog

### 0.5.3

* MAPパラメータを追加
* FORMATパラメータを追加
* Compositeタイプルールを追加
* ファイルから生成するフィルタのファイル名部分を大文字に変換するよう変更
* Firewallをリファクタリング
* 言語をshellからbashに変更

### 0.5.2

* リストファイルの相対パス指定に対応
* FW_STEALTHSCANをFIREWALLフィルタから除外
* IP追跡処理を改善

### 0.5.1

* IP追跡処理を改善

### 0.5.0

* Firewallをモジュールごとに利用できるよう改善
* ロールの設定を変更
* FW_BASICフィルタを追加
* FW_SPYをFW_INTRUDERに変更
* FW_INTRUDERをFIREWALLフィルタから除外

### 0.4.1

* ロールとルールの定義方法を配列に変更

### 0.4.0

* 仕様を刷新
* サービスへのフィルタ設定をロールベースに変更
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
