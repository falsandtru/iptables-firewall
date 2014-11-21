# iptables firewall
高機能iptables設定スクリプト。

```
=======================
  PACKET FLOW EXAMPLE
=======================

== config ==
  ROLES=(SSH)
  SSH=(BLOCK_COUNTRY "file{1,2}|TRACK_PROWLER|DROP" LOCAL_COUNTRY FIERWALL IPF "IPS|LOG...|DROP")
  ...
  MAP=("${MAP[@]}" "INPUT -p tcp --dport 60022 -j SSH")
  MAP=("${MAP[@]}" "INPUT -j TRAP_PORTSCAN")
  MAP=("${MAP[@]}" "FORWARD -j TRAP_PORTSCAN")


== apply ==
            INTERNET
        ______ V ______________________________________________________    _______
INPUT  |               | TCP UDP ICMP                                  |  |       |
       |   TCP 60022   |       TRAP_PORTSCAN  ( --> TRACK_PROWLER  )  --->| POLICY|
       |               |                                               |  |       |
       |====== | ======|===============================================|  |_______|
       |====== V ======|===============================================|   _______
Layer1 |                                                               |  |       |
       |                          BLOCK_COUNTRY                       --->|       |
       |______________________________ V ______________________________|  |       |
Layer2 | Rule1         | Rule101       | Rule201       | Rule202       |  |       |
       |     file1    -->    file2    -->TRACK_PROWLER-->    DROP     --->|       |
       |______ V ______|______ V ______|______ V ______|_______________|  | BLOCK |
Layer3 |                                                               |  |       |
       |                          LOCAL_COUNTRY                       --->|       |
       |______________________________ V ______________________________|  |       |
Layer4 |                                                               |  |       |
       |                            FIERWALL  ( --> TRACK_ATTACKER )  --->|       |
       |______________________________ V ______________________________|  |       |
Layer5 |                                                               |  |       |
       |                              IPF  ( ANTI_PROWLER/ATTACKER )  --->|       |
       |______________ V ______________________________________________|  |       |
Layer6 |                               |               |               |  |       |
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
* 設定保存確認およびタイムアウトによるロールバック

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

### LOCAL_COUNTRY_CODE
許可した国以外のIPからのパケットを破棄する。

国の設定を即座に更新するには既存のLOCAL_COUNTRYチェーンを初期化して再構築させる必要がある。

### BLOCK_COUNTRY_CODE
拒否した国のIPからのパケットを破棄する。性能が1/2から1/10程度に劣化するため注意が必要。

国の設定を即座に更新するには既存のLOCAL_COUNTRYチェーンを初期化して再構築させる必要がある。

### FORMAT
ファイルのデータ行を選択、整形する。

### PREPROCESS
事前に実行するコマンドを設定する。

### POSTPROCESS
事後に実行するコマンドを設定する。

### ROLES
任意のロールを作成する。

#### Example

```sh
# TESTロールを作成
ROLES=(TEST)
```

```sh
# TESTロールを適用
MAP=("${MAP[@]}" "INPUT -p tcp --dport 80 -j TEST")
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
File|/etc/iptables/からの相対パスまたは絶対パス。WL_FILENAMEチェーンを生成。ファイル名を識別子とするためファイルは重複しない固有の名前でなければならない。
Composite|他のタイプの組み合わせ。ROLENAME_ITEMNAMEチェーンを生成。先頭の要素名をロール内の識別子とするためこの名前がロール内で重複してはならない。ファイルのみで構成した場合はホワイトリストフィルタとして動作する。

#### Chain

Name|Description
----|-----------
LOCAL_COUNTRY|LOCAL_COUNTRY_CODEで指定した国のIPのみ通過させる。
BLOCK_COUNTRY|BLOCK_COUNTRY_CODEで指定した国のIPを破棄する。
FIREWALL|攻撃および不審なパケットを破棄し、そうでないパケットのみ通過させる。種類に応じてIPを追跡する。
IPF|攻撃者および不審者のIPを遮断する。既知のポート(0-1023)は保護しない。
IPS/IDS|IPS/IDSが設定されている場合にパケットを転送する。設定がない場合はすべて通過する。
TRAP_PORTSCAN|INPUTおよびFORWARDチェーンの末尾に設定することでポートスキャンを補足しIPを追跡する。
TRACK_PROWLER|不審者としてIPを追跡する。
TRACK_ATTACKER|攻撃者としてIPを追跡する。
WL_FILENAME|ファイルタイプのルールから生成されるホワイトリストフィルタ。遮断したIPは不審なIPとして登録される。
ROLENAME_ITEMNAME|複合タイプのルールにより生成されるフィルタ。

※ 動的に生成されるフィルタは名前の重複に注意。

#### Example

```sh
# TESTロールにルールを設定
TEST=(whitelist/private LOCAL_COUNTRY FIREWALL IPF IPS ACCEPT)
# 1. whitelist/private
# ファイルに記述されたIPのみ通過させ、ほかは遮断する。
#
# 2. LOCAL_COUNTRY
# 許可した国のIPのみ通過させ、ほかは遮断する。
#
# 3. FIREWALL
# Firewallを適用し接続を検疫する。
#
# 4. IPF
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

### MAP
設定後に実行するiptablesのコマンドを予約する。

#### Example

```sh
MAP=("${MAP[@]}" "INPUT -p tcp --dport 80 -j PUBLIC")
```

### INTERVAL
地域レジストリから取得するIP割り当ての更新間隔。

### IDSIPS
IDSまたはIPSを使用する場合に設定する。

### SECURE
国別IPフィルタの構築中このフィルタを使用するアクセスをすべて破棄するか、およびロールに設定されたファイルが存在しない場合にエラーを発生させるかを設定する。

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

## Environment
CentOS 6.6

## License
MIT License

## ChangeLog

### 0.6.3

* IPFを攻撃者から照合するよう変更

### 0.6.2

* FW_INTRUDERをIPFに変更

### 0.6.1

* 設定保存確認および自動ロールバック機能を追加
* Compositeタイプのホワイトリスト動作を修正

### 0.6.0

* MAPパラメータを追加
* FORMATパラメータを追加
* Compositeタイプルールを追加
* 国別コードの設定パラメータを変更
* 国別フィルタ名を変更
* 拒否国の適用を手動に変更
* BLACKLIST/WHITELIST機能を削除
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
