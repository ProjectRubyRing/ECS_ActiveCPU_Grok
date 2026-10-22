# Fargate 上の JDK は、最初から CPU 数を正しく数えるのか

調査日: 2026-10-23

この文書は、次の疑問への結論です。

ECS のスケールアップでは、タスク定義の外から `-XX:ActiveProcessorCount` を指定することが推奨される。特に Fargate では推奨される。この CPU 数の認識不良は、オートスケールで大きくしたときだけ起きるのか。最初からその CPU で起動している場合や、タスクを止めてきれいに起動し直した場合には起きず、その場合は JDK が Fargate の CPU 数を正しく認識するのか。

## 結論

認識の判定は、JVM が起動する瞬間に一度行われます。最初の起動でも、きれいに再起動したあとも、オートスケールで増えた新しいタスクでも、同じ読み取りをします。

Fargate のサービスオートスケールは、動いているタスクの CPU を後から足しません。タスクの台数を増減します。タスク定義の CPU を大きくする場合も、動いている JVM の中身を書き換えず、新しいタスクを起動します。新しいタスクは、最初のタスクと同じ入り口を通ります。

そのため、次のように整理できます。

| 起動のしかた | JVM は新しいプロセスか | 判定 |
| --- | --- | --- |
| 最初からその CPU で起動する | はい | その JDK と Fargate の組み合わせ次第。最初から合っていることもあるし、最初から合わないこともある |
| タスクを止めて、同じタスク定義できれいに起動し直す | はい | 同じ組み合わせなら、前回と同じ数を読む。再起動そのものは治療にならない |
| サービスオートスケールでタスクを増やす | 増えたタスクは新しい JVM | 既存タスクと同じ判定をする。スケールの瞬間だけ壊れる、という動きではない |
| タスク定義の CPU を大きくして、新しいリビジョンをデプロイする | 新しいタスクは新しい JVM | 新しいサイズを、その JDK の読み方で読む。読み方が合っていなければ、大きいサイズでも誤る |
| 動いている JVM の cgroup だけを後から書き換える | いいえ | Fargate の通常のオートスケールはこの操作をしない |

「正しく認識されるか」は、起動がきれいかどうかでは決まりません。次の三点で決まります。

1. JDK が CPU 数をどのファイルから計算するか。
2. Fargate がそのファイルに何を書くか。
3. タスク定義の CPU を、タスク全体だけに書いたか、コンテナにも書いたか。

2026-07-20 の AWS Containers Blog の現行案内は、こうです。現行の LTS のうち JDK 8 以外では、Fargate 上の自動検出がタスクの vCPU と一致しないことがある。`-XX:ActiveProcessorCount` をタスクの vCPU に合わせる。JDK 8 は、整数 vCPU では自動検出がおおむね信頼できる。

この案内の前提も、起動時の読み取りです。スケール中だけ特別な障害が起きる、とは書かれていません。

## この結論を、教室のたとえで読む

大きな学校の建物に、机がたくさんあります。これがコンピュータ本体です。

あなたのクラスは、そのうち 4 脚だけ使う契約をしています。これが Fargate のタスクです。256 CPU ユニットが 0.25 脚、512 が 0.5 脚、1024 が 1 脚、2048 が 2 脚、4096 が 4 脚です。

Java は、クラスが始まる朝、入り口で机の数を数え、その数だけお手伝い係を雇います。係は、ごみ集め、翻訳、仕事の分配です。朝に雇った人数は、その日のクラスの基本になります。授業の途中で学校が忙しくなっても、Java は契約書を読み直して係を増やしたりはしません。

ECS のオートスケールで起きていることは、多くの場合「今のクラスに机を足す」ではありません。「同じ契約のクラスをもう一つ開ける」です。新しいクラスも、朝に入り口で数えます。数え方が間違っていれば、一番最初のクラスから間違っています。クラスを一度解散して、同じ契約で開き直しても、同じ数え方をします。

`ActiveProcessorCount` は、入り口に貼る紙です。「このクラスの机は 4 脚」と、Java に直接書きます。Java は推測をやめ、紙の数字を使います。紙は、最初の朝も、開き直しの朝も、新しく開いたクラスの朝も、同じように働きます。

紙の数字は、タスク定義の CPU を変えても自分では追従しません。2 脚の契約のときに紙へ「2」と書き、あとから契約を 4 脚に変えて紙を 2 のままにすると、そのときは本当に「大きくしたのに Java が 2 のまま」になります。これは Fargate の検出不良ではなく、紙を更新しなかった結果です。

## なぜ「スケールアップのときだけ」に見えるのか

1 vCPU のタスクでは、間違った数え方でも答えが 1 になることがあります。見た目は成功です。2 vCPU や 4 vCPU に増やすと、答えが 1 のままなので、急に失敗に見えます。人が気づくタイミングがスケールアップだからといって、故障がスケールアップの瞬間に発生したことにはなりません。

OpenJDK が自分の古い数え方として記録している対応は、次のとおりです。出典は JDK-8281181 の説明文です。

| cgroup の cpu.shares | 古い JDK の解釈 |
| --- | --- |
| 0 から 1023 | 1 CPU |
| 1024 | 制限なし。そのあと、見えている CPU 数へ落ちる |
| 2048 | 2 CPU |
| 4096 | 4 CPU |

ECS の CPU ユニットは、この表と同じ数字を使います。1 vCPU は 1024 です。JDK の特別扱い「1024 は制限なし」と、ECS の「1024 は 1 vCPU」が同じ数字です。

コンテナの `cpu` を空欄にすると、別の小さい数字が入ります。ECS エージェントは、空欄、0、1 を Linux が許す最小の 2 shares として渡す、と長年文書化されています。2 は「0 から 1023」に入るので、古い JDK は 1 CPU と読みます。JJUG CCC 2022 Fall の発表では、Fargate 上でコンテナ CPU を未設定にすると、JVM のログに CPU Shares 2 が出て、結果として 1 になっていました。

この組み合わせで見える数は、こうなります。

| タスクに買った CPU | コンテナ cpu 未設定で shares が 2 のとき、古い JDK が見る数 | 人が感じること |
| --- | --- | --- |
| 0.25 vCPU（256） | 1 | 動いている。保証より多く見えているが、1 vCPU のときと区別しにくい |
| 0.5 vCPU（512） | 1 | 同上 |
| 1 vCPU（1024） | 1 | 正しく見える。錯覚が始まる地点 |
| 2 vCPU（2048） | 1 | 大きくしたのに速くならない |
| 4 vCPU（4096） | 1 | さらに差が開く |
| 8 vCPU 以上 | 1 | 同じ |

1 vCPU の行だけ、誤認と正解が同じ数字です。最初から 1 vCPU で運用している間は、検出が正しいように見えます。2 や 4 に変えて新しいタスクを起動したとき、初めて「認識していない」と分かります。きれいに再起動しても 1 のままなら、それは古い値を覚えているのではありません。新しい JVM が、同じファイルから再び 1 を読んだのです。

逆に、再起動のあとだけ正しい数になる場合は、再起動と同時に別のものも変わっています。コンテナの `cpu` を書いた、JDK の更新番号が変わった、`ActiveProcessorCount` を足した、プラットフォームの見え方が変わった、のいずれかです。プロセスを入れ替えたこと自体が、cgroup の読み方を賢くするわけではありません。

## 二つの「CPU の話」を分ける

会話の中で「スケールアップ」は、次の二つをまとめて指すことがあります。機械の動きは別です。

### 台数を増やす

Application Auto Scaling が ECS サービスに対して行うのは、desired count の変更です。2026-10-20 時点の Amazon ECS 開発者ガイドでも、スケールの単位はタスク数です。2026-06-18 に発表された高速化は、CPU とメモリのメトリクスを 20 秒解像度で見て、タスクを早く増やす機能です。動いているタスクの vCPU を広げる機能ではありません。

増えたタスクは、停止していた JVM の続きではありません。新しい Firecracker microVM、新しいコンテナ、新しい JVM プロセスです。入り口の紙と、JDK の数え方が同じなら、1 台目と 2 台目は同じ CPU 数を見ます。

### 1 台を大きくする

Fargate のタスク CPU は、そのタスクが生きている間は固定です。大きくするには、タスク定義の新しいリビジョンを登録し、サービスをデプロイします。古いタスクは止まり、新しいタスクが起動します。ここでも JVM は新しく生まれます。

「動いている Java が、増えた CPU に気づかない」という種類の問題は、別の世界の話です。Linux の CPU ホットプラグや、Kubernetes の in-place な Pod リサイズでは、プロセスを殺さずに制限ファイルが書き換わることがあります。Java のごみ集め係や `ForkJoinPool` の共通プールは、起動時の人数を基本にするので、ファイルが後から変わっても係の人数は追従しないことがあります。`Runtime.availableProcessors()` の説明文は「実行中に変わり得る」と書いてあり、HotSpot も短いキャッシュのあと読み直す実装を持っています。ただし Fargate の通常運用では、生きているタスクの CPU ファイルがオートスケールで書き換わりません。読み直しても、朝と同じ数字です。

したがって、Fargate で「再起動すれば直る」が成り立つのは、生きているプロセスが古い人数を覚えている、という状況です。通常の ECS オートスケールと、通常のタスク定義更新は、その状況を作りません。

## 歴史と、フラグが必要になった理由

### 2006 年から 2014 年頃。Java は機械全体を数えた

Java は、自分が動く機械に CPU がいくつあるかを、OS に聞いていました。Linux ではオンラインのプロセッサ数です。1 台のサーバを 1 つの Java が占有する時代には、その答えがそのまま使える CPU 数でした。

この人数は、次の計算に使われます。

- ごみ集めの作業員数。`ParallelGCThreads` や `ConcGCThreads`。
- JIT コンパイラのスレッド数。`CICompilerCount`。
- `ForkJoinPool` の共通プール。多くのライブラリが、ここへ平行な仕事を投げます。
- `Runtime.availableProcessors()`。アプリケーションが自分でスレッドプールを作るときの材料です。

人数が多すぎると、スレッドのスタックメモリが増え、短い CPU 時間を奪い合います。少なすぎると、買った CPU が暇なまま、処理が一本の列になります。

### 2013 年から 2017 年。容器が先に生まれ、Java は後から気づいた

Docker は、Linux の cgroup で「このプロセス群は CPU のこの割合まで」と制限します。制限の書き方は二系統あります。

- 重み。cgroup v1 の `cpu.shares`、cgroup v2 の `cpu.weight`。隣の容器と争ったときの取り分です。隣が休んでいれば、自分はもっと使えます。
- 上限。cgroup v1 の `cpu.cfs_quota_us` と `cpu.cfs_period_us`、cgroup v2 の `cpu.max`。周期の中で使ってよい時間の天井です。余っていても、天井を超えられません。

容器の中の `/proc/cpuinfo` は、長いあいだ建物全体の机を見せていました。Java は建物全体を数え、係を何十人も雇いました。実際に座れるのは 2 脚です。2015-12-23、Oracle の Bob Vandette 氏が JDK-8146115 を登録しました。容器の外の OS 情報ではなく、cgroup の割当を JVM の初期値に使え、という修正です。

### 2017 年 11 月から 2018 年 10 月。容器対応とフラグが届いた

JDK-8146115 は 2017-11-16 に解決し、Java 10 に入りました。同時に `-XX:ActiveProcessorCount` が追加されました。自動の推測をやめ、人数を直接指定する出口です。リリースノートは、このフラグが `UseContainerSupport` を切っていても有効だと書いています。

- 2018-03-20。Java 10 が一般提供。容器対応が標準で入った最初のリリースです。
- 2018-09-25。Java 11 LTS。同じ対応を含みます。
- 2018-10-16。Java 8u191。企業で残っていた Java 8 にも backport されました。`-XX:+UseContainerSupport` はデフォルトで有効です。

必要になった理由は単純です。容器が「使ってよい CPU」を機械全体より小さくしたのに、Java は機械全体を見ていたからです。フラグは、自動検出が環境と合わないときの、検出ロジックより優先される指定です。

### 2017 年末。Fargate が、共有しないタスクを始めた

AWS Fargate は re:Invent 2017 で ECS の起動タイプとして提供が始まりました。利用者はインスタンスを持ちません。タスク定義に CPU とメモリを書き、AWS が置き場所を用意します。

2019-10-21 の AWS 記事は、Fargate の各タスクが独自の分離境界を持ち、カーネルも CPU もメモリも ENI も他のタスクと共有しない、と説明しました。同じ記事は、当時の実装として、タスクの CPU はカーネルのハードな天井、コンテナの CPU はタスクの中の shares だと書いています。

この「タスクは quota、コンテナは shares」という 2019 年の説明は、2026 年の文書と違います。現行の正は、あとの節に書きます。

### 2018 年 12 月。空欄が「2」になり、Java が「1」と読む

aws/amazon-ecs-agent issue 1735 が、この落とし穴を記録しました。コンテナの CPU を空欄にすると、エージェントは 2 shares を渡します。Java 10、11、当時の Java 8 は、1024 未満を 1 CPU と読みます。大きなタスクでも `availableProcessors()` が 1 になります。

エージェント側の理由は、Linux の cpu.shares の最小が 2 だからです。Docker は 0 を受け取るといつもの 1024 に置き換えます。ECS は「指定なし」を 1024 ではなく最小値にしました。1024 は「1 CPU の予約」と「Docker の初期値」の両方に見える数字だったためです。

この挙動はスケール操作の副作用ではありません。タスク定義を登録して、最初のタスクを起動したときから入ります。

### 2020 年 4 月。プラットフォーム 1.4.0 と microVM

Fargate の Linux プラットフォーム 1.4.0 は、コンテナ実行を Docker から containerd 系へ移し、各タスクを Firecracker microVM に置く形を広めました。2026-10-20 時点の Amazon ECS 開発者ガイドでも、Linux の LATEST は 1.4.0 です。8 vCPU、16 vCPU、32 vCPU はこのプラットフォーム以降です。

microVM は自分のカーネルで起動します。容器の中から見える「ホスト CPU」は、建物全体の 64 コアではなく、その microVM に渡された vCPU 数です。古い「64 コアのサーバに置いたのに Java が 64 を見る」は、Fargate では形を変えます。見る相手が、小さな仮想機械になるからです。

小さな仮想機械の vCPU 数が、契約した vCPU と必ず一致する、とまでは現行資料は言い切っていません。AWS は 2026 年の CPU 節で、一致しないことがあるのでフラグを使え、と書いています。

### 2020 年から 2023 年。cgroup v2

cgroup v2 の対応は JDK 15 の JDK-8230305 で入り、JDK 11.0.16 以降と JDK 8u372 以降へ backport されました。Amazon Corretto も同じ更新番号です。それより古い JDK は、v2 のホストで制限を読まず、ホスト側の数へ静かに落ちることがあります。警告を出さない、と 2026-07-20 の AWS 記事は書いています。

Fargate のゲストカーネルは利用者が選べず、タスクの退役と入れ替えで AWS が更新します。今のタスクが v1 か v2 かは、起動ログで見るのが確実です。

### 2022 年。OpenJDK が「重みは CPU 数ではない」と判断した

2022-02-02、Ioi Lam 氏が JDK-8281181 を登録しました。対応する CSR は JDK-8281571 です。

cpu.shares は相対的な重みです。A と B が同じ重みなら、争っているとき半分ずつです。B が休んでいれば、A は残りの CPU を使えます。絶対的な「CPU が何個あるか」ではありません。

JDK は 2018 年以来、この重みを絶対数として使っていました。Kubernetes で CPU request だけを書き、limit を書かない運用では、小さな重みのために JVM が 1 CPU しか使わず、ノードに余力があっても仕事が進みません。これは過少利用です。OpenJDK は、重みを CPU 数の計算から外しました。上限である quota があるときだけ、その上限を CPU 数に使います。上限がなければ、見えている CPU 数を使います。

この修正は Kubernetes の「request は希望、limit が天井」には合います。Fargate には逆に効きます。2026 年の AWS 文書では、Fargate の CPU 制御は shares であり、quota と period は CPU 制限に使われない、と明記されているからです。JVM が shares を無視し、quota も無いと、タスクに書いた vCPU を読む材料が残らず、microVM 側の CPU 数へ落ちます。

修正の入り方は次のとおりです。

| リリース | 内容 |
| --- | --- |
| JDK 19 b13 | 2022-03-04 に修正。`UseContainerCpuShares` の初期値は false。古い挙動へ戻す一時フラグ |
| JDK 18.0.2 | backport |
| JDK 17.0.5 | 2022 年の更新リリースへ backport。初期値は shares を使わない |
| Oracle JDK 17.0.3.0.1 | Oracle 側の backport |
| JDK 11.0.17 | 2022 年の更新リリースへ backport |
| JDK 8 | この修正の backport 一覧には無い。AWS の 2026 年記事も、JDK 8 は例外として扱う |

更新リリースへ入れたあとに、利用中のワークロードがメモリ不足になった、という報告がありました。重みを「CPU 数の上限」として依存していた構成が、急に全コアを見るようになり、スレッドが増えたためです。JDK-8299037 は、8u、11u、17u では古い方を初期値に戻す提案でした。決議は Won't Fix です。11 と 17 の更新リリースは、新しい方を初期値のままにしています。戻すには `-XX:+UseContainerCpuShares` を明示します。

JDK 21 では、この戻しフラグ自体が削除されました。JDK-8282684 で廃止予定になり、21 で期限切れです。JDK 25 も同じです。21 以降に残る安定した指定は、`ActiveProcessorCount` です。

### 2022 年 11 月。日本語の現場が、Fargate で再現した

JJUG CCC 2022 Fall「Fargate上のJVMからCPUを認識するまで」は、ECS のタスク定義とコンテナ定義の両方に CPU があり、当時の JVM が見るのはコンテナ側だと示しました。コンテナ側が空だと shares が 2 になり、1 CPU になります。発表の推奨は、コンテナ定義にも CPU を書くことと、GC を明示することでした。

この発表は、起動時のログを Fargate の中で取った記録です。スケールの途中で値が劣化した記録ではありません。

### 2026 年 7 月 20 日。AWS が現行の書き方を一文にした

Fernanda Machado 氏と Henrique Santana 氏の AWS Containers Blog が、現行の実務案内です。

Fargate では CPU は shares で制御され、CFS の quota と period ではない。比例配分であり、余力があれば割当を超えてバーストでき、CFS スロットリングは掛からない。一方、現行 LTS の JDK 8 以外は JDK-8281571 により `cpu.shares` を `availableProcessors()` の計算に使わない。quota が無いので、JVM はホストの CPU 数へ落ちる。戻しフラグ `UseContainerCpuShares` は JDK 21 で削除された。タスクの vCPU に合わせて `-XX:ActiveProcessorCount=N` を使え。

同じ記事は、JDK 8 の整数 vCPU では自動検出がおおむね信頼できる、とも書いています。EC2 起動タイプと EKS on EC2 では、CPU は CFS quota という硬い天井になり、端数の CPU でスレッドを盛りすぎるとスロットリングと遅延が起きる、とも書いています。Fargate でフラグが要る理由と、EC2 でフラグが要る理由は、同じ文章の中でも別です。

同じ記事の少し前には、Fargate はタスクごとに固定リソースの microVM を割り当て、割当と利用可能量の隙間が無く、JVM の cgroup 検出は信頼できる、という段落もあります。この段落はメモリと、専用 microVM であることの説明に続いています。CPU 節は別に、「一致しないことがあるのでフラグを使え」と書いています。両方を同時に読むと、メモリや専用仮想機械であることと、CPU 数の自動検出が常にタスク vCPU と一致することとは、同じ主張ではありません。実務で優先するのは、CPU 節の明示的な推奨です。

### 2026 年 10 月の公式ドキュメント

Amazon ECS 開発者ガイドの CPU とメモリのページは、2026-10-21 前後の更新で次のように書いています。

- EC2 上のタスク。CPU period と CPU quota を使い、タスクサイズを cgroup の硬い上限にする。
- Fargate 上のタスク。CPU shares で配分する。quota と period の値は、Fargate タスクの CPU 制限には使わない。

コンテナ定義の `cpu` は、Linux では `CpuShares` に対応し、Fargate では任意です。タスク全体の `cpu` は Fargate では必須です。

ここに、二つの数字が並びます。JVM の古い検出はコンテナの shares を見ました。スケジューラと課金が見るのはタスクの CPU です。タスクだけを正しく書いて、コンテナを空にすると、二つの数字が分かれます。

## いまの Fargate で、JDK は何を見るか

起動時の判定を、順に書くとこうなります。

1. `-XX:ActiveProcessorCount` が 1 以上なら、その整数を採用する。自動検出より先です。0 以下は「未指定」として扱われ、検出へ進みます。
2. 未指定なら、容器対応が有効か見る。デフォルトは有効です。`-XX:-UseContainerSupport` があると、容器の制限を見ません。
3. 上限ファイルを見る。v1 なら quota / period、v2 なら `cpu.max`。Fargate では、2026 年の AWS 文書どおり quota が制限に使われていないなら、ここは「上限なし」です。
4. JDK 8、および 11.0.17 未満や 17.0.5 未満で shares をまだ使う JDK は、`cpu.shares` を上の古い表で CPU 数に換算する。
5. 11.0.17 以降、17.0.5 以降、19 以降の初期状態は、shares を換算しない。21 と 25 は戻しフラグがありません。
6. 換算結果が無いとき、JVM は自分から見えているプロセッサ数を使う。Fargate では、それが microVM の vCPU 数です。建物全体のコア数ではありません。

「見えている数」と「契約した数」が一致するなら、フラグが無くても `availableProcessors()` は合います。AWS は、現行 LTS の JDK 8 以外では一致しないことがある、と書いています。一致する環境を一度観測しても、ゲストカーネルや JDK の更新番号が変わると材料が変わります。フラグは、その変動から人数を切り離します。

### JDK ごとの実務

| JDK | Fargate での自動検出 | 実務 |
| --- | --- | --- |
| 8u191 未満 | 容器対応が無い | 更新する。フラグも、この世代には無いことがある |
| 8u191 以上 8u371 まで | shares を読む。cgroup v2 は未対応のことがある | 整数 vCPU かつ、コンテナ cpu が契約どおりなら、AWS の言うとおりおおむね合う。空欄のコンテナ cpu は 1 に落ち得る。v2 環境では更新が先 |
| 8u372 以降 | shares を読む。v1 と v2 の両方を検出できる | 整数 vCPU ではおおむね信頼できる、が AWS の表現。空欄コンテナの shares 2 は、この系統でも 1 になる。将来 21 へ上げるなら、今からフラグを書いておく |
| 11.0.16 未満 | v2 を読めないことがある。shares はまだ使う | パッチを上げる。上げる前はフラグで人数を固定する |
| 11.0.16 以上 11.0.17 未満 | v2 は読める。shares はまだ使う | コンテナ cpu の書き方が結果を左右する |
| 11.0.17 以降 | 初期状態では shares を使わない。`+UseContainerCpuShares` で戻せる | フラグで N を指定する。戻しフラグは 21 で消えるので、長期の正にはしない |
| 17.0.5 未満 | 17.0.4 までは shares を使う世代がある | 更新番号を確認する。17.0.4 と 17.0.6 で `--cpu-shares` の挙動が違う、という報告が OpenJDK の議論にある |
| 17.0.5 以降 | 11.0.17 以降と同じ。初期状態は shares を無視 | `ActiveProcessorCount` を書く |
| 21、25 | shares を読むフラグが無い | `ActiveProcessorCount` が指定方法になる |

整数 vCPU は 1、2、4、8、16、32 です。0.25 と 0.5 は、どの JDK でも「0.25 個のプロセッサ」とは表現できません。`availableProcessors()` は int で、最小は 1 です。0.25 も 0.5 も、検出がうまくいっても 1 に見えます。フラグも 1 としか書けません。

これは、0.25 vCPU のタスクが 1 vCPU 分の時間を保証される、という意味ではありません。JVM の係の人数が 1 人になる、という意味です。保証が 0.25 で係が 1 人なら、係は保証より大きな前提で動きます。Fargate は余力があるときバーストできる、と 2026 年の AWS 記事が書いています。余力がある間は 1 人の係が働けます。ホストが混むと、保証の 0.25 まで戻ります。0.25 や 0.5 でフラグを 1 にしても、その性質は残ります。Java の常駐サービスを小さく始めるなら、その差を分かったうえで 1 vCPU を選ぶ方が、人数の説明は簡単です。

### コンテナ cpu とタスク cpu

Fargate ではタスクの `cpu` が必須で、コンテナの `cpu` は任意です。単一コンテナで、JDK 8 のように shares をまだ読む場合は、コンテナの `cpu` をタスクの `cpu` と同じにします。空欄のままにすると、shares 2 の経路が残ります。

複数コンテナがある場合、コンテナ cpu の合計はタスク cpu 以下です。JVM が読むべきなのは、その JVM のコンテナに割り当てた分です。サイドカーとアプリで 4 vCPU のタスクを分けているなら、アプリの `ActiveProcessorCount` を無条件に 4 にはしません。アプリのコンテナに残した vCPU の整数にします。

JDK 21 以降は shares を見ないので、コンテナ cpu を書いただけでは `availableProcessors()` は直りません。フラグが必要です。コンテナ cpu は、カーネルの配分としては引き続き意味があります。

## 起動パターンごとに、何が起きるか

| パターン | 新しい JVM か | 期待してよいこと |
| --- | --- | --- |
| タスク定義を作った日の、最初の 1 タスク | はい | その JDK の表どおり。最初だから正しい、とは限らない |
| 同じタスク定義で強制再起動、障害置換、デプロイの置き換え | はい | 前回と同じ数。設定を変えていなければ、正しい数にはならない |
| CPU 目標追跡でタスクが 2 台から 4 台になる | 増分ははい | 新しい 2 台も、元の 2 台と同じ人数を見る |
| タスク cpu を 1024 から 4096 に変えてローリングデプロイ | 新しいタスクははい | コンテナ cpu とフラグも 4096 と 4 に更新していれば、新しい JVM は 4 を使う。フラグが 1 のまま、または shares 2 のままなら、新しい JVM も 1 のまま |
| 生きているタスクの中で、cgroup ファイルだけを書き換える | いいえ | Fargate のオートスケールはこれをしない。実験としてやった場合、GC 係や共通プールは起動時の人数のままであることが多い |

「初期状態から設定している」が、タスク定義の cpu を最初から 4096 と書いている、という意味なら、それでも JDK とコンテナ cpu の組み合わせ次第で、最初の起動から 1 になります。

「初期状態から設定している」が、`-XX:ActiveProcessorCount=4` を最初から書いている、という意味なら、最初の起動から 4 です。再起動後も 4 です。台数を増やした新しいタスクも 4 です。ここでの「設定」は、Fargate の cpu フィールドと、JVM のフラグでは、別の設定です。

## 人数が違うと、何が遅くなるか

フラグはカーネルの配分を変えません。変えるのは、JVM が信じる人数です。カーネルは、これまでどおり shares と microVM の大きさで CPU 時間を配ります。

信じる人数が小さすぎるとき。

- ごみ集めの作業員が少なく、大きなヒープの回収が長引く。
- コンパイラのスレッドが少なく、起動直後の最適化が遅れる。
- `ForkJoinPool.commonPool()` が細く、平行処理をそこへ任せるライブラリが直列に近くなる。
- アプリケーションが `availableProcessors()` を起動時に一回だけ読んでプールを作っていると、そのプールも細いままです。
- 4 vCPU を買っても、スループットが 1 vCPU の近くに留まる。メトリクスの CPU 使用率は低いままに見えることがあります。余っているのは機械ではなく、Java が仕事を出していない状態です。

信じる人数が大きすぎるとき。

- スレッドスタック、ごみ集めの作業員、コンパイラが増える。
- EC2 起動タイプでは、CFS の硬い天井に多くのスレッドがぶつかり、スロットリングと遅延のスパイクになる。これが EC2 でフラグを勧める理由です。
- Fargate では CFS スロットリングは掛からない、と AWS の 2026 年記事にあります。大きすぎる人数は、余力がある間はバーストで吸収され、余力が無いときに待ち行列になります。0.25 や 0.5 で 1 と見るのは、この「少し大きい」側です。

起動時に人数が固まる代表例は、次です。

| 使う場所 | いつ人数を読むか | 小さすぎると |
| --- | --- | --- |
| ParallelGCThreads、ConcGCThreads | JVM 起動時のエルゴノミクス。一部の収集器は動的調整を持つ | 回収が長引く |
| CICompilerCount | 起動時 | 温まるまで遅い |
| ForkJoinPool 共通プール | クラス初期化のとき。後から CPU が増えても基本的に増やさない | 平行処理が細る |
| 多くのアプリの固定スレッドプール | 開発者が起動時に一度呼んだとき | そのプールの設計どおりに細る |
| Runtime.availableProcessors() 自体 | 呼ぶたびに答え得る。短いキャッシュがある | 呼ぶ側が保存していれば、保存した方が残る |

ヘルスチェックとの関係も、CPU 人数の話です。ごみ集めが長く止まると、アプリケーションは生きていても応答が遅れます。ECS のコンテナヘルスチェックは `interval`、`timeout`、`retries`、`startPeriod` です。AWS の記事は、チェックを緩めるより先に、CPU 人数を割当へ合わせることを勧めています。

## タスク定義へ、外から書く

イメージを作り直さずに人数を変える入り口が、タスク定義の環境変数です。HotSpot のランチャは `JAVA_TOOL_OPTIONS` を読み、起動ログに `Picked up JAVA_TOOL_OPTIONS` と出します。

単一コンテナで 4 vCPU の例です。

```json
{
  "family": "sample",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "4096",
  "memory": "8192",
  "containerDefinitions": [
    {
      "name": "app",
      "image": "example:latest",
      "cpu": 4096,
      "environment": [
        {
          "name": "JAVA_TOOL_OPTIONS",
          "value": "-XX:ActiveProcessorCount=4 -XX:MaxRAMPercentage=75.0"
        }
      ]
    }
  ]
}
```

対応表です。フラグは整数だけです。

| タスクの cpu | vCPU | ActiveProcessorCount | コンテナが JVM 一つならコンテナ cpu |
| --- | --- | --- | --- |
| 256 | 0.25 | 1 | 256 |
| 512 | 0.5 | 1 | 512 |
| 1024 | 1 | 1 | 1024 |
| 2048 | 2 | 2 | 2048 |
| 4096 | 4 | 4 | 4096 |
| 8192 | 8 | 8 | 8192 |
| 16384 | 16 | 16 | 16384 |
| 32768 | 32 | 32 | 32768 |

`MaxRAMPercentage` はメモリの話で、CPU 人数とは別のつまみです。同じ環境変数に並べられるので、例に入れています。ヒープを固定の `-Xmx` だけで書くと、タスクメモリを大きくしてもヒープが追従しません。CPU フラグも同じで、タスク cpu を変えたデプロイでは、環境変数の N を同じリビジョンで変えます。

エントリポイントがシェルスクリプトで、その先が `java` なら、環境変数は届きます。ランチャを通さない埋め込み JVM や、環境変数を消す起動スクリプトでは届きません。起動ログの一行で確認します。

タスクを大きくするたびに人の手で N を変えたくない場合は、起動スクリプトがメタデータから読む方法があります。ECS のタスクメタデータエンドポイント v4 は、タスクの Limits に CPU ユニットを持っています。

```sh
CPU_UNITS=$(python -c 'import json,os,urllib.request; print(json.load(urllib.request.urlopen(os.environ["ECS_CONTAINER_METADATA_URI_V4"]+"/task"))["Limits"]["CPU"])')
N=$(( CPU_UNITS / 1024 ))
if [ "$N" -lt 1 ]; then N=1; fi
export JAVA_TOOL_OPTIONS="${JAVA_TOOL_OPTIONS:-} -XX:ActiveProcessorCount=${N}"
exec java -jar /app/app.jar
```

このスクリプトは、新しいタスクが生まれたときに、そのタスク定義の CPU を整数へ切り捨てて紙に書きます。0.25 と 0.5 は 1 になります。動いている JVM を後から書き換えるものではありません。メタデータの形は、導入前に自分のプラットフォームで一度表示して確認します。

## 自分のタスクで確かめる手順

推測のままにせず、起動ログを見ます。

JDK 17 以降。

```text
java -XshowSettings:system -Xlog:os+container=trace -version
```

JDK 11。

```text
java -XshowSettings:all -Xlog:os+container=trace -version
```

JDK 8。

```text
java -XX:+UnlockDiagnosticVMOptions -XX:+PrintContainerInfo -XshowSettings:all -version
```

ログで見る行は、次です。

- Detected cgroups v1 または v2。
- CPU Quota。Fargate で「制限に使っていない」なら、無制限や -1 に見えることがあります。
- CPU Shares または CPU weight。
- active processor count。
- `Picked up JAVA_TOOL_OPTIONS` に、意図した N が入っていること。

アプリケーションの中では、起動直後に `Runtime.getRuntime().availableProcessors()` を一度ログへ出します。フラグが 4 なら 4 です。

`-XX:+PrintFlagsFinal` の `ActiveProcessorCount = -1` は、故障ではありません。-1 は「フラグ未指定。自動検出せよ」という初期値です。検出結果そのものは、このフラグ欄ではなく、`availableProcessors()` と container ログに出ます。

確認は、1 vCPU だけで終わらせません。1 は、誤認でも正解でも同じ表示になることがあるからです。2 vCPU または 4 vCPU のタスクで、ログの人数が 2 または 4 になることまで見ます。可能なら 0.25 か 0.5 でも一度見て、1 に丸まることを記録します。

## 混同しやすい隣の問題

メモリ検出は、CPU 検出と別のファイルを見ます。メモリがコンテナ限界と一致していても、CPU 人数が正しい証拠にはなりません。逆も同じです。

2025 年から 2026 年にかけて、JDK 21 の一部更新は、ECS の cgroup v1 で `memory.limit_in_bytes` が実質無制限で、実際の限界が `hierarchical_memory_limit` にだけある配置を、限界なしと読む回帰が報告されています。JDK-8370572 です。これはヒープサイズの話です。`ActiveProcessorCount` では治りません。メモリは `MaxRAMPercentage` の結果を起動ログで別に確認します。

Windows コンテナでは、タスクレベルの CPU とメモリのパラメータが無視される、と ECS の文書に注意があります。この文書の結論は Linux の Fargate についてです。

サービスオートスケールが JVM を理解しない、という 2026 年の別記事もあります。あれは、CPU 使用率やメモリ使用率のメトリクスが、ヒープや GC の内部状態を表さない、という話です。タスクの台数を増やす判断の話であり、1 台の JVM が自分の vCPU をいくつと数えるかとは別です。

## 実務の決定

Linux の Fargate で、JDK 11.0.17 以降、17.0.5 以降、21、25 を使うなら、タスクの整数 vCPU を `-XX:ActiveProcessorCount` に書き、タスク定義の環境変数で渡します。最初の起動から書きます。再起動や台数増加に任せません。

JDK 8u372 以降で、整数 vCPU かつコンテナ cpu も同じ値なら、自動検出が合う見込みは高いです。それでもフラグを書いておくと、1 vCPU の偶然の一致に頼らず、JDK 21 へ上げた日に人数が突然広がりません。

0.25 と 0.5 は、フラグを 1 にします。それで保証量が 1 vCPU になるわけではないことだけ、容量計画に残します。

タスク cpu を変えるリビジョンでは、同じ変更の中で N も変えます。ここを忘れると、スケールアップした新しいタスクが、紙に書いた古い人数のまま起動します。それは検出不良が再発したのではなく、指定が古いサイズのままです。

## 出典

- Amazon Web Services. "JVM memory, CPU, and classpath best practices for Java containers on AWS." Containers Blog. Fernanda Machado, Henrique Santana. 2026-07-20. https://aws.amazon.com/blogs/containers/jvm-memory-cpu-and-classpath-best-practices-for-java-containers-on-aws/
- Amazon Web Services. "Troubleshoot Amazon ECS task definition invalid CPU or memory errors." Amazon ECS Developer Guide. 2026-10-21 前後の内容。Fargate は CPU shares、EC2 は quota と period。https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-cpu-memory-error.html
- Amazon Web Services. "Amazon ECS task definition parameters." コンテナ `cpu` は Linux で CpuShares。Fargate では任意。2026-10 の文書。https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_definition_parameters.html
- Amazon Web Services. "Fargate platform versions for Amazon ECS." Linux の LATEST は 1.4.0。2026-10-20 前後。https://docs.aws.amazon.com/AmazonECS/latest/developerguide/platform-versions.html
- Amazon Web Services. "Automatically scale your Amazon ECS service." スケール対象はタスク数。2026-10-20 前後。https://docs.aws.amazon.com/AmazonECS/latest/developerguide/service-auto-scaling.html
- Amazon Web Services. "Amazon ECS announces faster service auto scaling." 2026-06-18。20 秒メトリクス。変えるものはタスク数。https://aws.amazon.com/about-aws/whats-new/2026/06/amazon-ecs-faster-autoscaling/
- Amazon Web Services. "How Amazon ECS manages CPU and memory resources." 2019-10-21。当時の「タスクは硬い天井、コンテナは shares」と、Fargate の分離境界。現行の quota 説明は 2026 年のガイドを優先する。https://aws.amazon.com/blogs/containers/how-amazon-ecs-manages-cpu-and-memory-resources/
- OpenJDK. JDK-8146115. Improve docker container detection and resource configuration usage. 登録 2015-12-23、解決 2017-11-16。8u191 のリリースノートに ActiveProcessorCount。https://bugs.openjdk.org/browse/JDK-8146115
- Oracle. JDK 8u191 リリースノート。2018-10-16。容器検出と ActiveProcessorCount。
- OpenJDK. JDK-8230305. cgroup v2。JDK 15。backport は 11.0.16 と 8u372。Red Hat Developer, Severin Gehwolf, 2022-04-19 と 2023-04-19。
- OpenJDK. JDK-8281181 / CSR JDK-8281571. Do not use CPU Shares to compute active processor count. 登録 2022-02-02、JDK 19 で 2022-03-04 に解決。backport は 18.0.2、17.0.5、11.0.17。https://bugs.openjdk.org/browse/JDK-8281181
- OpenJDK. JDK-8299037. 8u、11u、17u で UseContainerCpuShares の初期値を戻す提案。Won't Fix。
- OpenJDK. JDK-8282684. UseContainerCpuShares の廃止。JDK 21 で削除。
- OpenJDK. JDK-8227006. availableProcessors のコストとキャッシュ。人数の再読込が議論されている。
- aws/amazon-ecs-agent issue 1735. 2018-12。未設定 CPU が 2 shares になり、Java が 1 と読む。
- JJUG CCC 2022 Fall. 「Fargate上のJVMからCPUを認識するまで」. コンテナ CPU 未設定の Fargate で shares 2 をログ確認。https://speakerdeck.com/orekyuu/jjug-ccc-2022-fall-fargateshang-nojvmkaracpuworen-shi-surumade-zheng-sikuren-shi-sarenaicpunomi-wozhui-e
- OpenJDK. JDK-8370572. ECS の hierarchical memory limit。CPU とは別問題。2025-10 以降の報告。

## この調査で確認したことと、確認していないこと

確認したのは、2026-10-23 時点で公開されている AWS の文書とブログ、OpenJDK のバグ記録、2018 年の ECS エージェント issue、2022 年の JJUG 発表です。Fargate が quota を使わず shares を使うこと、現行 LTS の JDK 8 以外が shares を CPU 数に使わないこと、オートスケールがタスク数を変えること、ActiveProcessorCount が起動時の上書きであることは、これらの一次資料で一致しています。

この作業環境から Fargate へタスクは起動していません。2026 年 10 月の特定アカウントで、コンテナ cpu を空にしたときに shares がまだ 2 と書かれるかは、再計測していません。2022 年の Fargate 計測と、2026 年も残っている「コンテナ cpu は shares であり、2 未満は特別」という文書から、そのリスクは残っていると判断しています。自分のタスク定義では、上の起動ログで Shares と active processor count を一度見るのが最終確認です。

microVM の vCPU 数が、整数のタスク vCPU と常に一致するかは、AWS が「一致しないことがある」と書いている以上、常に一致するとは扱いません。一致するタスクを観測しても、フラグを省いてよい理由にはしません。フラグは観測値を固定するためのもので、観測の代わりではありません。
