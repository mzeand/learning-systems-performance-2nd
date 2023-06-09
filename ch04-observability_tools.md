# 第4章 可観測性ツール

参）👩‍💻 オブザーバビリティとは？
https://www.splunk.com/ja_jp/data-insider/what-is-observability.html
> オブザーバビリティ(可観測性)とは、システムの出力を調査することによって内部の状態を測定する能力を指します。出力からの情報すなわちセンサーデータのみを使用して現在の状態を推定できるシステムは「オブザーバビリティがある」とみなされます。

Observabilityと英語では表記し、Observe（観察する）とAbility（能力）を組み合わせた意味の単語です。この言葉は最近の流行語ではなく、制御理論(自己調整システムについて説明し理解するための理論)に関連して数十年前に提唱された用語に由来します。

---
この章での学習目標は次の通り。
- 静的パフォーマンスチューニングツールとクライシスツールを見分けられるようにする。
  - クライシスツール：緊急用ツール、アウトドアや日常の作業で使えるツールをコンパクトに内蔵したツールナイフ。
    - ![クライシスツール](./images/ch04/crisis_tool.png)
- ツールのタイプとそのオーバーヘッドを理解する: カウンタ、プロファイリング、トレーシング
- 可観測性データのソースについて学ぶ: /proc、/sys、トレースポイント、kprobe、uprobe、USDT、
PMCなど
- 統計量のアーカイブを作るsar(1) の設定方法を学ぶ。

## 4.1 取り上げるツール
![図4.1 Linuxワークロード可観測性ツール](./images/ch04/figure-4-1.png)
- 大半はCPU、メモリ、ディスクなどの特定のデバイスにターゲットを絞ったもの
- そのほかに、さまざまなものを解析できるperf、Ftrace、BCC、bpftrace などの多用途ツール

### 4.1.1 静的パフォーマンスツール
- アクティブなワークロードを抱えたシステムではなく、休止状態のシステムの属性を解析するタイプのもの。
- 構成ミスによって起こるパフォーマンス障害を調査する。
- 構成やコンポーネントの問題をチェックするときに図4.2の使う。
![図4.2 Linuxパフォーマンスチューニングツール](./images/ch04/figure-4-2.png)
### 4.1.2 クライシスツール
- 表4.1 はクライシスツールとして推奨されているインストールパッケージ、またはソースリポジトリ
![表4.1 Linuxクライシスツールパッケージ](./images/ch04/table-4-1.png)
- デフォルトのLinux ディストリビューションには、procpsとutil-linux がインストールされているだけという場合があるので、そういうときにはその他すべてのパッケージをインストールしなければならない。
- コンテナ環境では、システムへのフルアクセスを持ち、すべてのツールがインストールされている特権的なデバッグコンテナを作るとよい場合がある。必要なときにこのコンテナのイメージをコンテナホスト
にインストールし、デプロイすればよい。
- ツールパッケージを追加しただけでは不十分な場合があるので必要な設定も行わなければならない。
  
## 4.2 ツールタイプ
- 可観測性ツールの分類

![図4.3 可観測性ツールのタイプ](./images/ch04/figure-4-3.png)

- いくつかの一部のツールは、複数の四半球に収まる。
  - 例えば、top(1)は、システム全体の集計情報を持っており、コマンドにオプション（-p PID）をつけるとは特定のプロセスだけを対象にするようにフィルタリングできる。
- イベントベースツール
  - プロファイラ：イベントについての一連のスナップショットを取ってアクティビティを観測し、ターゲットのおおよその姿を描く。
  - トレーサー：対象のイベントをすべてインストルメンテーションし、たとえばカスタマイズされたカウンタを生成するなどの処理を加えることもある。
  
### 4.2.1 固定カウンタ
- カーネルは、システム統計を提供するためにさまざまなカウンタを維持している。
- 通常のカウンタは、イベントが発生するとインクリメントされる符号なし整数として実装されている。
  - 受信したネットワークパケットの数、発行したディスクI/Oの数、発生した割り込みの数のカウンタ
- カーネルは、一般に2個（イベントの数とイベント処理にかかった時間の合計）の累積値カウンタを管理するという方法をとる。
- カウンタはデフォルトで有効になっており、カーネルによって継続的にメンテナンスされているので、パフォーマンス的には「タダ」で使えると考えてよい。

#### 4.2.1.1 システム全体
- カーネルカウンタを使ってシステムソフトウェアやハードウェアリソースのコンテキストでシステム全体のアクティビティを解析する。
  - vmstat(8): 仮想メモリ、物理メモリの統計量。システム全体
  - mpstat(1): CPUごとの使用率。
  - iostat(1): ディスクごとのI/Oの使用状況。ブロックデバイスインターフェイスから報告される。
  - nstat(8): TCP/IPスタックの統計量。
  - sar(1): さまざまな統計量。履歴情報の報告のために情報をアーカイブ化することもできる。
- 多くのツールは、オプションでインターバル（interval、間隔）と個数（count）を指定できるという慣習的なルールに従っている。

```
$ vmstat 1 3
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
r b swpd free buff cache si so bi bo in cs us sy id wa st
4 0 1446428 662012 142100 5644676 1 4 28 152 33 1 29 8 63 0 0
4 0 1446428 665988 142116 5642272 0 0 0 284 4957 4969 51 0 48 0 0
4 0 1446428 685116 142116 5623676 0 0 0 0 4488 5507 52 0 48 0 0
```


#### 4.2.1.2 プロセスごと
- プロセス指向で、カーネルが個々のプロセスのためにメンテナンスしているカウンタを使っている。
- Linux ツールとしては次のようなものがある。
  - ps(1): プロセスのステータス、メモリやCPUの使用率などのさまざまな統計量を表示する。
  - top(1): CPUの使用率、その他の統計量順に上位のプロセスを表示する。
  - pmap(1): プロセスのメモリセグメントを利用統計付きでリストアップする。

- 一般に、/proc ファイルシステムから統計量を読み出している。

### 4.2.2 プロファイリング
- プロファイリングは、ターゲットの挙動のサンプル、スナップショットを集めてターゲットの特徴を表す。
  - 例えば、タイマーに基づいて命令ポインタやスタックトレースを __サンプリング__ し、CPUサイクルを消費しているコードパスの特徴を浮かび上がらせる。
- サンプルは、通常すべてのCPUを通じて100Hz（サイクル/秒）などの決まった頻度で1分などの短い期間に収集される。
- プロファイリングツールは、プロファイラ: profiler と呼ぶ。
- プロファイラは、ターゲットのアクティビティとサンプリングが同じ歩調にならないように、100Hz ではなく99Hz を使うことが多い。歩調が揃うと、数えた数が多過ぎたり少な過ぎたりすることがある。
- 固定カウンタとは異なり、一般に必要なときに限り有効にされる。
- 収集のためにCPUオーバーヘッド、格納のためにストレージオーバーヘッドがかかるため。

#### 4.2.2.1 システム全体
- システム全体を対象とするLinuxプロファイラには次のものがある。
  - perf(1): Linux の標準プロファイラ
  - profile(8): BCCリポジトリに含まれるBPFベースのCPUプロファイラ
  - Intel VTune Amplifier XE: Linux とWindows のプロファイリング
  
#### 4.2.2.2 プロセスごと
- プロセス指向のLinux プロファイラには次のものがある。
  - gprof(1): GNUプロファイリングツール。コンパイラが追加した（たとえばgcc -pg）プロファイリング情報を分析する。
  - cachegrind: valgrind ツールキットに含まれている。ハードウェアキャッシュの使用状況をプロファイリングし、kcachegrind を使ってビジュアライズできる。
  - Java Flight Recorder (JFR): プログラミング言語は、言語コンテキストを調査できる専用プロファイラを持っていることがよくある。Java のJFRはその例である。

- プロファイリングツールの詳細は、「6 章CPU」と「13 章perf」を参照
  
### 4.2.3 トレーシング
- トレーシングは __発生したイベントをすべてインストルメンテーション__ し、あとで分析するためにイベントごとの詳細情報を格納したり、集計を生成したりすることができる。
- トレーシングはプロファイリングよりもCPUとストレージのオーバーヘッドが高くなるため、ターゲットの実行速度を遅らせることがある。
- トレーシングも必要なときしか使われない。
- ロギング（logging）は、デフォルトで有効にされる頻度の低いトレーシングと考えることがで
きる。

#### 4.2.3.1 システム全体
- カーネルのトレーシング機能を使って、システムソフトウェアやハードウェアリソースの __コンテキスト__ でシステム全体のアクティビティを解析する。
- Linux トレーサーとしては次のようなものがある。
  - tcpdump(8): ネットワークパケットのトレーシング
  - biosnoop(8): ブロックI/Oのトレーシング
  - execsnoop(8): 新しいプロセスのトレーシング
  - perf(1): Linux の標準プロファイラだが、イベントトレーシングにも使える。
  - perf trace: システム全体のシステムコールをトレースするperf の特別なサブコマンド
  - Ftrace: Linux の組み込みトレーサー
  - BCC: BPFベースのトレーシングライブラリとツールキット
  - bpftrace: BPFベースのトレーサー（bpftrace(8)）とツールキット

#### 4.2.3.2 プロセスごと
- Linux トレーサーとしては次のようなものがある。
  - strace(1): システムコールのトレーシング
  - gdb(1): ソースレベルデバッガ
- デバッガはイベントごとのデータを解析できるが、そのためにはターゲットの実行を停止、開始しなければならない。オーバーヘッドが莫大なものになることがあるので、本番環境での使用には適していない。

### 4.2.4 モニタリング
#### 4.2.4.1 sar(1)
- sar(1)（System Activity Reporter）
- 単一OSホストのモニタリングで古くから使われている。AT&T Unix に起源を持つ。
- カウンタベースで、cronによってスケジューリングされた時刻に実行されるエージェントがシステム全体のカウンタの状態を記録していく。
```
# sar
Linux 4.15.0-66-generic (bgregg) 12/21/2019 _x86_64_ (8 CPU)
12:00:01 AM CPU %user %nice %system %iowait %steal %idle
12:05:01 AM all 3.34 0.00 0.95 0.04 0.00 95.66
12:10:01 AM all 2.93 0.00 0.87 0.04 0.00 96.16
12:15:01 AM all 3.05 0.00 1.38 0.18 0.00 95.40
12:20:01 AM all 3.02 0.00 0.88 0.03 0.00 96.06
[...]
Average: all 0.00 0.00 0.00 0.00 0.00 0.00
```
- 数十種もの統計量を記録し、CPU、メモリ、ディスク、ネットワーキング、割り込み、消費電力などについての知見を提供する。
- ネットワークに指標を送るサードパーティのモニタリングプロダクトも多数ある。
#### 4.2.4.2 SNMP
- SNMP（Simple Network Management Protocol）
- ネットワークのモニタリング
- ほとんどの環境はカスタムエージェントベースのモニタリングに切り替わっている。
  - SNMPエージェント
    - ネットワーク機器がSNMPにより外部の機器と通信し、自身の状態を報告したり、管理や操作を受け付ける
#### 4.2.4.3 エージェント
- 最近のモニタリングソフトウェアでは、カーネルとアプリケーションの指標を記録するためにエージェント（agent、エクスポーター: exporter とかプラグイン: plugin と呼ばれることもある）を実行する。
- システムカウンタだけでは入手できない詳細なアプリケーション要求の指標を提供できる。
- Linux 用のモニタリングソフトウェアとエージェントには次のものがある。
  - Performance Co-Pilot（PCP）: PMDA（Performance Metric Domain Agents）と呼ばれる数十種のエージェントをサポートし、そのなかにはBPFベースの指標も含まれる。
  - Prometheus: データベース、ハードウェア、メッセージング、ストレージ、HTTP、API、ロギングなどのための数十種のエクスポーターをサポートする。
  - collectd: 数十種のプラグインをサポートする
- モニタリングアーキテクチャの例
  
![図4.4 サンプルモニタリングアーキテクチャ](./images/ch04/figure-4-4.png)
- 各サーバから指標がデータベースに送られ、クライアントUIでグラフとしてダッシュボードに表示される。
- モニタリングデータベースサーバーとしてはGraphite Carbon、モニタリングウェブサーバー/ダッシュボードとしてはGrafana などがある。
- モニタリング製品は何十種もあるが、カーネルのカウンタに基づくシステム指標を学習しておくと、モニタリング製品を理解するために役に立つ。
## 4.3 可観測性ツールの情報ソース
- 取り上げるソース

![表4.2 Linuxの可観測性ツールの情報ソース](./images/ch04/table-4-2.png)

- イベント、グループ名を使ってこれらのトレー
シングソースが何を対象としているかまとめたも

![図4.5 Linuxのトレーシングソース](./images/ch04/figure-4-5.png)

### 4.3.1 /proc
- カーネル統計に対するファイルシステムインターフェイス。
- いくつかのディレクトリが含まれており、各ディレクトリには対象プロセスのプロセスIDに基づく名前が付けられている。
  - statはシステムの全体的な動作状況を表示
- カーネルによって動的に作成される。
- ストレージデバイスの裏付けはない（インメモリで実行されている）。
- ほとんどが読み出し専用で、可観測性ツールのために統計量を提供している。
- 一部のファイルは、プロセスやカーネルの動作をコントロールするために書き込み可能になっている。
  - tcp_rmem など、設定変更が可能
- 大半の/proc ファイルの読み出しにかかるオーバーヘッドは無視できる。
- ただし、ページテーブルをたどっていかなければならないメモリマップ関連のファイルは例外。

#### 4.3.1.1 プロセスごとの統計量
- /proc には、プロセスごとの統計量を格納するさまざまなファイルが含まれている
```
$ ls -F /proc/18733
arch_status environ mountinfo personality statm
attr/ exe@ mounts projid_map status
autogroup fd/
  :
```

- プロセスごとのパフォーマンス計測に関連するファイル
  - limits: 有効になっているリソース制限
  - maps: メモリマップ
  - sched: CPUスケジューラのさまざまな統計量
  - schedstat: CPU の実行時間、レイテンシ、タイムスライス
  - smaps: マッピングされているメモリ領域とその使用状況
  - stat: CPUとメモリの全体的な使用状況を含むプロセスのステータスと統計量
  - statm: ページ単位でのメモリの使用状況
  - status: ラベル付きのstat、statm情報
  - fd: ファイルディスクリプタシンボリックリンクのディレクトリ（fdinfo）
  - cgroup: cgroupのメンバー情報
  - task: タスク（スレッド）ごとの統計量のディレクトリ

- top(1) は、システム上のすべてのアクティブプロセスでこれらを読み出しているので、これを実行するためにかかるオーバーヘッドが見てわかるほどになることがある

#### 4.3.1.2 システム全体の統計量
- システム全体の情報も含むように/proc を拡張しており、次のようなファイルとディレクトリが追加されている
```
$ cd /proc; ls -Fd [a-z]*
acpi/ dma kallsyms mdstat schedstat thread-self@
buddyinfo driver/ kcore meminfo scsi/ timer_list
bus/ execdomains keys misc self@ tty/
cgroups fb
 :
```
- システム全体のパフォーマンス計測に関連するファイル
  - cpuinfo: すべての仮想CPU、モデル名、クロックスピード、キャッシュサイズなどの物理プロセッサ情報
  - diskstats: すべてのディスクデバイスのディスクI/O統計
  - interrupts: CPUごとの割り込み数
  - loadavg: 負荷の平均
  - meminfo: システムメモリの使用状況
  - net/dev: ネットワークインターフェイスの統計量
  - net/netstat: システム全体でのネットワーク統計
  - net/tcp: アクティブなTCPソケットの情報
  - pressure/: PSI（Pressure stall information）ファイル
  - schedstat: システム全体のCPUスケジューラ統計
  - self: 現在のプロセスIDディレクトリへのシンボリックリンク
  - slabinfo: カーネルスラブアロケータのキャッシュ統計
  - stat: カーネルとシステムリソース統計のサマリーで、CPU、ディスク、ページング、スワップ、プロセスの情報が含まれる
  - zoneinfo: メモリゾーン情報

##### 4.3.1.2.1 CPU統計の精度
- /proc/stat ファイルはシステム全体でのCPU 使用率の情報を提供している
- この情報の精度はカーネルの構成によって変わる。
- 解像度の高いカウンタ（VIRT_CPU_ACCOUNTING_NATIVEVIRT_CPU_ACCOUTING_GEN）
を使い、より正確なIRQ時間を要求するオプション（IRQ_TIME_ACCOUNTING）を指定すれば、パフォーマンスにわずかな影響が及ぶが精度を上げられる。

#### 4.3.1.3 ファイルの内容
- /proc ファイルは通常テキスト形式
  - カーネルには統計量をテキストにエンコードするというオーバーヘッド、ユーザーランドツールにはテキストをパースするというオーバーヘッドがかかっている

### 4.3.2 /sys
- 👩‍💻 /proc/sys から分離して拡張された
  - デバイスドライバ関連のプロセス統計
  - 参: sysfs https://docs.oracle.com/cd/E39368_01/porting/ch06s09.html
- /sys ファイルシステムは、一般に、数万の統計量を格納する読み出し専用ファイルと、カーネルの状態を変更するための書き込み可能ファイルを持っている。

- 次に示すのは、CPU 0 の/sys ファイルリストである（一部だけに省略してある）
```
$ find /sys/devices/system/cpu/cpu0 -type f
/sys/devices/system/cpu/cpu0/uevent
/sys/devices/system/cpu/cpu0/hotplug/target
/sys/devices/system/cpu/cpu0/hotplug/state
/sys/devices/system/cpu/cpu0/hotplug/fail
/sys/devices/system/cpu/cpu0/crash_notes_size
/sys/devices/system/cpu/cpu0/power/runtime_active_time
/sys/devices/system/cpu/cpu0/power/runtime_active_kids
/sys/devices/system/cpu/cpu0/power/pm_qos_resume_latency_us
/sys/devices/system/cpu/cpu0/power/runtime_usage
[...]
```
### 4.3.3 遅延アカウンティング
- 遅延アカウンティング（delay accouting）は、プロセスやスレッドグループが、どれだけカーネルに待たされたかを測定できる仕組み。
- CONFIG_TASK_DELAY_ACCT オプションを指定したLinux システムは、次の項目についてタスクごとの時間を管理する。
  - スケジューラレイテンシ（scheduler latency）: on-CPUになるのを待つ時間
  - ブロックI/O（block I/O）: ブロックI/Oが完了するのを待つ時間
  - スワッピング（swapping）: ページングを待つ時間（メモリの圧迫度）
  - メモリの回収（memory reclaim）: メモリ回収ルーチンが実行されるのを待つ時間
- スケジューラレイテンシ統計は、schedstats（/proc に含まれている）から情報を得ている
- これらの統計量は、taskstats を使うユーザーレベルツールで読むことができる
- netlink ベースのインターフェイスである。カーネルのソースコードに次のものが含まれている
  - Documentation/accounting/delay-accounting.txt: ドキュメント
  - tools/accounting/getdelays.c: プログラム例
    - https://gitlab.sdu.dk/sdurobotics/linux-kernels/kernel/-/blob/v4.13.13-Ubuntu-4.13.0-32.35/tools/accounting/getdelays.c
  - ![getdelays出力](./images/ch04/4-3-3-getdelays.png)

- 👩‍💻 CONFIG_TASK_DELAY_ACCTを有効にする方法
  - カーネルのソースをダウンロードして設定を有効にした後、再ビルドする必要がある
    - https://chat.openai.com/chat/1c3611f6-d2d0-4157-b8a1-6c9d545f7a05
  - CONFIG_TASKSTATS も y に設定しておく必要がある
  
### 4.3.4 netlink
- カーネル情報をフェッチするための特殊なソケットアドレスファミリ（AF_NETLINK）
  - ソケットベースのプログラム
- /procよりも効率が良く通知もサポートされる
- straceを使って ss（socket statistics: TCP ポートや UDP ポートの通信状態を確認するためのコマンド）がカーネルのどこから情報を得ているかを調べる
  - strace: プログラムから呼ばれるシステムコールを確認する
  - ![getdelays出力](./images/ch04/4-3-4-strace-ss.png)
  - NETLINK_SOCK_DIAG グループを指定してAF_NETLINK ソケットをオープンしている
- netlink families 主なもの
  - NETLINK_ROUTE: ルート情報（/proc/net/route もある）
  - NETLINK_SOCK_DIAG: ソケット情報
  - NETLINK_SELINUX: SELinux のイベント通知
  - NETLINK_AUDIT: 監査（セキュリティ）
  - NETLINK_SCSITRANSPORT: SCSI トランスポート
  - NETLINK_CRYPTO: カーネル暗号情報

### 4.3.5 トレースポイント
- カーネルコードの論理的な位置にハードコードされたインストルメンテーションポイント
- トレーシングのためのイベントソース
- トレースポイントインフラストラクチャは、マシュー・デスノイヤーズが開発して2009 年のLinux 2.6.32 で使えるようになった
- Stable API (過去に1回 変更されたことがあるが・・)
  - ロバスト（堅牢）なツールを開発できる
- サマリー統計の域を超えてカーネルの動作を深く知るための高度なトレーシングツールを作る基礎となるもので、パフォーマンス分析の重要なリソース

#### 4.3.5.1 トレースポイントの例
- 利用できるトレースポイントは、perf listコマンドで調べられる
  - ![getdelays出力](./images/ch04/4-3-5-1-tracepoint.png)
- イベントの発生時だけでなく、イベントのコンテキストデータも表示できる
- トレースポイントは、実際にはカーネルソースコードに配置されたトレーシング関数である（トレーシングフック: tracing hook とも呼ばれる）
- たとえば、trace_sched_wakeup( ) というトレースポイントがあり、from kernel/sched/core.c にこの関数の呼び出しが含まれている
  - https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/kernel/sched/core.c?h=v6.2.10#n4145
- 実際には TRACE_EVENT マクロで定義されたトレースイベント
  - https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/include/trace/events/sched.h?h=v6.2.10
  - `TRACE_EVENT` https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/include/linux/tracepoint.h?h=v6.2.10#n549
  - ` __DECLARE_TRACE` https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/include/linux/tracepoint.h?h=v6.2.10#n240

#### 4.3.5.2 トレースポイントの引数と書式文字列
- 個々のトレースポイントは、イベントのコンテキスト情報であるイベント引数を記述する書式文字列を持つ。
- 書式文字列の構造は、/sys/kernel/debug/tracing/events ディレクトリの下にあるformat ファイルで
見ることができる。最後の行が書式文字列と引数を示している。
  - ![block_rq_issue](./images/ch04/4-3-5-2-block_rq_issue.png)
- 👩‍💻 /sys/kernel/debug/tracing/ にトレース情報が出力される
- perf や bpftrace　のトレーサでを使うことでサイズや表示形式を指定することもできる

#### 4.3.5.3 トレースポイントのインターフェイス
- トレーシングツールは、tracefs（一般に、/sys/kernel/debug/tracing にマウントされる）のトレースイベントファイルかperf_event_open(2) システムコールを介してトレースポイントを利用できる

```
# strace -e openat ~/Git/perf-tools/bin/iosnoop
chdir("/sys/kernel/debug/tracing") = 0
openat(AT_FDCWD, "/var/tmp/.ftrace-lock", O_WRONLY|O_CREAT|O_TRUNC, 0666) = 3
[...]
openat(AT_FDCWD, "events/block/block_rq_issue/enable", O_WRONLY|O_CREAT|O_TRUNC,
0666) = 3
openat(AT_FDCWD, "events/block/block_rq_complete/enable", O_WRONLY|O_CREAT|O_TRUNC,
0666) = 3
[...]
```

#### 4.3.5.4 トレースポイントのオーバーヘッド
- トレースポイントを使うときにはオーバーヘッドのことを考える必要がある
  - トレースポイント, トレーシングツール ともにCPUオーバヘッドが可kる
  - 本番アプリケーションに影響が出るかどうかはイベント発生頻度とCPUの数によって決まる
  - 今日の一般的はシステム（CPU 4 -128個）では、毎秒1万イベントまでのオーバーヘッドは無視できる
    - スケジューライベントは毎秒10万回を軽く超えることがある
  - 2018年のLinux 4.7 で追加された rawトレースポイントという新しいタイプのトレースポイントは、安定したトレースポイント引数を作るためのコストを取り除き、オーバーヘッドを削減している
  - トレースポイントを無効化するときのオーバーヘッドもある
    - 非常に小さいが、考慮する必要がある

#### 4.3.5.5 トレースポイントのドキュメント
- カーネルソースのDocumentation/trace/tracepoints.rst でドキュメントされている

### 4.3.6 kprobe
- 動的インストルメンテーションをするトレーサーのためのLinuxカーネルイベントソース
- あらゆるカーネル関数、命令をトレースできるもので、2004 年のLinux 2.6.9 で使えるようになった
- kprobe は、カーネルのバージョンによって変更される可能性のあるカーネル関数と引数を表に出してしまうので、不安定なAPIだと考えられている
- 標準的な方法は、実行中のカーネルコードの命令テキストを書き換えて、必要な命令を挿入するというもの
- kprobe は、本番稼働しているカーネルの動作についてほぼ無限の情報を引き出すための最後の手段
- ![表4.3 kprobeとトレースポイントの比較](./images/ch04/table-4-3.png)

#### 4.3.6.1 kprobeの例
- 👩‍💻 bpftrace などのトレーシングツールの内部で利用されている
- 次のbpftrace コマンドは、カーネル関数のdo_nanosleep() をインストルメンテーションし、on-CPUのプロセスを表示する。
```
# bpftrace -e 'kprobe:do_nanosleep { printf("sleep by: %s\n", comm); }'
Attaching 1 probe...
sleep by: mysqld
sleep by: mysqld
sleep by: sleep
^C
#
```
- この出力には、myslqd という名前のプロセスによる2 回のスリープと、sleep（おそらく/bin/sleep）による1回のスリープが含まれている。do_nanosleep( ) に対するkprobeイベントはbpftrace プログラムが実行を開始したときに作られ、bpftrace が強制終了（Ctrl-C）されたときに削除される。

#### 4.3.6.2 kprobeによる引数のトレース
- kprobe はカーネル関数呼び出しをトレースできる
```
# bpftrace -e 'kprobe:do_nanosleep { printf("mode: %d\n", arg1); }'
Attaching 1 probe...
mode: 1
mode: 1
mode: 1
[...]
```

#### 4.3.6.3 kretprobe 
- カーネル関数からのリターンと戻り値は、kretprobe（kernel return probe を意味する）でトレースできる
  - 関数の実行開始をトレースするkprobe を使って実装されている
- kprobe、タイムスタンプを記録するトレーサーとの組み合わせでkretprobeを使うと、カーネル関数の実行時間を計測できる。
```
# bpftrace -e 'kprobe:do_nanosleep { @ts[tid] = nsecs; }
kretprobe:do_nanosleep /@ts[tid]/ {
@sleep_ms = hist((nsecs - @ts[tid]) / 1000000); delete(@ts[tid]); }
END { clear(@ts); }'
Attaching 3 probes...
^C
@sleep_ms:
[0] 1280 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@|
[1] 1 | |
[2, 4) 1 | |
:
:
```
- bpftrace の構文は、「15 章BPF」で説明する
  
#### 4.3.6.4 kprobeのインターフェイスとオーバーヘッド
- krobeは、/sys ファイル、perf_event_open(2)、register_kprobe()カーネルAPIを介してインストルメンテーションできる
- 関数の実行開始をトレースするときには、オーバーヘッドはトレースポイントとほぼ同じだが、オフセットをトレースするとき（ブレークポイント方式）やkretprobe を使うとき（trampoline 関数）にはトレースポイントよりもオーバーヘッドが高くなる

#### 4.3.6.5 kprobeのドキュメント
- kprobe はLinux ソースのDocumentation/kprobes.txt でドキュメントされている
  - https://www.kernel.org/doc/Documentation/kprobes.txt

### 4.3.7 uprobe
- uprobe（user-space probeを意味する）は、アプリケーションやライブラリに含まれる関数の動的インストルメンテーションをすることができ、ほかのツールでできる範囲を越えて、ソフトウェアの内部を深く掘り下げるための不安定APIを提供する

#### 4.3.7.1 uprobeの例
- シェルのbash(1) のなかでuprobe関数を挿入できる位置をリストアップする
```
# bpftrace -l 'uprobe:/bin/bash:*'
uprobe:/bin/bash:rl_old_menu_complete
uprobe:/bin/bash:maybe_make_export_env
uprobe:/bin/bash:initialize_shell_builtins
uprobe:/bin/bash:extglob_pattern_p
uprobe:/bin/bash:dispose_cond_node
uprobe:/bin/bash:decode_prompt_string
[..]
```
- デフォルトでは、ユーザー空間コードは変更されずに実行される

#### 4.3.7.2 uprobeによる引数のトレース
- bpftrace を使ってbash のdecode_prompt_string( ) 関数をインストルメンテーションし、第1 引数を文字列で表示し
```
# bpftrace -e 'uprobe:/bin/bash:decode_prompt_string { printf("%s\n", str(arg0)); }'
Attaching 1 probe...
\[\e[31;1m\]\u@\h:\w>\[\e[0m\]
\[\e[31;1m\]\u@\h:\w>\[\e[0m\]
^C
```
- uprobe イベントは、bpftrace プログラムが実行を開始したときに作られ、bpftrace が強制終了（Ctrl-C）されたときに削除される

#### 4.3.7.3 uretprobe
- ユーザー関数からのリターンと戻り値は、uretprobe（user-level return probe を意味する）でトレースできる
- uretprobe のオーバーヘッドによって高速な関数の計測値は大きく歪むことに注意しなければならない

#### 4.3.7.4 uprobeのインターフェイスとオーバーヘッド
- uprobe のインターフェースは/sys ファイルかperf_event_open(2)システムコール（こちらの方が望ましい）でインストルメンテーションできる
- 現在のuprobe は、トラップでカーネルに入り込んで動作する
- これはkprobe やトレースポイントよりもCPUオーバーヘッドが高い

#### 4.3.7.5 uprobeのドキュメント
- Documentation/trace/uprobetracer.rst
  - https://www.kernel.org/doc/Documentation/trace/uprobetracer.txt

### 4.3.8 USDT
- USDT（User-level Statically-Defined Tracing）は、トレースポイントのユーザー空間バージョンである
- 一部のアプリケーションとライブラリは、コードにUSDTプローブを追加し、アプリケーションレベルイベントをトレースするための安定（公開）API を提供している
- 👩‍💻 DTrace で使われている技術
  - https://ja.wikipedia.org/wiki/DTrace
- bpftrace を使ってOpenJDKのUSDTプローブをリストアップする場合
```
# bpftrace -lv 'usdt:/usr/lib/jvm/openjdk/libjvm.so:*'
usdt:/usr/lib/jvm/openjdk/libjvm.so:hotspot:class__loaded
usdt:/usr/lib/jvm/openjdk/libjvm.so:hotspot:class__unloaded
usdt:/usr/lib/jvm/openjdk/libjvm.so:hotspot:method__compile__begin
usdt:/usr/lib/jvm/openjdk/libjvm.so:hotspot:method__compile__end
usdt:/usr/lib/jvm/openjdk/libjvm.so:hotspot:gc__begin
usdt:/usr/lib/jvm/openjdk/libjvm.so:hotspot:gc__end
[...]
```
- ディスクI/Oではなくファイルシステム内のロックの競合のためにクエリーが遅くなっていることまで明らかにすることができる
- USDTは、パッケージバージョンのアプリケーションでは有効になっていないため、使うためには適切な構成オプションを指定してリビルドする必要がある
- USDTプローブは、インストルメンテーションの対象である実行可能ファイルにコンパイルして組み込まなければならない
- Javaのようにその場でJITコンパイルされる言語では、ダイナミックUSDTを使う
  - ダイナミックUSDTプローブは共有ライブラリという形でプローブをプレコンパイルし、JIT コンパイルされる言語からプローブを呼び出すためのインターフェイスを提供する
- USDTプローブは、有効化したときのオーバーヘッドだけでなく、無効可したときにもコード内にnop 命令が入るというオーバーヘッドがかかる

#### 4.3.8.1 USDTのドキュメント
- USDTプローブを使えるようにしているアプリケーションは、そのドキュメントでUSDTプローブのこ
とを説明しているはずだ

### 4.3.9 ハードウェアカウンタ(PMC)
- プロセッサなどのデバイスは、一般にアクティビティの観測のためにハードウェアカウンタをサポートしている
- PMC（Performance Monitoring Counter）
- CPUサイクルレベルの低水準パフォーマンス情報を提供するプロセッサのプログラマブルなハードウェアレジスタを参照している
- パフォーマンス解析に利用

#### 4.3.9.1 PMCの例
- Intelは IntelのPMCの中で、中心的な機能の高レベルの概要を提供する7個を選んで「architectural set」としている

表4-4 Intel のarchitectural PMC

| イベント名 | UMASK | イベント選択 | イベントマスクニーモニックの例 |
| -------- | ----- | ---------- | ------------------------- |
| ホルトされていないクロック数 | 00H | 3CH | CPU_CLK_UNHALTED.THREAD_P |
| リタイアした命令 | 00H | C0H | INST_RETIRED.ANY_P |
| ホルトされていない参照サイクル | 01H | 3CH | CPU_CLK_THREAD_UNHALTED.REF_XCLK |
| LLC参照 | 4FH | 2EH | LONGEST_LAT_CACHE.REFERENCE |
| LLCミス | 41H | 2EH | LONGEST_LAT_CACHE.MISS |
| リタイアした分岐命令 | 00H | C4H | BR_INST_RETIRED.ALL_BRANCHES |
| リタイアした分岐ミス | 00H | C5H | BR_MISP_RETIRED.ALL_BRANCHES |

- イベントを指定せずに（-eなし）perf statコマンドを実行すると、デフォルトでarchitectural PMCをインストルメンテーションする
- ![perf_start_zip_words](./images/ch04/4-4-9-1-pmc.png)
- 最初の欄は未加工のカウンタの値
- #の後ろは重要なパフォーマンス指標であるサイクルあたりの命令数（insn per cycle）などの統計量
- insn per cycleはCPUがどの程度効率よく命令を実行しているかを示し、高ければ高いほどよい

#### 4.3.9.2 PMCのインターフェイス
- Linux では、PMCはperf_event_open(2) システムコールでアクセスされ、perf(1) などのツールで使われる
- CPUのレジスタは数に限りがあり、同時に計測できるのは6個ぐらいまで
- モード
  - カウンティングモード: 実質的にオーバーヘッド0 でイベントをカウント
    - 問題の定量化に使える
  - オーバーフローサンプリングモード: 設定可能な回数のイベントのうちのひとつに対して割り込みを生成し、状態を捕捉する
    - 問題の原因となっているコードパスを示すために使える

#### 4.3.9.3 PMCの難点

- オーバーフローサンプリングの難点
  - 割り込みのレイテンシ（「スキッド」と呼ばれることが多い）や命令のアウトオブオーダー実行のために、イベントをトリガリングした命令ポインタを正しく記録できない
    - サンプリングとターゲットが同サイクルにならないようにわざとジッターを入れる
    - サンプリングレートを99Hzにする
    - LLCミスなどのイベントの計測では、サンプリングされた命令ポインタは正確でなければならないので上記対策は良くない
      - 👩‍💻 LLCミス：CPUのキャッシュメモリであるLLC（Last Level Cache）において、意図しないキャッシュミスが発生すること
  - プロセッサがprecise event（正確なイベント）という機能をサポートして解決されている
    - Linuxのperf(1)コマンドは、precise eventをサポートしている
- クラウド環境での利用
  - 多くのクラウド環境がゲストのPMCアクセスを無効化している
  - 技術的には、クラウドでPMCを有効化することはできる
    - Xenハイパーバイザーは、vpmuというコマンドラインオプションを持っており、ゲストに対して異なるPMCセットを示すことができる
    - Amazonは、Nitroハイパーバイザーのゲストに対して多くのPMCを有効にしている
      - 🤔 最近のインスタンスは Nitro System で動いているインスタンスは多いが、PMCが有効かどうかはよくわからなかった
    - ベアメタルインスタンスが提供されているクラウドプロバイダもある
      - 👩‍💻ベアメタルインスタンス：仮想化レイヤを経由せずに直接、基盤となるCPU/メモリにアクセス可能
    - 👩‍💻 vPMCの利用 https://kb.vmware.com/s/article/2030221
      
#### 4.3.9.4 PMCのドキュメント
- PMCはプロセッサごとに固有のものであり、プロセッサの適切なソフトウエアデベロッパマニュアルにドキュメントされている

### 4.3.10 可観測性ツールのその他の情報ソース
- MSR
- ptrace(2)
- 関数のプロファイリング
- ネットワークスニッフィング（libpcap）
- netfilter conntrack
  - イベント発生時にカスタムハンドラを実行できるようにしており、用途はファイアウォールだけでなく、接続追跡も含まれる
  - これを使うとネットワークのフローのログを作れる
- プロセスアカウンティング
  - メインフレームの時代からプロセスの実行と実行時間に基づいて部門やユーザーにコンピュータの利用料を請求するために使われていた
  - 👩‍💻コマンド実行履歴だがhistory logとは違ってユーザによって消されることはない
- ソフトウェアイベント
- システムコール
#### 4.3.10.1 さらなる情報ソース
- 情報ソースを見つけるためのテクニックとしては、観察したいカーネルコードを読み、どのような統計やトレースポイントが配置されているかを見てみるなどの方法がある
- カーネル/アプリケーション変数をフェッチするgdb(1) やlldb(1) などのデバッガが役立つことがある
#### 4.3.10.2 Solaris Kstat
- Kstat は、次の4 要素のタプルを使ってカーネル統計の統一的な階層構造を提供している
  - module: 統計量を作成したカーネルモジュール
  - instance:
  - name: 統計量のグループの名前
  - statistic: 個別の統計量の名前
- 4 タプルを全部指定したkstat(1M) を使ってnproc 統計量を読み出し
```
$ kstat -p unix:0:system_misc:nproc
unix:0:system_misc:nproc 94
```
## 4.4 sar
- sar(1) は、それだけで多くのパフォーマンス障害を解決できる重要なシステムパフォーマンスツールだ
- sar(1) は、sysstat パッケージから提供されている
### 4.4.1 sar(1)でできること
- ![sar](./images/ch04/figure-4-6.png)
### 4.4.2 sar(1)モニタリング
#### 4.4.2.1 構成（Ubuntu）
- Ubuntu システムでは、ENABLED がtrue になるように/etc/default/sysstat ファイルを編集して sysstatを再起動すれば、sar(1) データ収集が有効になる。
```/etc/default/sysstat
#
# Default settings for /etc/init.d/sysstat, /etc/cron.d/sysstat
# and /etc/cron.daily/sysstat files
#
# Should sadc collect system activity informations? Valid values
# are "true" and "false". Please do not put other values, they
# will be overwritten by debconf!
ENABLED="true"
```

```shell
ubuntu# service sysstat restart
```

- 統計量を記録するスケジュールは、sysstat のcrontab ファイル /etc/cron.d/sysstat で変更できる

```
# The first element of the path is a directory where the debian-sa1
# script is located
PATH=/usr/lib/sysstat:/usr/sbin:/usr/sbin:/usr/bin:/sbin:/bin
# Activity reports every 10 minutes everyday
5-55/10 * * * * root command -v debian-sa1 > /dev/null && debian-sa1 1 1
# Additional run at 23:59 to rotate the statistics file
59 23 * * * root command -v debian-sa1 > /dev/null && debian-sa1 60 2
```

- 全部の記録を取るなら、-S ALL オプションを使う
  
#### 4.4.2.2 出力
- たとえば、次のコマンドラインは、CPU統計（-u）とTCP（-n TCP）、TCPエラー（-n ETCP）を出力する
- ![sar](./images/ch04/4-4-2-2-sar.png)
- 出力の先頭行はシステムのサマリー
- sar -Aを実行すれば、すべての統計量が出力される
#### 4.4.2.3 出力形式
- sysstat パッケージにはsar(1) 統計を異なる形式で表示するためのsadf(1) コマンドが付属していて、JSON、SVG、CSV などの形式がサポートされている
  
### 4.4.3 sar(1)のライブ出力
- インターバルとオプションの回数を指定してsar(1) を実行すると、ライブレポートが見られる
- 1 秒間隔で5 回分のTCP統計を表示
- ![sar](./images/ch04/4-4-3-sar.png)
### 4.4.4 sar(1)のドキュメント
- sar(1) の具体的な使い方は本書で後述していく。6 章から10 章を参照していただきたい
## 4.5 トレーシングツール
- 主要なトレーシングツール
  - perf(1)
    - Linux の公式プロファイラ
  - Ftrace
    - Linux の公式トレーサー
    - function tracing
  - BPF（BCC, bpftrace）
    - BCC (BPF Compiler Collection)
      - https://github.com/iovisor/bcc/
    - bpftrace
      - トレーシング処理を専用の言語で記述できる
      - 👩‍💻 参考 https://atmarkit.itmedia.co.jp/ait/articles/2006/15/news016.html
  - SystemTap
    - 👩‍💻カーネルを再構築することなくカーネルの動作を深く詳細に確認できる
      - 実行中のカーネルの処理をフックして、必要に応じて情報を収集し、出力する
      - 参考 https://gihyo.jp/admin/serial/01/ubuntu-recipe/0584
  - LTTng
    - https://lttng.org/
  
  > 私は、Netflix の仕事では、CPU の分析にはperf(1)、カーネルコードの掘り下げにはFtrace、その他すべて（メモリ、ファイルシステム、ディスク、ネットワーキング、アプリケーションのトレーシング）ではBCC/bpftrace を使っている。

## 4.6 可観測性ツールに対する観察

> 新しく触れる統計には健全な懐疑主義をもって臨み、その本当の意味は何か、本当に正しいのかどうかを問うようにすべきだ。

- 指標には、次の問題が潜んでいる可能性がある
  - ツールと計測値が間違っている場合がある
  - manページはかならずしも正しくない
  - 得られる指標が不完全な場合がある
  - 得られる指標のデザインに問題があり、紛らわしい場合がある
  - 指標収集コード（たとえば、ツールの出力のパーサー）にバグが含まれている場合がある
  - 指標の処理（アルゴリズム/スプレッドシート）でも誤りが入る場合がある

- 対策
  - 複数の可観測性ツールでクロスチェック、ダブルチェックする
    - 異常な結果が出たときや会社にとって死活的に重要な意味を持つ結果を扱うときだけでよい
  - 結果がわかっているワークロードを使って、可観測性ツールが予想通りの結果を出せるかどうかをチェックする
  - 「2 章メソドロジ」を読み、足りない指標がないか検討する


## 4.7 練習問題

1. 静的パフォーマンスツールの例をいくつか挙げなさい。
2. プロファイリングとは何か
3. プロファイラが100Hzではなく99Hzを使うのはなぜか。
4. トレーシングとは何か。
5. 静的インストルメンテーションとは何か。
6. 動的インストルメンテーションが重要な理由を説明しなさい。
7. トレースポイントとkprobe の違いは何か。
8. 以下の作業で予想されるCPUのオーバーヘッド（低いか普通か高いか）を説明しなさい。
   1. ディスクのIOPSカウンタ（iostat(1) で見られるもの）
   2. トレーシングポイントかkprobe を使ったディスクI/Oごとのトレーシング
   3. トレーシングポイントかkprobe を使ったコンテキストスイッチごとのトレーシング
   4. トレーシングポイントかkprobe を使ったプロセスの起動（execve(2)）ごとのトレーシング
   5. uprobe を使ったlibc malloc( ) 呼び出しごとのトレーシング
9. パフォーマンス分析でPMCに価値がある理由を説明しなさい。
10. 可観測性ツールが与えられたとき、どうすればそれが使っているインストルメンテーションソースを明らかにできるかを説明しなさい。


## ４.８ 参考文献
