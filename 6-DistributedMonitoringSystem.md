6 Monitoring Distributed Systems
==============================

この章は、人間をpageすべきものはなにか、pageするまでもないイシューをどのようにして扱うか、についてのガイドラインを示す。
(memo: pageはアラートに対する電話やメールによる呼び出しを表している。)

## Definitions (言葉の定義)

- `Monitoring`
- `White-box monitoring`
  - 内部システムから出力されたメトリックをベースにするモニタリングのこと。ログやJVM Profiling Interface、`/server/status`のようなHTTPハンドラを含む
- Black-box monitoring
  - ユーザ視点での外からみえる振る舞いのテスト
- Dashboard
- Alert 
  - `tickets`、`email alerts`、`pages` として分類される。
- Root cause
  - 根本原因
- Node and machine
- Push
  - 稼働中のソフトウェアや設定に対するあらゆる変更のこと。デプロイ。

## Why Monitor? (なぜ監視が必要か)

システムを監視する理由はたくさんある。

- 長期傾向の解析
- 時間比較とグループ比較
  - memcachedのヒットレートは、他のノードと比べてどれがよりよいか。先週と比べてサイトが遅くなっているか。
- アラート
- ダッシュボード作成
- デバッグ

- モニタリングとアラートは、システムがいつ壊れて、なにが壊れたのかを我々に教えてくれる。システムが自動的に問題を修正することができないとき、我々人間がアラートを調査し、当面の現実的な問題があるかどうかを決定して、問題を軽減し、問題の根本原因を特定する。なにか少しおかしいっぽいという理由だけで、アラートをトリガーすべきではない。

- 人間のpageは、非常に高価。勤務時間中のpageは従業員のワークフローを阻害する。従業員が家にいる場合、個人の時間や睡眠時間が阻害される。頻繁にpageされると着信アラートを無視するようになる。他の雑音が迅速な診断および修正を妨害するので、障害時間が長引くきうる。効果的な警告システムは、良好な信号と非常に低いノイズを持つ。

## Setting Reasonable Expectations for Monitoring (根拠のある例外を設定する)

- モニタリングそれ自体は重要なエンジニアリングの努力
- 10-12人のGoogleのSREチーム(memo:おそらく1サービスあたり10-12人)では、大抵1人か2人は彼らのサービスの監視システムに構築とメンテナンスにアサインされている
- モニタリング基盤の汎用化や共通化のため時間の経過とともに人数は減っていくものの、すべてのSREチームは最低1人はこの`monitoring person` を置いている

- GoogleのSREチームは、複雑な依存関係をもつヒエラルキであまり成功した経験がない。
- 「データベースが遅いことを知っているなら、遅いDBに向けてアラートする」というようなルールを使わない。遅くなっているウェブサイトに向けてアラートする。
- 依存に頼ったルールは大抵、DCからユーザトラフィックを排出するためのシステムのような非常に安定した部分に適している。
- 例えば、「もしDCからでていく場合、そのレイテンシについてはアラートしない」はDC共通のアラートルール。

- 低ノイズで高シグナルを保つために、モニタリングシステムの要素は非常にシンプルで堅牢である必要がある。人間のために生成するアラートは、明らかな失敗を理解し、表現するためにシンプルであるべきだ。

## Symptoms Versus Causes (症状と原因)

- モニタリングシステムは、「何が壊れたのか」「なぜ壊れたのか」の2つの質問を解決すべきである。
- 「何が壊れたのか」は症状(symptoms)を示す。「なぜ壊れたのか」は原因を示す。表6-1
- "What"と"Why"は最大シグナルで最小ノイズなよいモニタリングを書く上で非常に重要な区別の一つ。

## Black-Box Versus White-Box

- 我々は、ブラックボックス監視とホワイトボックス監視を組み合わせる。ブラックボックス監視は症状指向であり、システムが今正しく動いていないことを表す。ホワイトボックス監視は、ホワイトボックス監視は今にも起こりそうな問題を検出できる。

- マルチレイヤなシステムでは、1人の症状は他の人の原因となる。例えば、DBのパフォーマンスが遅いと過程する。遅いDBのreadがデータベースSRE向けの症状である。しかし、遅いウェブサイトを観測するフロントエンドSREにとっては、同じ遅いDBのreadが原因である。したがって、ホワイトボックスが情報量に応じて、ホワイトボックス監視はときどき症状指向で、ときどき原因指向である。 (memo: おそらく `/monitor/dbread` みたいなエンドポイントを生やして、ホワイトボックス監視すれば、症状だけでなく原因にたどりつきやすくなるという意味だと思う。)

## The Four Golden Signals (4つのゴールデンシグナル)

もし、4つのメトリックだけ測定するなら、これらの4つのにフォーカスせよ。

- レイテンシ
  - リクエストをサービスする時間。成功リクエストのレイテンシと失敗リクエストのレイテンシの区別は重要だ。DB接続できない500エラーは通常のリクエスト処理より高速なため、5xx系のレイテンシを全体に含めると、誤解を招く計算になるかもしれない。
- トラフィック
  - どれくらいの需要があるか測定することは、ハイレベルなシステム固有メトリックを測定していることを意味する。Webサービスにとって、この測定は大抵HTTP reqs/sである。オーディオストリーミングシステムにとっては、ネットワークI/Oレートか並列セッション数かもしれない。KVSにとっては、transactions/s かもしれない。
- エラー
  - 失敗したリクエストのレート。プロトコルのレスポンスコードは、すべての失敗状態を表現するには不十分であり、二次（内部）プロトコルが部分的な失敗をトラックするために必要かもしれない。
- 飽和(saturation)
  - どれくらいサービスが"full"なのか。ボトルネックとなるリソース量（メモリやI/O)。多くのシステムは、100% utilizationに到達する前にパフォーマンスがデグレードするので、utilization targetをもつことは必須である。 
  - 非常にシンプルなサービスにとっては負荷テストの値で十分かもしれない。しかし、ほとんどのサービスは、上限が知られているCPU利用率やネットワーク帯域のような間接的なシグナルが必要だ。レイテンシの増加は、代表的なsaturationの兆しである。小さなウィンドウサイズにおけるレスポンスタイムの99%ileは、saturationの早期のシグナルとなる。

## Worrying About Your Tail (or, Instrumentation and Performance)

- 最初から監視システムを構築する場合、平均レイテンシや平均CPU利用率など、平均値をベースに設計したくなる。毎秒1,000件のリクエストを100msの平均レイテンシでWebサービスを実行する場合、リクエストの1%は簡単に5秒かかることがある。ユーザーが自分のページをレンダリングするために、いくつかのウェブサービスに依存している場合、1バックエンドの99%ileは、簡単にフロントエンドの中央値になりうる。

- 遅い平均値と遅い"tail"を区別する簡単な方法は、実際のレイテンシよりは(ヒストグラムに適した)レイテンシによりバケットされたリクエスト数を収集すること。例えば、0-10ms、10-30ms、30-100ms、100-300ms それぞれの区間のリクエスト数。ほぼ指数関数的にヒストグラムの境界を分散させると、リクエストの分散を可視化できる。

## Choosing an Appropriate Resolution for Measurements (適切な解像度を選ぶこと)

- システムの異なる側面は、異なる粒度で測定されるべき。例えば
  - 分のスパンでCPU負荷を観察しても、スパイクを見逃してしまう
  - 一方で、年間でたった9時間のダウンタイム(99.9%)を目標としたウェブサービスのために、分単位で、200ステータスを再三調査することは、おそらく無駄である
  - 同じように、99.9%の可用性目標のサービスのために、HDDが満杯かチェックするのは無駄である。

- CPUを毎秒測定するのは興味深いデータが得られるかもしれないが、収集、格納、分析するのが高コストになることもある。サーバ内でサンプリングすると、コストを減らせる。それから外部システムでサーバ間の分布を時間をかけて解析する。
  - 1. CPU利用率を毎秒を計測
  - 2. 5%の粒度のバケットを使って、毎秒、適切なCPU利用率バケットを増やす
  - 3. これらの値を毎分集める

## As Simple as Possible, No Simpler (できるだけシンプルにする)

- 以上の要求をすべて組み込むと、とても複雑な監視システムができあがる。
  - 異なるメトリックで、異なるパーセンタイルで、異なるレイテンシ閾値でのアラート
  - 可能性のある原因を検出するための余分なコード
  - 可能性のある原因のそれぞれに対して関連したダッシュボード

- したがって、シンプルにするという視点で監視システムを設計する。監視するために何を選択するかは、以下のガイドラインに従う。
  - 実際のインシデントを捉えるルールは、大抵、できるだけシンプルで、予測可能で、信頼性があるべき。
  - めったに実行されない(4半期に一回を下回る)データの収集、集約、アラート設定は、除去されるべき。
  - 収集したが、どのアラートにも使用されず、どの(prebaked)ダッシュボードにも表示されない信号は、除去候補

- Googleの経験では、アラートとダッシュボードをペアにした、メトリックの基本的な収集・集計は、比較的スタンドアローンなシステムとしてうまく働く。(実際、Googleの監視システムは、いくつかのバイナリに分かれているが、基本的に人々はこれらのバイナリのすべての側面について学んでいる)
システムプロファイリングやシングルプロセスデバッグ、負荷テスト、クラッシュ解析、ログ収集と解析、トラフィック検査のような、モニタリングとその他の機能を組み合わせたくなる誘惑に駆られる。この手のシステムは、過度に複雑であまりに多くの結果をブレンドしてしまう。

## Tying These Principles Together (これらの原則を結びつけること)

- この章で説明する原則は、GoogleのSREチーム内で広く承認された監視とアラートの哲学と紐付けることができる。
- 以下の質問に答えると、false positives と page爆発を避けられる。
  - このルールは緊急で、アクション可能で、ユーザにみえる、このルールでなければ検出されない状態を検出するか？
  - 私はこのアラートを無視することができて、それが良性だと知っているか？私はこのアラートをいつ無視できてなぜ無視できるだろうか？どのようにしてこのシナリオを避けられるだろうか？
  - このアラートは間違いなく、ユーザーがマイナスの影響を受けていることを示しているか？ユーザーがマイナス影響を受けていないことを検出可能なケースはあるか？トラフィックの排出やテストデプロイメントなどフィルタすべきか？
  - 私は、このアラートに反応してアクションできるか？そのアクションが緊急か？もしくは朝まで待てるのか？そのアクションは安全に自動化できたか？そのアクションは長期的な修正になるのか？もしくは単に短期的なワークアラウンドになるのか？
  - 他の人々がこのイシューに対してpageされるか？  therefore rendering at least one of the pages unnecessary?

- 以下の質問は、pagesとpagerの基本的な哲学を反映している。
  - pagerが鳴り響くたびに、私は切迫感をもって反応できるべきか？私は疲労する前に、1日のうち数回、切迫感をもって反応できる
  - すべてのpageはアクション可能であるべき
  - すべてのpageのレスポンスは知性が必要であるべき。もしpageが単に機会的な反応になるなら、pageであるべきではない
  - pageは新規の問題か、これまでにはみられなかったイベントであるべき

## Monitoring for the Long Term (長期間の監視)

- 監視システムは、進化し続けるシステムをトラックする。現在、自動化することが稀で難しいアラートが頻繁に発生するかもしれない。この時点で、誰かがその問題の根本原因を見つけて排除すべき。もしそのような解決が不可能なら、そのアラート対応は完全に自動化する価値がある。
- 監視に関する意思決定が長期的なゴールに沿って為されることが重要。今日起こるすべてのpageは明日のシステムを改善から人間の目をそらす。したがって、長期の見通しを改善するために、可用性やパフォーマンスを短期的に"hit"することをとるケースもしばしばある。 (memo: "hit" は犠牲にする？)

以下2つのケーススタディ。

### Bigtable SRE: A Tale of Over-Alerting

- Googleの内部インフラストラクチャは、SLOにそって提供され、測定される。数年前に、BigtableのSLOが、振る舞いのよいクライアントの平均パフォーマンスをベースにしていた。Bigtable内の問題と、ストレージスタックの下位レイヤの問題のせいで、平均パフォーマンスには、"large tail"が含まれていた。リクエストの最悪値5%は、残りの部分より大幅に遅かった。

- SLOが近づくと、メールがトリガされ、SLPを超えたときにpagingアラートがトリガされた。両方のタイプのアラートが大量にトリガされ、許容できないエンジニアリング時間を消費した。チームはかなりの時間を費やして、本当にアクション可能な数少ないアラートを見つけるためにアラートを分類した。そして、我々はユーザ影響のあった問題を見逃した。多くのpageは緊急ではなく、インフラストラクチャのよく理解されていた問題のせいで、機会的な対応か未対応になった。

- 状況を改善するために、チームは3方面からアプローチした。Bigtableのパフォーマンスを向上させるために多大な努力をしながら、75%tileのレイテンシを使って、一時的にSLOをdialed backした（memo: SLOをゆるくした？)。診断するために時間を使いすぎるメールアラートを無効にした。

- この戦略によって、戦術的な問題を常に修正するよりも、長期の問題を修正する余力がでてきた。最終的には、一時的にアラートをback offすることで、よりよいサービスに向けて速く進歩できた。

### Gmail: Predictable, Scriptable Responses from Humans

- Gmailの非常に初期の頃、ワークキューと呼ばれる分散プロセス管理システム上に構築された。ワークキューは、スケジューラ内の比較的不透明なコードベース内の特定のバグの解決が難しい状況だった。その時点で、Gmail監視は、タスクがワークキューにより重複スケジュールされたときにアラートが発火するように、構築されていた。しかし、その時点で数千のタスクを持っていたので、そのようなアラートは保守不能だった。

- この問題を解決するために、GmailのSREはユーザ影響を最小限にするツールを作った。よりよい長期的な解決に達するまでに、チームでは、問題を検出してリスケジューラをつつくループ全体を単に自動化するべきかどうか議論があった。しかし、何人かはこのワークアラウンドにより、真の解決が遅れるのではないか心配した。

- この種の不安はチームの一般的なものであり、チームの自己規律の根底にある不信感を反映している。適切な修正のために時間をかせぐために、"hack"を実装したい一方で、他の人がハックの内容を忘れるか、適切な修正を無期限延期になるかを心配する。"hack"により技術的負債の層が積み上がるのは簡単なので、この関心には説得力がある。

- 機会的な対応をするpage"は危険信号でなければならない。そのようなpageを自動化するのに気が進まないのは、技術的負債の回収ができるという自信がチームに欠けていることを示唆している。これはエスカレートする価値のある大きな問題。

### The Long Run

BigtableとGmailの例は、短期間と長期間の可用性の間の摩擦、という共通のテーマに結びつく。努力の強制により、ガタガタなシステムを高可用にできる。しかし、これは大抵、短命で少数の英雄的メンバーのバーンアウトと依存をはらむ。制御された、短期的な可用性を減らすことは、しばしば痛みを伴う。しかし、システムの長期の安定性のための戦略的なトレードとなる。

## Conclusion

- 健全な監視とアラートはシンプルで、推論することが簡単です。健全な監視は、問題をデバッグするための助けとするために、原因志向のヒューリスティックをpageするための症状にフォーカスする。症状を監視することは、saturationの監視とサブシステムのパフォーマンスを監視を通して、 監視するスタックをより"up"するのが簡単になる。メールアラートは価値が限られ、ノイズが多すぎになりがちである。代わりに、進行中のサブクリティカルな問題をすべて監視するダッシュボードを使うべき。ダッシュボードはヒストリカルな相関性を解析するためにログと一緒になっているかもしれない。

