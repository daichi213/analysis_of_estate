# **賃貸物件時系列データ蓄積および通知システムの基本設計ドキュメント**

## **システムアーキテクチャの基本構想と設計原則**

本システムは、UR都市機構などの公式賃貸ウェブサイトからスクレイピング技術を用いて空室情報を定期的に取得し、エンドユーザーの希望条件（エリア、家賃帯、間取り等）に合致する物件が募集された際の即時通知機能を提供するとともに、取得したデータを時系列の履歴として蓄積し、不動産投資に向けた高度なトレンド分析を可能にする統合プラットフォームである。  
アーキテクチャの基本方針は、第一に「非同期かつスケーラブルなデータ収集基盤」、第二に「法務および対象サーバーへの負荷を最小化するコンプライアンス駆動型設計」、第三に「過去の任意の時点における物件の空室状態を完全に再現できる時系列データベース（Slowly Changing Dimension Type 2）の構築」である。賃貸物件の空室情報は極めて流動的であり、募集が開始されてから数時間から数日で埋まることも珍しくない。とりわけUR賃貸は先着順のシステムを採用しており、24時間365日いつでも公式サイトで空き状況の確認が可能であるため、引越しシーズン等においては物件の確保競争が激化する1。  
したがって、データ収集は一定の高い頻度を保つ必要がある一方で、過剰なアクセスによる対象システムへの業務妨害リスクを完全に排除しなければならない。さらに、単なる現在の空室状況の把握にとどまらず、将来的な賃貸物件のトレンド分析や空室率の算出といった不動産投資のためのメトリクス抽出を目的とするため、スナップショットデータの単純な上書き（UPSERT）ではなく、変更履歴を完全に保存しデータの系譜を追跡可能なデータウェアハウス設計が求められる3。

## **コンプライアンスおよびスクレイピングにおける法的リスク管理**

スクレイピングシステムの設計において技術選定よりも先行して検討すべきは、対象サイトおよび関連法規に対する厳格なコンプライアンスの遵守である。ウェブスクレイピング自体は違法な行為ではないものの、目的や実行態様、取得したデータの取り扱いによっては、複数の法律や契約に抵触する複合的な法的リスクを内包している5。

| 関連法規・規範 | リスクの性質 | 本システムにおける具体的な対策方針 |
| :---- | :---- | :---- |
| **著作権法（第30条の4）** | 複製権の侵害7 | 享受目的の画像転載を避け、事実データ（価格・面積等）の情報解析に限定する8。 |
| **刑法（偽計業務妨害罪等）** | サーバー過負荷による業務妨害6 | 厳格なレートリミット、指数的バックオフ、並行アクセスの制限をアーキテクチャレベルで強制する8。 |
| **不正アクセス禁止法** | 認証や技術的制限の不正な回避10 | ログインが必要な領域に対するスクレイピングや、CAPTCHA等の意図的な迂回を避ける8。 |
| **個人情報保護法 (APPI)** | 個人データの不正取得・第三者提供5 | 取得対象を物件という「モノ」のデータに限定し、個人の氏名や要配慮個人情報を収集しない8。 |
| **利用規約（民事上の契約）** | 損害賠償、IPブロック6 | 自動収集を禁止する規約が存在する場合の法的リスクを認知し、運用上の負荷低減措置を徹底する11。 |

日本の著作権法においては、2018年の改正により第30条の4（著作物に表現された思想又は感情の享受を目的としない利用）が新設され、情報解析やAIの学習データとしてのスクレイピングが原則として権利者の許諾なく適法化されている5。ウェブサイト上のデータ（価格、所在地、床面積、築年数、物件種別など）は事実ベースの事業データであり、創作的な表現を味わう「享受」を目的とせず、トレンド分析や空室率の算出といった情報解析を目的とする限り、著作権法上の複製権（第21条）の侵害には当たらない可能性が高いと解釈される7。ただし、物件の画像や独創的な説明文をそのまま再公開するような行為は、享受目的とみなされ第30条の4の適用外となるため、本システムでは事実データのみを抽出・解析対象とする7。また、但書にある「権利者の利益を不当に害する場合」に該当しないよう、有償で提供されているデータベースの代替となるような市場を侵食する利用形態は避ける必要がある12。米国におけるフェアユース（変容的利用）の概念とは異なり、日本法では非享受目的か否かが適法性の分水嶺となる12。  
さらに深刻なリスクとして、過剰なアクセス頻度による刑法上の偽計業務妨害罪（第233条）や電子計算機損壊等業務妨害罪に問われる可能性が挙げられる6。2010年に発生した岡崎市立中央図書館事件（Librahack事件）では、純粋な技術的興味による個人プロジェクトであったにもかかわらず、高頻度なアクセスが図書館の蔵書検索システムの障害を引き起こしたとして、開発者が逮捕される事態となった13。この事例が示唆するのは、悪意の有無やDoS攻撃の意図がなかったとしても、結果として対象システムの運営を妨害すれば刑事リスクが生じるという点である13。また、UR賃貸の利用規約第4条においては「本サービスに対するスクレイピング、クローリング等の自動的なデータ収集行為」が明確に禁止されている11。利用規約は法令ではないものの、これに違反した場合は民事上の損害賠償請求やIPアドレス単位でのアクセス遮断といったリスクが顕在化するため、事業継続性の観点から致命的なインシデントとなり得る6。  
したがって、本システムのクローラー設計においては、対象サーバーへの負荷を人間のブラウジングと同等レベルに抑えるための厳格なレートリミット、指数的バックオフ（Exponential Backoff）、および並行アクセス数の制限（Concurrency Limit）を実装することが不可欠である。さらに、技術的に閲覧できることと適法に取得できることは別であるという認識のもと、不正アクセス禁止法（UCAL）に抵触する恐れのある認証の突破やアクセス制御の回避は行わない設計とする8。

## **サーバーレス・スクレイピング基盤のインフラ設計**

データの収集には、ヘッドレスブラウザを用いた自動化フレームワークであるPlaywrightを採用し、これをコンテナ化してパブリッククラウドのサーバーレス環境（Google Cloud Run等）で稼働させるアーキテクチャを基本とする。

### **Playwrightの優位性とアーキテクチャ選定**

動的なJavaScriptによってレンダリングされる現代のウェブサイトからデータを抽出するためには、単なるHTTPリクエスト（RequestsやBeautifulSoup等）では不十分であり、完全なブラウザ環境をシミュレートしてDOMツリーを構築する必要がある16。この領域では歴史的にSeleniumが広く利用されてきたが、本システムにおいてはMicrosoftが開発するPlaywrightを採用する17。Playwrightは、Seleniumと比較して起動が高速であり、クラウドのサーバーレス環境におけるリソース効率が約15〜25%優れている17。また、非同期アーキテクチャ（Asynchronous Architecture）を標準でサポートしており、動的コンテンツに対する自動待機（Auto-wait）機能が充実しているため、リバースエンジニアリングを行うことなく安定したスクレイピングが可能となる17。  
インフラストラクチャとしては、AWS LambdaとGoogle Cloud Runの二つの選択肢が主流である。AWS Lambdaはアイドル時のコストがゼロであるという利点があるものの、最大実行時間が15分に制限されており、広範なエリアの物件情報をサーバーに負荷をかけずにゆっくりとクロールする要件とは相性が悪い場合がある19。一方、Google Cloud Runは最大実行時間を長く（デフォルトで600秒、最大で60分等）設定でき、並行処理の制御も柔軟である21。

| クラウド環境 | 最大実行時間 | デプロイメント方式 | コンテナの起動とアイドル時のコスト制御 |
| :---- | :---- | :---- | :---- |
| **AWS Lambda** | 15分 | コンテナイメージまたはZip \+ Layer | イベント駆動型。アイドル時の課金はなし。ブラウザ起動時のコールドスタートに課題あり20。 |
| **Google Cloud Run** | 最大60分 | コンテナイメージ | \--min-instances 0を設定することでアイドル時の待機コストを完全にゼロ化可能21。 |
| **Browserless.io等** | サービス依存 | WebSocket経由でのリモート実行 | サードパーティのクラウドブラウザ。インフラ管理が不要でスケーラビリティが高い22。 |

コスト最適化と運用保守のバランスを鑑み、本システムではGoogle Cloud Runを採用し、Cloud Schedulerを用いて定期的にCloud RunのエンドポイントにHTTP POSTリクエストを送信してタスクをトリガーするアーキテクチャを構築する21。

### **Dockerコンテナの最適化とシステム制約**

Cloud Run上でPlaywrightを安定稼働させるためには、公式のDockerベースイメージ（例：mcr.microsoft.com/playwright/python:v1.53.0-jammy などのUbuntu LTSベースのイメージ）を利用してコンテナを構築することがベストプラクティスである24。Dockerfileの作成においては、依存パッケージのインストールとブラウザバイナリの取得プロセスを最適化する必要がある24。  
スクレイピング対象のサイトが信頼できない場合や、セキュリティのサンドボックス機能（Chromiumのサンドボックス）を有効にする必要がある場合は、コンテナ内でデフォルトのrootユーザーを使用せず、専用の非rootユーザーを作成し、セキュアコンピューティングモード（seccomp）のプロファイル（seccomp\_profile.json）を適用することが推奨される21。また、Chromiumは大量のメモリを消費してクラッシュする可能性があるため、Docker実行環境においてはプロセス間通信の名前空間をホストと共有する \--ipc=host オプションの付与、または十分な共有メモリ（shm）の確保が重要となる24。さらに、プロセスPIDが1になることによるゾンビプロセスの発生を防ぐため、--init フラグに相当するプロセス管理（dumb-init等の活用）を考慮する24。  
Cloud Runのデプロイメントパラメータとしては、コスト削減の要となる \--min-instances 0 の設定に加え、想定外のトラフィックによるスケールアウトを防ぐための \--max-instances の制限、1コンテナあたりのリクエスト処理数を制限する \--concurrency 1、および最低1GiB（推奨は2GiB以上）のメモリ割り当てを実施する21。

## **ボット検知の回避とステルス実装の技術的考察**

現代のウェブサイトの多くは、Cloudflare、DataDome、PerimeterXなどの強力なアンチボットシステム（WAF）を導入しており、自動化されたヘッドレスブラウザからのアクセスをミリ秒単位の速度で検知し、ブロック画面やCAPTCHAによる遮断を実行する23。

### **ヘッドレスブラウザ特有のフィンガープリント**

デフォルトのPlaywrightやPuppeteerは、通常のユーザーが利用するブラウザとは異なる特有の痕跡（フィンガープリント）を残す。アンチボットシステムはこれらを複合的に分析することで、自動化ツールを正確に識別する27。

| 検知ベクトル | デフォルトのPlaywright（Headless） | 本物のブラウザ（人間による操作） |
| :---- | :---- | :---- |
| **navigator.webdriver** | true（自動化ツールであることを宣言） | false または未定義26 |
| **navigator.plugins** | 要素数0（空の配列） | PDF Viewerなど複数のプラグインが存在26 |
| **ユーザーエージェント** | HeadlessChrome という文字列が含まれる | 通常のOSおよびブラウザ識別子27 |
| **WebGLレンダラー** | SwiftShader（ソフトウェアレンダリング） | 実際のGPUハードウェア名（例: NVIDIA, Apple M1）26 |
| **CDP（Chrome DevTools Protocol）** | chrome.runtime や window.cdc\_ などの変数が漏洩 | 存在しない26 |
| **TLSフィンガープリント** | 自動化ライブラリ固有のJA3/JA4シグネチャ | 一般的なブラウザエンジンのシグネチャと一致26 |

### **ステルス拡張と回避技術の実装**

これらの検知を回避するために、Python環境においては playwright-stealth（Node.js環境においては puppeteer-extra-plugin-stealth）などのサードパーティ製パッケージが利用される23。このパッケージは、自動化のシグナルとなるJavaScriptオブジェクトをページロード前にパッチ（初期化スクリプトとして注入）し、navigator.webdriver を上書きし、偽のプラグイン情報を構築し、WebGLの応答を偽装することで、ブラウザのフィンガープリントを人間に近づける処理を行う26。  
さらに高度な対策として、プロキシプロバイダが提供する住宅用IP（Residential Proxies）を活用したIPローテーションが挙げられる23。データセンターのIPアドレスはアンチボットシステムによって静的にスコアリングされ、容易にブロックされる傾向にあるため、一般家庭のISPから割り当てられたIPを経由することで、トラフィックを分散しIPベースのレート制限を回避する23。加えて、マウスの自然な移動シミュレーション、ランダムなタイピング遅延、人間らしいスクロール動作といった行動的生体認証（Behavioral Biometrics）の模倣も、検知リスクを低減する上で有効な手段となる28。  
しかしながら、前述の法的リスク管理の観点と照らし合わせると、アンチボットシステムを高度な偽装によって突破する行為は、サイト管理者の明確なアクセス拒否の意思（技術的保護措置）を意図的に迂回するものであり、法的・倫理的なリスクを孕んでいる8。したがって、本システムにおけるステルス技術の適用は「悪意のあるDDoS攻撃やスクレイピングと誤認されて不要なIPブロックを受けることを防ぐための最低限の偽装」にとどめるべきである。根本的な対策としては、プロキシを用いて制限を回避すること以上に、アクセス間に十分かつランダムな遅延（スリープ）を挿入し、リクエスト頻度自体を低減する「振る舞い」による配慮を最優先するアーキテクチャを採用する28。

## **データレイクハウスとSlowly Changing Dimension Type 2**

本システムの核心的な価値は、現在募集中の物件情報を通知するだけでなく、将来の不動産投資分析に向けた「過去の空室履歴の完全な復元」を可能にすることにある。標準的なリレーショナルデータベースにおける正規化されたテーブルでの上書き（UPSERT処理）では、新たな募集状況を取得した際に古いステータスが失われ、「特定の物件がいつからいつまで空室であったか」という時系列の事実が喪失してしまう。

### **SCD Type 2による履歴のモデリング**

この問題を解決するため、データウェアハウスやディメンショナル・モデリング（Kimball手法）のベストプラクティスである Slowly Changing Dimension Type 2 (SCD Type 2\) パターンを採用する3。SCD Type 2は、属性（家賃、空室状況など）に変更が生じるたびに既存のレコードを上書きするのではなく、新しいレコードを挿入することで、データ全体の履歴と状態遷移を完全に保持するデータモデリング技術である3。  
具体的には、物件情報や空室情報を格納するディメンションテーブルに、有効期間の開始日を示す valid\_from、有効期間の終了日を示す valid\_to、および現在の最新レコードであるかを示すフラグ is\_current\_record の3つのカラムを追加する3。

| property\_id | room\_number | rent\_price | valid\_from | valid\_to | is\_current\_record |
| :---- | :---- | :---- | :---- | :---- | :---- |
| UR-101 | 201号室 | 85,000円 | 2026-03-01 09:00:00 | 2026-03-15 14:30:00 | FALSE |
| UR-101 | 305号室 | 90,000円 | 2026-03-10 10:00:00 | 2999-12-31 23:59:59 | TRUE |
| UR-101 | 201号室 | 82,000円 | 2026-08-01 11:00:00 | 2999-12-31 23:59:59 | TRUE |

上記のスキーマ設計が示唆する通り、201号室は2026年3月1日に空室として登録され、3月15日に募集が終了（入居者が決定、または募集取り下げ）したことがわかる。その後、同年の8月1日に家賃が82,000円に値下げされて再度募集が開始されたという一連の履歴が、一つのテーブル内で完全に追跡可能となる。分析アナリストやBIツールは、このテーブルに対して特定の日付（AS OF 日付）を指定し、WHERE valid\_from \<= '対象日時' AND valid\_to \> '対象日時' というSQLを発行するだけで、過去の任意の時点における募集物件一覧（スナップショット）を正確に再構築できる4。

### **データベースエンジンの選定とデータパイプライン**

時系列データの蓄積と高度なSQLクエリの実行を両立させるため、データベースエンジンにはPostgreSQLと、その時系列拡張であるTimescaleDBを採用する。TimescaleDBは、標準的なSQLインターフェースを提供しながら、背後でデータを時間と空間に基づいて自動的にパーティショニングする「ハイパーテーブル（Hypertable）」構造を持つ36。ハイパーテーブルを利用することで、数百万件から数億件規模に膨れ上がる時系列のスナップショットデータに対しても、インデックスの劣化を防ぎ、高速な読み書きを実現できる36。  
より大規模なデータレイクハウスアーキテクチャ（Data Lakehouse Architecture）を志向する場合、抽出されたJSONやCSVのローデータをクラウドストレージ（Amazon S3やGoogle Cloud Storage）上の「Rawゾーン」にParquetフォーマットで保存し、dbt (data build tool) やApache Sparkを用いて変換処理を行い、「Curatedゾーン」にSCD2形式のテーブルとしてロードする設計が考えられる3。Parquetは列指向（Columnar）のファイルフォーマットであり、高圧縮かつAmazon AthenaやGoogle BigQueryでのスキャンベースのクエリを極めて安価かつ高速に実行できる特長を持つ38。DatabricksのDelta Lakeを使用する場合、Change Data Feed (CDF) 機能を利用することで、レコードの変更履歴（CDC: Change Data Capture）を自動的にキャプチャし、SCD2テーブルの構築を透過的に行うことも可能である3。

## **希望物件の通知システム設計とレジリエンス**

抽出されたデータは、SCD Type 2に基づいてデータベースに記録されると同時に、ユーザーが事前に設定した希望条件（指定のエリア、広さ、家賃の上限下限、駅からの徒歩分数など）と照合される。合致する新規物件が発見された場合、即座に通知システムがトリガーされる。

### **重複通知の防止と状態評価**

通知システムにおいて最も留意すべきユーザーエクスペリエンス上の課題は、同一物件に対する繰り返しの重複通知（スパム）の防止である。クローラーが1時間ごとに実行される場合、単純なロジックでは募集中の物件が毎時間通知されてしまう。この問題は、前述のSCD Type 2モデルを活用することで極めてエレガントに解決できる。通知モジュールはデータベースにデータを挿入するプロセスと連携し、「新たに is\_current\_record \= TRUE として挿入されたレコードであり、かつ直前の状態が空室ではなかった物件」のみを差分として抽出し、通知の対象とする。

### **APIのレート制限とサーキットブレーカー**

エンドユーザーへの通知チャネルとしては、利便性の高いLINE Messaging APIやSlack webhook、電子メールなどを想定する。これらの外部APIをコールする際には、APIプロバイダが設定するレート制限（Rate Limit）を考慮する必要がある39。例えば、LINE Messaging APIにおいては、一定期間内に許可されるリクエスト数に上限が設けられており、短時間に過度な負荷をかけると制限（HTTPステータスコード429 Too Many Requests）を受ける39。大規模な団地で退去のタイミングが重なり、数十件の新規空室が一斉に検知された場合、一度に大量の通知リクエストを同期的に送信するとAPI側でブロックされるリスクがある。  
このリスクを回避し、システムのレジリエンス（耐障害性）を高めるため、通知モジュールには非同期メッセージキュー（Google Cloud TasksやAWS SQS）とサーキットブレーカー（Circuit Breaker）パターンを実装する40。外部APIの呼び出しにおいて一時的なネットワークエラーやレート制限に抵触した場合は、即座に失敗とするのではなく、リクエストをキューに退避させ、指数的バックオフ（Exponential Backoff：再試行の間隔を段階的に長くする手法）を用いて安全な頻度で再試行（Retry）を行うアーキテクチャが求められる40。これにより、外部システムへの依存による障害の連鎖を防ぎ、通知の確実な到達を担保することができる。

## **不動産データ分析と投資メトリクスの抽出**

蓄積された時系列データは、単なる通知のトリガーとして消費されるだけでなく、不動産投資判断のための強力なインテリジェンス（Business Intelligence）として機能する。本システムによって構築されたSCD2データベースからは、市場における様々な重要メトリクスを算出することが可能である。

### **空室率（Vacancy Rate）の多角的な算出**

不動産投資において、物件の収益性や運営の健全性を測る最も重要な指標の一つが空室率である。空室率の算出には、主に「時点空室率」と「稼働空室率」の二つの計算アプローチが存在し、分析の目的に応じて使い分ける必要がある42。

| 指標名 | 算出ロジック | 特徴と分析における用途 |
| :---- | :---- | :---- |
| **時点空室率** | (ある時点の空室戸数 ÷ 総戸数) × 100 | 最も単純な計算方法。特定の日（月末等）の瞬間の状態を示すため、入退去のタイミングによるブレが大きい42。 |
| **稼働空室率** | (年間の空室日数の合計 ÷ (総戸数 × 365日)) × 100 | 時間軸を加味したより実態に近い正確な数値。投資利回りのシミュレーションや、物件の実力を評価する際に用いられる43。 |

時点空室率は計算が容易であるが、特定の日付における瞬間風速に過ぎず、投資判断の指標としては稼働空室率の方がより実態に近い正確な数値とされる43。本システムでは、SCD Type 2によって物件ごとの空室開始日時（valid\_from）と空室終了日時（valid\_to）が履歴として記録されているため、この日時の差分をとることで各部屋の「空室日数」をミリ秒単位の精度で正確に算出できる46。  
具体的な計算例として、全部で20室あるマンションにおいて、3室がそれぞれ3ヶ月間（91日間）空室であった場合、稼働空室率は (3室 × 91日) / (20室 × 365日) \= 約3.7% と算出される46。このデータを単一の物件にとどまらず、特定の行政区全体や、特定の物件種別（例：築浅の1K、築古のファミリー向け等）に拡張して集約・計算することで、「どのエリアのどの間取りが、年間を通じて最も稼働率が高く、空室リスクが低いか」という高度なマクロ分析が可能となる。

### **市場トレンド分析とDays on Market（DOM）の可視化**

長期間にわたるデータの蓄積は、不動産賃貸市場の季節性やトレンドの構造的な変化を浮き彫りにする。

* **平均掲載期間（Days on Market）の分析**：募集が開始されてからウェブサイト上から削除される（＝成約または募集停止）までの期間（DOM）の中央値を算出することで、そのエリアの「賃貸需要の強さ（市場の流動性）」を定量的に測ることができる。掲載期間が極端に短い物件の特性（例：特定の駅からの距離、特定の間取り、階数）を機械学習やクラスタリングを用いて分析することで、投資利回りが最も高くなる優良物件の条件を逆算して定義できる。  
* **募集家賃の季節性とトレンド分析**：同じ物件であっても、需要が高まる引越しシーズン（2月〜3月）と、需要が落ち込む閑散期（8月など）では、家賃設定や敷金・礼金といった初期費用の条件が変動する傾向がある。SCD Type 2の履歴を横断的に分析することで、不動産管理会社（UR都市機構を含む）における賃料改定のアルゴリズムを推定したり、季節ごとの最適な入居タイミング（または退去・再募集のタイミング）を予測するモデルを構築することが可能となる。

## **結論**

本報告書で詳述したシステムアーキテクチャは、エンドユーザーに対するリアルタイムな価値（希望物件の即時通知）と、不動産投資家や市場アナリストに対する中長期的な価値（時系列データに基づく市場分析とメトリクス抽出）を同時に提供する堅牢かつスケーラブルな基盤である。  
本システムの設計における要諦は以下の3点に集約される。 第一に、法的リスクを最小化し事業継続性を担保するコンプライアンス駆動型の運用である。著作権法第30条の4に基づくデータ解析の適法性を担保しつつも、利用規約違反や刑法上の業務妨害罪といったリスクを避けるため、クローラーは対象サーバーへの配慮（適切なインターバル、エラー時の待機、極端なアクセス集中を避けるスケジュール設定）を最優先した振る舞いをアーキテクチャレベルで強制されなければならない。 第二に、PlaywrightとCloud Runを用いたスケーラブルかつコスト効率の高いデータ抽出基盤の構築である。Dockerコンテナ技術により環境依存を排除し、サーバーレスアーキテクチャにより待機コストを極小化しつつ、JavaScriptレンダリングが必要な最新の動的ウェブサイトを確実に処理する。 第三に、Slowly Changing Dimension Type 2 (SCD2) とTimescaleDB（あるいはクラウドデータレイクハウス）を組み合わせたディメンショナル・モデリングである。データの状態変化を上書きせず「有効期間」として履歴保存することで、重複通知の完全な排除、稼働空室率の正確な算出、そして過去の任意の時点における市場スナップショットの確実な再現を可能にする。  
これらの設計原則に忠実に沿ってシステムを実装することで、単なる一時的なスクレイピングツールを超えた、極めて価値の高い不動産データ・インテリジェンス基盤が実現される。

#### **引用文献**

> 1. UR賃貸の空き状況を確認する方法【2026年最新・最速で空室 ... \- note, [https://note.com/monk\_mode\_youcan/n/ne64536311ee0](https://note.com/monk_mode_youcan/n/ne64536311ee0)  
> 2. 【2026年最新】UR賃貸の空室待ちを効率化する方法, [https://kuushitsu-info.jp/blog/ur-vacancy-wait](https://kuushitsu-info.jp/blog/ur-vacancy-wait)  
> 3. Practical Guide To Slowly Changing Dimensions Type 2 In DBT Part 1, [https://xebia.com/blog/a-practical-guide-to-creating-slowly-changing-dimensions-type-2-in-dbt-part-1/](https://xebia.com/blog/a-practical-guide-to-creating-slowly-changing-dimensions-type-2-in-dbt-part-1/)  
> 4. Data point versioning infrastructure for time traveling to a precise, [https://www.reddit.com/r/dataengineering/comments/vf8cii/data\_point\_versioning\_infrastructure\_for\_time/](https://www.reddit.com/r/dataengineering/comments/vf8cii/data_point_versioning_infrastructure_for_time/)  
> 5. 【IT弁護士監修】スクレイピングは違法？法律に基づいて徹底解説, [https://pig-data.jp/blog\_news/blog/scraping-crawling/scrapinglaw/](https://pig-data.jp/blog_news/blog/scraping-crawling/scrapinglaw/)  
> 6. スクレイピングとは？仕組み・違法性・活用例まで実務目線で徹底, [https://wilico.co.jp/ja/blog/what-is-web-scraping-mechanism-legality-use-cases](https://wilico.co.jp/ja/blog/what-is-web-scraping-mechanism-legality-use-cases)  
> 7. スクレイピングは違法？注目を集める便利なデータ収集方法の法的, [https://monolith.law/corporate/scraping-datacollection-law](https://monolith.law/corporate/scraping-datacollection-law)  
> 8. 日本でのウェブスクレイピングは合法？知っておくべきすべての法律, [https://thunderbit.com/ja/blog/web-scraping-legal-japan-guide](https://thunderbit.com/ja/blog/web-scraping-legal-japan-guide)  
> 9. SUUMOの賃貸データをスクレイピングで取得する｜実装ガイドと4, [https://wilico.co.jp/ja/blog/suumo-scraping](https://wilico.co.jp/ja/blog/suumo-scraping)  
> 10. スクレイピングはどこまで許されるのか 生成AIの学習データ収集も, [https://www.ys-law.jp/IT/column/column-11384/](https://www.ys-law.jp/IT/column/column-11384/)  
> 11. 利用規約 \- 空室情報.jp, [https://kuushitsu-info.jp/ur/terms](https://kuushitsu-info.jp/ur/terms)  
> 12. 生成AIによる著作権の侵害事例と最新の判例 \- 企業法務弁護士ナビ, [https://houmu-pro.com/property/297/](https://houmu-pro.com/property/297/)  
> 13. 高木浩光＠自宅の日記 \- 岡崎図書館事件から3年 〜 もう一つの誤認, [http://takagi-hiromitsu.jp/diary/20130316.html](http://takagi-hiromitsu.jp/diary/20130316.html)  
> 14. いま、社会教育施設に求められるウェブ活用 \- － Librahack 事件, [https://www.i-repository.net/contents/outemon/ir/504/504110308.pdf](https://www.i-repository.net/contents/outemon/ir/504/504110308.pdf)  
> 15. 負荷について考えたこと | Librahack ： 容疑者から見た岡崎図書館事件, [http://librahack.jp/okazaki-library-case/stress-test-thinking.html](http://librahack.jp/okazaki-library-case/stress-test-thinking.html)  
> 16. Web Scraping with AWS Lambda: 2026 Guide (Python and Java), [https://www.scrapingbee.com/blog/serverless-web-scraping-with-aws-lambda-and-java/](https://www.scrapingbee.com/blog/serverless-web-scraping-with-aws-lambda-and-java/)  
> 17. AWS LambdaとDockerで作るWebスクレイピング（Playwright）, [https://qiita.com/XXRyuzouXX/items/24bb97e03d19baf61ea5](https://qiita.com/XXRyuzouXX/items/24bb97e03d19baf61ea5)  
> 18. Web Scraping With Playwright: Python Guide for 2026 \- Scrapfly, [https://scrapfly.io/blog/posts/web-scraping-with-playwright-and-python](https://scrapfly.io/blog/posts/web-scraping-with-playwright-and-python)  
> 19. Scaling up a Serverless Web Crawler and Search Engine \- AWS, [https://aws.amazon.com/blogs/architecture/scaling-up-a-serverless-web-crawler-and-search-engine/](https://aws.amazon.com/blogs/architecture/scaling-up-a-serverless-web-crawler-and-search-engine/)  
> 20. AWS lambda : r/aws \- Reddit, [https://www.reddit.com/r/aws/comments/1mfaegy/aws\_lambda/](https://www.reddit.com/r/aws/comments/1mfaegy/aws_lambda/)  
> 21. Playwright on Cloud Run：低コストで実現するサーバーレス自動化, [https://zenn.dev/oharu121/articles/3a777b93f1d70d](https://zenn.dev/oharu121/articles/3a777b93f1d70d)  
> 22. Advanced Web Scraping with Cloud Browsers: A Technical Deep Dive, [https://scrape.do/blog/cloud-web-scraping/](https://scrape.do/blog/cloud-web-scraping/)  
> 23. Playwright Cloudflare Bypass: BQL Guide for Scraping \- Browserless, [https://www.browserless.io/blog/bypass-cloudflare-with-playwright](https://www.browserless.io/blog/bypass-cloudflare-with-playwright)  
> 24. Docker | Playwright Python \- cloudns.asia, [https://nightcat.cloudns.asia:9981/sitedoc/playwright/v1.54.1/python/docs/docker](https://nightcat.cloudns.asia:9981/sitedoc/playwright/v1.54.1/python/docs/docker)  
> 25. よくわかるGoogle Cloud\#4\_PlaywrightをCloud Runで動かす \- note, [https://note.com/kaho\_enterprise/n/n31d7dce8a961](https://note.com/kaho_enterprise/n/n31d7dce8a961)  
> 26. Playwright Bot Detection Bypass: What Works in 2026 \- AlterLab, [https://alterlab.io/blog/playwright-bot-detection-what-actually-works-in-2026](https://alterlab.io/blog/playwright-bot-detection-what-actually-works-in-2026)  
> 27. Playwright Stealth: Bypass Bot Detection in Python & Node.js, [https://scrapfly.io/blog/posts/playwright-stealth-bypass-bot-detection](https://scrapfly.io/blog/posts/playwright-stealth-bypass-bot-detection)  
> 28. How to Bypass Bot Detection Using Playwright Stealth in 2026, [https://www.scrapeless.com/en/blog/avoid-bot-detection-with-playwright-stealth](https://www.scrapeless.com/en/blog/avoid-bot-detection-with-playwright-stealth)  
> 29. Headless Browser Detection Bypass 2026 \- Sendwin, [https://blog.send.win/headless-browser-detection-bypass-2026/](https://blog.send.win/headless-browser-detection-bypass-2026/)  
> 30. Playwright Stealthによるボット検出の回避 \- Bright Data, [https://brightdata.jp/blog/%E5%90%84%E7%A8%AE%E3%81%94%E5%88%A9%E7%94%A8%E6%96%B9%E6%B3%95/avoid-bot-detection-with-playwright-stealth](https://brightdata.jp/blog/%E5%90%84%E7%A8%AE%E3%81%94%E5%88%A9%E7%94%A8%E6%96%B9%E6%B3%95/avoid-bot-detection-with-playwright-stealth)  
> 31. How to Bypass Cloudflare with Playwright in 2026 \- BrowserStack, [https://www.browserstack.com/guide/playwright-cloudflare](https://www.browserstack.com/guide/playwright-cloudflare)  
> 32. Understand star schema and the importance for Power BI, [https://learn.microsoft.com/en-us/power-bi/guidance/star-schema](https://learn.microsoft.com/en-us/power-bi/guidance/star-schema)  
> 33. Dimensional modeling \- Firma Joakim Dalby, [http://www.joakimdalby.dk/HTM/DimensionalModeling.htm](http://www.joakimdalby.dk/HTM/DimensionalModeling.htm)  
> 34. 107 Qlik View Interview Questions \- Adaface, [https://www.adaface.com/blog/qlik-view-interview-questions/](https://www.adaface.com/blog/qlik-view-interview-questions/)  
> 35. The Many-to-Many Revolution 2.0 \- SQLBI, [https://www.sqlbi.com/wp-content/uploads/The\_Many-to-Many\_Revolution\_2.0.pdf](https://www.sqlbi.com/wp-content/uploads/The_Many-to-Many_Revolution_2.0.pdf)  
> 36. Scalable SQL for Time-Series Data | PDF \- Scribd, [https://www.scribd.com/document/493677006/timescaledb](https://www.scribd.com/document/493677006/timescaledb)  
> 37. Postgres Extensions Guide: pgvector, TimescaleDB, PostGIS 2026, [https://kanopylabs.com/blog/postgres-extensions-guide-pgvector-timescaledb-postgis](https://kanopylabs.com/blog/postgres-extensions-guide-pgvector-timescaledb-postgis)  
> 38. Data Lakehouse vs Data Warehouse vs Data Lake \- Pipecode.ai, [https://pipecode.ai/blogs/data-lakehouse-vs-data-warehouse-vs-data-lake-which-architecture-wins](https://pipecode.ai/blogs/data-lakehouse-vs-data-warehouse-vs-data-lake-which-architecture-wins)  
> 39. LINE Messaging APIのAPI制限とリトライ処理の実装方法 \- ISSUE, [https://i-ssue.com/topics/48bfc3cc-0e5b-4b3f-9a50-f26b89b9e56e](https://i-ssue.com/topics/48bfc3cc-0e5b-4b3f-9a50-f26b89b9e56e)  
> 40. Error Handling and Retry Patterns for Playwright AI Agents, [https://callsphere.ai/blog/error-handling-retry-patterns-playwright-ai-agents](https://callsphere.ai/blog/error-handling-retry-patterns-playwright-ai-agents)  
> 41. Retry and Circuit Breaker in Spring Boot: When Resilience Backfires, [https://dev.to/akdevcraft/when-resilience-backfires-retry-and-circuit-breaker-in-spring-boot-10m](https://dev.to/akdevcraft/when-resilience-backfires-retry-and-circuit-breaker-in-spring-boot-10m)  
> 42. 空室率とは何か？ 賃貸経営・不動産投資の重要指標を事例で徹底解説, [https://relo-fudosan.jp/hack/knowledge/rental-management/rental-management\_vacancy/vacancy\_rate/](https://relo-fudosan.jp/hack/knowledge/rental-management/rental-management_vacancy/vacancy_rate/)  
> 43. 賃貸経営と投資判断に活かす「空室率」の考え方, [https://aoyama-chintaikanri.com/20251227/](https://aoyama-chintaikanri.com/20251227/)  
> 44. 3つの計算式を保有物件で検証・空室率の出し方と全国平均の目安, [https://go1101.com/blog-entry-169.html](https://go1101.com/blog-entry-169.html)  
> 45. 【東建】【土地活用】3つの空室率とは？ 計算方法や平均値を解説, [https://www.token.co.jp/estate/column/estate-library/h\_193/](https://www.token.co.jp/estate/column/estate-library/h_193/)  
> 46. 不動産投資の空室率とは？データ推移と空室対策アイデア例, [https://owners-cb.jp/realestate\_investment/6338](https://owners-cb.jp/realestate_investment/6338)  
> 47. 賃貸アパート３つの空室率とその計算方法, [https://numatashi-fudousan.com/fudousan/%E8%B3%83%E8%B2%B8%E3%82%A2%E3%83%91%E3%83%BC%E3%83%88%E7%A9%BA%E5%AE%A4%E5%AF%BE%E7%AD%96/43523/](https://numatashi-fudousan.com/fudousan/%E8%B3%83%E8%B2%B8%E3%82%A2%E3%83%91%E3%83%BC%E3%83%88%E7%A9%BA%E5%AE%A4%E5%AF%BE%E7%AD%96/43523/)