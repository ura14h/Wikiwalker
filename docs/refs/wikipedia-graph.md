# Wikipedia 関連記事の可視化手法

調査日：2026-09-15
対象：一つのWikipedia記事を開始点として関連記事をオンラインで逐次取得し、記事群をネットワークまたは関係付き点群に構造化して2D・3D表示するシステム

## 要旨と評価方法

本改訂版は分類を次の5カテゴリだけに統合し、**各40件、合計200件**のアルゴリズム・手法・実装を収録する。

1. 取得と探索（001–040）
2. 関係定義と順位づけ（041–080）
3. 点群化と粒度フィルタ（081–120）
4. 2次元レイアウト（121–160）
5. 3次元レイアウト（161–200）

各カテゴリ内は、本システムへの適合度を100点満点で評価した順に並べた。点数は学術論文の品質順位ではなく、Wikipedia適合度30%、目的への忠実度25%、オンライン／増分処理または計算規模20%、実装成熟度15%、説明可能性10%を基本にした**実装優先度**である。用途が変われば順位も変わるため、最終選定では実データによるnDCG、近傍保持率、stress、処理時間、視認性の比較が必要である。

基本データモデルは、記事を頂点、本文リンクを有向辺とする型付き多重グラフである。頂点には `wiki`、`pageid`、Wikidata QID、revision ID、取得日時を、辺には `type`、方向、重み、根拠、取得元revisionを保持する。リダイレクトは正規化するが元表記は別名として保存する。公開APIの一般仕様は既存の [Wikipedia Action API仕様書](./wikipedia-api.md) も参照する。

---

## 1. 取得と探索（40件、オンライン逐次取得を主眼）

### 001. Action API `prop=links` の逐次BFS — 98点

- **評価**：98/100（カテゴリ順位1/40）
- **典拠**：[MediaWiki API:Links](https://www.mediawiki.org/wiki/API:Links)
- **概要**：開始記事の内部リンクを取得し、未訪問記事をFIFOフロンティアへ追加して深さごとに展開する。`plnamespace=0`、`redirects=1`、継続トークンを基本とする。
- **メリット**：公式・最新・認証不要で、オンライン探索の基準実装にしやすい。深さと総頂点数を明確に制御できる。
- **デメリット**：高次数記事で候補が急増し、列挙順は関連度順ではない。
- **実装要点**：`visited pageid`、深度、親、取得revision、打切り理由を永続化し、1件ずつ受理判定してキューへ入れる。

### 002. `generator=links` と複数`prop`の同時取得 — 97点

- **評価**：97/100（カテゴリ順位2/40）
- **典拠**：[MediaWiki API:Query — Generators](https://www.mediawiki.org/wiki/API:Query)
- **概要**：リンク先をgeneratorとして流し、`info|pageprops|categories|extracts`などの候補評価用属性を同じ応答で取得する。
- **メリット**：候補一覧取得後のN+1リクエストを減らし、逐次探索の待ち時間とAPI負荷を小さくできる。
- **デメリット**：generatorと各propertyの継続状態が絡み、応答のマージ処理が複雑になる。
- **実装要点**：`batchcomplete`までは同一候補バッチとして統合し、generatorの次バッチへ進む前に属性継続を処理する。

### 003. 出リンク＋被リンクの交互展開 — 96点

- **評価**：96/100（カテゴリ順位3/40）
- **典拠**：[MediaWiki API:Backlinks](https://www.mediawiki.org/wiki/API:Backlinks)、[API:Links](https://www.mediawiki.org/wiki/API:Links)
- **概要**：各記事について出リンクと `list=backlinks` を交互に取得し、前方・後方近傍を別フロンティアで展開する。
- **メリット**：記事が参照する話題と、その記事を参照する話題の双方を拾い、片方向探索の偏りを減らせる。
- **デメリット**：著名記事の被リンクが巨大で、一覧・年・一般語記事が候補を占有しやすい。
- **実装要点**：探索方向と実際の辺方向を分離し、方向別上限と相互リンク優先枠を設ける。

### 004. 関連度優先ビーム探索 — 95点

- **評価**：95/100（カテゴリ順位4/40）
- **典拠**：[NetworkX `bfs_beam_edges`](https://networkx.org/documentation/stable/reference/algorithms/traversal.html)
- **概要**：各深度で候補をリンク位置、相互リンク、本文類似、PPRなどで採点し、上位 `beamWidth` 件だけ次層へ進める。
- **メリット**：幅優先の説明可能な深度を保ちながら、指数的な候補増加を強く抑える。
- **デメリット**：初期スコアが誤ると、後から有用になる経路を早期に捨てる。
- **実装要点**：全体上位だけでなくカテゴリ別quotaを併用し、棄却候補と理由も監査ログへ残す。

### 005. 最良優先探索（Best-first） — 94点

- **評価**：94/100（カテゴリ順位5/40）
- **典拠**：[Pearl, *Heuristics: Intelligent Search Strategies for Computer Problem Solving*](https://books.google.com/books/about/Heuristics.html?hl=en&id=1HpQAAAAMAAJ&output=html_text)
- **概要**：深さではなく開始記事への推定関連度が最大の候補からpriority queueで逐次展開する。
- **メリット**：限られたAPI予算で有望な記事を早く取得でき、対話UIへ途中結果を返しやすい。
- **デメリット**：人気度やテキスト類似だけを使うと同質な局所領域へ偏る。
- **実装要点**：`priority = relevance - depthPenalty + noveltyBonus` とし、未取得状態でも計算できる特徴と取得後の再採点を分ける。

### 006. `action=parse`による節・位置付きリンク抽出 — 93点

- **評価**：93/100（カテゴリ順位6/40）
- **典拠**：[MediaWiki API:Parsing wikitext](https://www.mediawiki.org/wiki/API:Parsing_wikitext)
- **概要**：parser出力のリンク、節、HTMLを取得し、導入部、本文、脚注、関連項目などの位置情報を辺属性にする。
- **メリット**：単なるリンク有無より豊かな逐次関連度を得られ、UIで根拠箇所を説明できる。
- **デメリット**：`prop=links`より応答が重く、HTML構造や記事テンプレート差への対応が必要。
- **実装要点**：全候補へ使わず、一次選抜された記事だけを `oldid` 固定でparseする二段取得にする。

### 007. MediaWiki REST APIのHTML取得とDOM走査 — 92点

- **評価**：92/100（カテゴリ順位7/40）
- **典拠**：[MediaWiki REST API](https://www.mediawiki.org/wiki/API:REST_API)、[REST API Reference](https://www.mediawiki.org/wiki/API:REST_API/Reference)
- **概要**：RESTのページHTMLを取得し、DOM上のアンカー、節、注釈、データ属性を順に走査してリンク候補を作る。
- **メリット**：URLが単純でキャッシュされやすく、表示構造に近い文脈を利用できる。
- **デメリット**：Action APIより機能範囲が狭く、DOM解析の実装負担がある。
- **実装要点**：HTMLとrevision IDを組にして保存し、本文領域外リンク、赤リンク、非記事名前空間を明示的に除く。

### 008. カテゴリ二部グラフの逐次展開 — 91点

- **評価**：91/100（カテゴリ順位8/40）
- **典拠**：[API:Categories](https://www.mediawiki.org/wiki/API:Categories)、[API:Categorymembers](https://www.mediawiki.org/wiki/API:Categorymembers)
- **概要**：記事からカテゴリを取得し、選ばれたカテゴリのメンバーを次候補にする。記事–カテゴリを異種辺のまま保持する。
- **メリット**：本文リンクがなくても同一主題の記事を発見でき、コミュニティの説明ラベルにもなる。
- **デメリット**：巨大・保守・循環カテゴリが多く、単純射影はクリークを作る。
- **実装要点**：カテゴリ次数のIDF減衰、隠しカテゴリ除外、カテゴリ側の深度・件数上限を独立に設定する。

### 009. Wikidata `wbgetentities` の型付き近傍展開 — 90点

- **評価**：90/100（カテゴリ順位9/40）
- **典拠**：[Wikidata API: wbgetentities](https://www.wikidata.org/w/api.php?action=help&modules=wbgetentities)、[Wikidata:Data access](https://www.wikidata.org/wiki/Wikidata:Data_access)
- **概要**：記事のQIDからclaimとsitelinkをオンライン取得し、`instance of`、`part of`、`subclass of`など許可したプロパティをたどる。
- **メリット**：関係の意味と方向が明示され、言語版をまたぐ概念グラフを構成できる。
- **デメリット**：プロパティごとの意味差、欠損、statement rank、qualifierの処理が必要。
- **実装要点**：プロパティallowlistと型別予算を持ち、記事ノードとWikidata概念ノードを混同しない。

### 010. 継続トークンのチェックポイント走査 — 89点

- **評価**：89/100（カテゴリ順位10/40）
- **典拠**：[MediaWiki API:Continue](https://www.mediawiki.org/wiki/API:Continue/en)
- **概要**：`plcontinue`、`blcontinue`等を応答ごとに保存し、候補列挙を中断・再開可能な逐次ストリームとして扱う。
- **メリット**：巨大な隣接リストでも欠落と重複を抑え、プロセス停止後に再取得せず続行できる。
- **デメリット**：同時編集でページ内容が変化すると、異なる時点の候補が混在し得る。
- **実装要点**：query fingerprint、revision ID、continue値、受領済みID集合を一つのcheckpointとして原子的に保存する。

### 011. リダイレクト正規化付き探索 — 88点

- **評価**：88/100（カテゴリ順位11/40）
- **典拠**：[MediaWiki API:Query — Resolving redirects](https://www.mediawiki.org/wiki/API:Query)
- **概要**：各バッチに `redirects=1` を指定し、normalized・redirects対応表で別名を最終pageidへ統合してから訪問判定する。
- **メリット**：同一概念の重複頂点、別名を経由する循環、分散した次数を防げる。
- **デメリット**：節リダイレクトや履歴上の表記差を完全に捨てると説明性が落ちる。
- **実装要点**：正規IDを主キーにしつつ、入力タイトル、正規化タイトル、転送元、fragmentを別属性で保持する。

### 012. `pageprops`による曖昧さ・QID・特殊ページ判定 — 87点

- **評価**：87/100（カテゴリ順位12/40）
- **典拠**：[MediaWiki API:Pageprops](https://www.mediawiki.org/wiki/API:Pageprops)
- **概要**：リンク候補と同時にpage propertiesを取得し、曖昧さ回避、Wikibase item、既定ソート等を候補選択に使う。
- **メリット**：余分な本文取得前に曖昧さ回避や概念IDを判定でき、逐次フィルタが安い。
- **デメリット**：プロパティはwikiや拡張機能に依存し、空値を「通常記事」と断定できない。
- **実装要点**：曖昧さ回避は除外固定にせず、開始記事と同名候補の分岐ノードとして別扱いできる設定にする。

### 013. Pageviews APIによる人気度優先探索 — 86点

- **評価**：86/100（カテゴリ順位13/40）
- **典拠**：[Wikimedia Analytics API: Page views](https://doc.wikimedia.org/generated-data-platform/aqs/analytics-api/reference/page-views.html)
- **概要**：候補記事の直近閲覧数をオンライン取得し、関連度と組み合わせて取得優先度または表示優先度を調整する。
- **メリット**：閲覧需要の高い説明記事を先に返せ、対話体験を改善しやすい。
- **デメリット**：ニュース・季節・言語規模に強く左右され、意味的関連性とは異なる。
- **実装要点**：`log1p(pageviews)`と期間中央値を使い、関連度を上書きせず弱い補助特徴にする。

### 014. CirrusSearchの全文検索による候補補完 — 85点

- **評価**：85/100（カテゴリ順位14/40）
- **典拠**：[MediaWiki API:Search](https://www.mediawiki.org/wiki/API:Search/en)
- **概要**：開始記事のタイトル、要約、重要語を `list=search` またはgeneratorへ渡し、リンクで直接接続されない記事を補完する。
- **メリット**：新規記事やリンク編集が薄い分野でも語彙的に近い候補を発見できる。
- **デメリット**：検索バックエンドのスコアは完全には説明できず、キーワード一致へ偏る。
- **実装要点**：検索由来の辺を本文リンクと分け、snippet、検索順位、query、取得時刻を根拠として保存する。

### 015. EventStreamsによる探索済み記事の増分更新 — 84点

- **評価**：84/100（カテゴリ順位15/40）
- **典拠**：[Wikimedia EventStreams](https://wikitech.wikimedia.org/wiki/EventStreams)
- **概要**：`recentchange`または`revision-create`のSSEを購読し、探索済みpageidに関係する更新だけ再取得キューへ送る。
- **メリット**：全グラフを再クロールせず、新規・変更リンクをほぼリアルタイムに反映できる。
- **デメリット**：切断、再送、順序、保持期間に対応する状態管理が必要で、初期構築には使えない。
- **実装要点**：Last-Event-ID、冪等upsert、domain filter、デバウンスを実装し、イベント本文をリンク差分そのものとはみなさない。

### 016. revision ID固定の本文再取得 — 83点

- **評価**：83/100（カテゴリ順位16/40）
- **典拠**：[MediaWiki API:Revisions](https://www.mediawiki.org/wiki/API:Revisions)、[REST API page history](https://www.mediawiki.org/wiki/API:REST_API/Reference)
- **概要**：候補採用時の最新revision IDを取得し、その版のwikitextまたはHTMLからリンクを抽出する。
- **メリット**：オンライン取得でも同一入力を再現でき、更新判定はrevision ID比較だけで済む。
- **デメリット**：特定版のparseは高コストになり得て、古い版を永続保持すると容量が増える。
- **実装要点**：頂点の `sourceRevision` を必須にし、最新版確認と本文取得を分けて変更記事だけ再解析する。

### 017. Wikidata SPARQLの制約付き展開 — 82点

- **評価**：82/100（カテゴリ順位17/40）
- **典拠**：[Wikidata Query Service](https://www.wikidata.org/wiki/Wikidata:SPARQL_query_service)
- **概要**：開始QIDをVALUESで限定し、特定プロパティ、型、言語sitelinkをSPARQLで小さく取得する。
- **メリット**：複数hopの型付きパターンを一問合せで表現でき、概念条件による候補絞込みが強い。
- **デメリット**：公開endpointはtimeoutと利用制約があり、無制約property pathは不安定。
- **実装要点**：ページング、短いtimeout、結果cache、property allowlistを使い、失敗時は `wbgetentities` へフォールバックする。

### 018. 言語間リンク／sitelinkの逐次統合 — 81点

- **評価**：81/100（カテゴリ順位18/40）
- **典拠**：[MediaWiki API:Langlinks](https://www.mediawiki.org/wiki/API:Langlinks)、[Wikidata data access](https://www.wikidata.org/wiki/Wikidata:Data_access)
- **概要**：記事のlanglinkまたはQIDのsitelinkを取得し、同一概念の各言語版を束ねるか別頂点として接続する。
- **メリット**：言語版固有のリンクと記事密度を比較でき、欠けた関係を他言語版から発見できる。
- **デメリット**：sitelinkは意味完全同一とは限らず、全言語を展開すると急増する。
- **実装要点**：QIDを概念層、`wiki+pageid`を記事層にして分離し、対象言語allowlistを設ける。

### 019. 「関連項目」節限定の逐次取得 — 80点

- **評価**：80/100（カテゴリ順位19/40）
- **典拠**：[WikiLink semantic network研究](https://pmc.ncbi.nlm.nih.gov/articles/PMC9680257/)、[API:Parsing wikitext](https://www.mediawiki.org/wiki/API:Parsing_wikitext)
- **概要**：記事の節一覧から「関連項目／See also」に相当する節だけをparseし、編集者が明示した関係候補を優先する。
- **メリット**：本文の偶発的リンクより意図的な関連リストを得やすく、候補数が小さい。
- **デメリット**：節名・慣習が言語で異なり、節がない記事も多い。
- **実装要点**：言語別の節名辞書とsection indexを使い、通常本文リンクとは異なるedge typeにする。

### 020. First-Link Network — 79点

- **評価**：79/100（カテゴリ順位20/40）
- **典拠**：[Lamprecht et al., “Connecting every bit of knowledge”](https://arxiv.org/abs/1605.00309)
- **概要**：本文を順に読み、括弧・注釈等を除いた最初の有効な記事リンクだけを次の候補としてたどる。
- **メリット**：分岐をほぼ1に抑え、定義的・上位概念的な連鎖を非常に安く抽出できる。
- **デメリット**：記事文体に依存し、主題の多様な関連群をほとんど拾えない。
- **実装要点**：ナビゲーション規則を版とともに固定し、主探索の代表骨格または初期経路として使う。

### 021. Wikimedia Clickstreamによる遷移辺の補完 — 78点

- **評価**：78/100（カテゴリ順位21/40）
- **典拠**：[Wikimedia Clickstream](https://meta.wikimedia.org/wiki/Research:Wikipedia_clickstream)
- **概要**：月次公開データから探索済み記事間の実クリックを照合し、読者遷移を重み付き辺として後付けする。
- **メリット**：編集リンクの存在だけでなく、利用者が実際に選ぶ経路を反映できる。
- **デメリット**：オンライン即時値ではなく月次で、低頻度遷移は公開されず、意味関連とは別物である。
- **実装要点**：初期探索はAPIで逐次実行し、clickstreamは非同期補助レイヤーとして差分更新する。

### 022. テンプレート埋込み関係の逐次展開 — 77点

- **評価**：77/100（カテゴリ順位22/40）
- **典拠**：[MediaWiki API:Embeddedin](https://www.mediawiki.org/wiki/API:Embeddedin)、[API:Templates](https://www.mediawiki.org/wiki/API:Templates)
- **概要**：記事が使うnavbox等のテンプレートと、そのテンプレートを埋め込む記事を異種関係としてたどる。
- **メリット**：同一シリーズ、地域、人物群など編集上まとまった記事集合を発見しやすい。
- **デメリット**：汎用・保守テンプレートは巨大で、強い人工的クリークを作る。
- **実装要点**：テンプレート次数と名前空間で抑制し、本文リンクへ潰さず `transclusion` として保持する。

### 023. 外部ドメイン共有による候補補完 — 76点

- **評価**：76/100（カテゴリ順位23/40）
- **典拠**：[MediaWiki API:Extlinks](https://www.mediawiki.org/wiki/API:Extlinks)、[API:Exturlusage](https://www.mediawiki.org/wiki/API:Exturlusage)
- **概要**：記事の外部リンクをドメインまたは正規化URLへまとめ、同じ公式サイト・資料を参照する記事を候補化する。
- **メリット**：本文内部リンクでは結ばれない同一組織・データ源の関係を見つけられる。
- **デメリット**：大手サイトやアーカイブは識別力が低く、URL正規化と外部取得の安全性が課題。
- **実装要点**：ドメインIDF、tracking query除去、HTTPS正規化を行い、外部URL自体は自動巡回しない。

### 024. GeoSearchによる地理近傍展開 — 75点

- **評価**：75/100（カテゴリ順位24/40）
- **典拠**：[MediaWiki API:Search and discovery](https://www.mediawiki.org/wiki/API:Search_and_discovery/en)
- **概要**：座標を持つ記事から `list=geosearch` で半径内記事を逐次候補化し、距離を関係属性にする。
- **メリット**：場所、建造物、出来事ではリンク有無と独立した明確な近傍を得られる。
- **デメリット**：非地理記事には使えず、同一地点の大量記事や縮尺差を調整する必要がある。
- **実装要点**：半径を記事型と密度に応じて変え、地理辺を意味辺とは別レイヤーにする。

### 025. 双方向探索（開始点と目標集合） — 74点

- **評価**：74/100（カテゴリ順位25/40）
- **典拠**：[NetworkX shortest paths](https://networkx.org/documentation/stable/reference/algorithms/shortest_paths.html)
- **概要**：開始記事から出リンク、目標記事またはテーマ集合から被リンクを展開し、フロンティアの交差で接続経路を得る。
- **メリット**：特定の二概念を結ぶ説明経路を、片側探索より少ない取得で発見しやすい。
- **デメリット**：目標が未指定の一般的な関連記事収集には直接適用できない。
- **実装要点**：両側で同じ正規pageidを使い、交差後も代替経路を一定数探索して一経路への偶然依存を避ける。

### 026. 反復深化DFS（IDDFS） — 73点

- **評価**：73/100（カテゴリ順位26/40）
- **典拠**：[NetworkX depth-limited DFS](https://networkx.org/documentation/stable/reference/algorithms/traversal.html)
- **概要**：深さ上限を0、1、2…と増やし、メモリを抑えながら浅い関連経路を優先して再探索する。
- **メリット**：DFS程度のメモリでBFSに近い浅い解の発見順を得られる。
- **デメリット**：同じ記事の隣接リストを繰り返し扱うため、キャッシュなしではAPI浪費が大きい。
- **実装要点**：取得応答は永続cacheし、反復するのはローカルな受理・経路判定だけにする。

### 027. 重み付き一様費用探索 — 72点

- **評価**：72/100（カテゴリ順位27/40）
- **典拠**：[NetworkX Dijkstra algorithms](https://networkx.org/documentation/stable/reference/algorithms/shortest_paths.html)
- **概要**：辺コストを `-log(relevance)` 等で定義し、開始記事から累積コストが小さい候補を順に展開する。
- **メリット**：強い関係が連続する少し深い記事を、弱い1-hop記事より先に取得できる。
- **デメリット**：負コストを使えず、辺重みが取得後にしか分からない場合は再評価が必要。
- **実装要点**：未取得辺には下界コストを与え、確定後のdecrease-keyと最大累積コストを実装する。

### 028. A*型の目標指向探索 — 71点

- **評価**：71/100（カテゴリ順位28/40）
- **典拠**：[Hart, Nilsson & Raphael, A*](https://doi.org/10.1109/TSSC.1968.300136)
- **概要**：特定テーマまたは目標記事までの既知コスト `g` と、埋め込み距離等の推定残余 `h` を加えて展開順を決める。
- **メリット**：二概念間の関連経路やテーマ到達を少ないAPI呼出で狙える。
- **デメリット**：意味距離は最短路に対するadmissible heuristicになりにくく、最適性保証が崩れる。
- **実装要点**：説明経路探索では `h=0` のDijkstra結果も比較し、近似利用であることを明示する。

### 029. Random Walk with Restart探索 — 70点

- **評価**：70/100（カテゴリ順位29/40）
- **典拠**：[Random Walk with Restart](https://pmc.ncbi.nlm.nih.gov/articles/PMC6426185/)
- **概要**：現在記事のリンクへ確率的に移動し、一定確率で開始記事へ戻る操作を繰り返して訪問候補を集める。
- **メリット**：開始点の局所性を保ちつつ深さ固定でない探索ができ、PPR近似と整合する。
- **デメリット**：高次数・強連結領域に偏り、低頻度候補の再現性が低い。
- **実装要点**：乱数seed、再始動率、最大step、重複訪問回数を記録し、未取得隣接だけAPIから補う。

### 030. 単純ランダムウォーク — 69点

- **評価**：69/100（カテゴリ順位30/40）
- **典拠**：[Yeh et al., “WikiWalk”](https://nlp.stanford.edu/pubs/wikiwalk-textgraphs09.pdf)
- **概要**：現在記事の出リンクから一様または重み付きで次記事を選び、複数walkerでオンライン巡回する。
- **メリット**：実装が簡単で、固定幅探索と異なる長い経路を低メモリで発見できる。
- **デメリット**：ハブとsinkに影響され、関連記事集合の網羅性を保証できない。
- **実装要点**：teleport、dead-end復帰、複数chain、訪問頻度のburn-in除外を備える。

### 031. Metropolis–Hastings Random Walk — 68点

- **評価**：68/100（カテゴリ順位31/40）
- **典拠**：[Leskovec & Faloutsos, “Sampling from Large Graphs”](https://www.cs.cmu.edu/~jure/pubs/sampling-kdd06.pdf)
- **概要**：次数による訪問偏りを受理確率で補正し、所望の定常分布へ近づけながら記事を取得する。
- **メリット**：単純walkよりハブ過大評価を抑え、構造統計用サンプルを作りやすい。
- **デメリット**：有向グラフでの設計が難しく、自己ループ的な棄却で探索が遅い。
- **実装要点**：相互リンクで作る無向提案グラフなど、遷移核と目標分布を明記して使う。

### 032. Forest Fire Sampling — 67点

- **評価**：67/100（カテゴリ順位32/40）
- **典拠**：[Leskovec & Faloutsos, “Sampling from Large Graphs”](https://www.cs.cmu.edu/~jure/pubs/sampling-kdd06.pdf)
- **概要**：訪問記事の隣接先から確率的な個数を選んで「燃え広がる」ように再帰展開する。
- **メリット**：コミュニティ内部と橋を混ぜた現実的な部分グラフを少ない予算で得やすい。
- **デメリット**：燃焼確率に規模が敏感で、一度の実行結果が大きく揺れる。
- **実装要点**：前方・後方燃焼率、seed、総API予算を固定し、複数サンプルの安定辺を採用する。

### 033. Snowball／多段ego sampling — 66点

- **評価**：66/100（カテゴリ順位33/40）
- **典拠**：[Hu & Lau, A Survey and Taxonomy of Graph Sampling](https://arxiv.org/abs/1308.5865)
- **概要**：seed近傍を層ごとに一定割合または一定件数ずつ採用し、その隣接へ段階的に拡張する。
- **メリット**：実装が容易で、開始記事周辺の局所構造を確実に含められる。
- **デメリット**：高次数・高密度コミュニティへの偏りが強く、全体統計には不向き。
- **実装要点**：深度別採用率、親ごとのcap、複数seedを使い、サンプルであることを属性に残す。

### 034. Reservoir sampling付きフロンティア — 65点

- **評価**：65/100（カテゴリ順位34/40）
- **典拠**：[Vitter, “Random Sampling with a Reservoir”](https://doi.org/10.1145/3147.3165)
- **概要**：継続取得される未知長のリンク列から、一定メモリで一様な `k` 候補を保持する。
- **メリット**：巨大な被リンクやカテゴリメンバーでも全件保持せず、公平な候補枠を作れる。
- **デメリット**：関連度を使わない単純reservoirは有用な候補を落とし、全列の走査通信は残る。
- **実装要点**：重み付きreservoirと早期打切りを分け、途中打切り時は一様標本でないことを記録する。

### 035. 親記事ごとの次数cap探索 — 64点

- **評価**：64/100（カテゴリ順位35/40）
- **典拠**：[MediaWiki API limits](https://www.mediawiki.org/wiki/API:Lists/en)
- **概要**：一つの親から採用する出リンク・被リンク数を `p` 件に制限し、広い記事が全予算を消費するのを防ぐ。
- **メリット**：単純で予測可能な上限を与え、開始点ごとの応答時間を安定化できる。
- **デメリット**：API列挙順の先頭をそのまま採ると恣意的で、重要候補を落とす。
- **実装要点**：先に軽量特徴を取得してtop-p化するか、reservoirと組み合わせ、打切り状態を辺集合に付す。

### 036. 多様性quota付き層別探索 — 63点

- **評価**：63/100（カテゴリ順位36/40）
- **典拠**：[xQuADによる検索結果多様化](https://theses.gla.ac.uk/4106/)
- **概要**：カテゴリ、Wikidata型、言語、関係種ごとに最低・最大枠を設け、同一話題だけでフロンティアが埋まらないよう採用する。
- **メリット**：関連記事クラウドの主題幅を保ち、人気カテゴリの独占を抑えられる。
- **デメリット**：quota設計が結果を作り込み、分類誤りや少数群の水増しを招く。
- **実装要点**：関連度下限を先に適用し、その上で不足aspectにnovelty bonusを加える。

### 037. Monte Carlo Tree Search／UCT探索 — 62点

- **評価**：62/100（カテゴリ順位37/40）
- **典拠**：[Kocsis & Szepesvári, UCT](https://doi.org/10.1007/11871842_29)
- **概要**：記事を状態、リンク選択を行動とみなし、関連候補の発見を報酬として探索と活用をUCTで調整する。
- **メリット**：深い経路の潜在価値をrolloutで評価でき、固定深度やgreedyの盲点を補える。
- **デメリット**：報酬設計と大量rolloutが必要で、API呼出を直接rolloutに使うと高価。
- **実装要点**：取得済み部分グラフ内でrolloutし、未取得ノードの展開だけ予算管理されたAPI操作にする。

### 038. HTTP条件付き取得と応答cache — 61点

- **評価**：61/100（カテゴリ順位38/40）
- **典拠**：[MediaWiki REST API](https://www.mediawiki.org/wiki/API:REST_API)、[API Etiquette — Caching](https://www.mediawiki.org/wiki/API:Etiquette)
- **概要**：ETag、Last-Modified、revision ID、query keyを使い、同じ記事・継続ページの再取得を避ける。
- **メリット**：反復探索、UI再表示、失敗再開時の待ち時間とWikimedia負荷を大幅に減らす。
- **デメリット**：cache invalidation、容量、異なるquery parameterの同一視ミスが起き得る。
- **実装要点**：raw応答cacheと正規化結果cacheを分け、TTLよりrevision IDを優先して整合性を判断する。

### 039. `maxlag`＋指数backoff＋jitter — 60点

- **評価**：60/100（カテゴリ順位39/40）
- **典拠**：[MediaWiki API:Etiquette](https://www.mediawiki.org/wiki/API:Etiquette)、[Manual:Maxlag](https://www.mediawiki.org/wiki/Manual:Maxlag_parameter)
- **概要**：非対話処理に `maxlag` を付け、429・lag・一時障害ではRetry-Afterまたは指数backoffで逐次再試行する。
- **メリット**：APIへの負荷を抑え、長時間クロールの失敗率とブロック危険を減らす。
- **デメリット**：高負荷時は進行が遅く、無制限再試行はジョブを停止不能にする。
- **実装要点**：識別可能なUser-Agent、jitter、再試行上限、dead-letter queueを備え、原則直列要求にする。

### 040. 冪等checkpoint／再開可能ワークキュー — 59点

- **評価**：59/100（カテゴリ順位40/40）
- **典拠**：[EventStreams resume semantics](https://wikitech.wikimedia.org/wiki/EventStreams)、[API continuation](https://www.mediawiki.org/wiki/API:Continue/en)
- **概要**：`pending/fetching/done/failed`状態、query hash、continue値、結果versionを永続化し、各取得単位を冪等に処理する。
- **メリット**：数時間規模のオンライン探索でもクラッシュ、通信断、再実行から正確に復帰できる。
- **デメリット**：アルゴリズムというより運用手法で、DBと状態遷移の実装量が増える。
- **実装要点**：lease timeoutと一意制約で二重処理を抑え、同じpageidへの複数親辺はupsertで失わない。

---

## 2. 関係定義と順位づけ（40件）

### 041. 型付き有向多重グラフ — 98点

- **評価**：98/100（カテゴリ順位1/40）
- **典拠**：[Wikimedia link tables](https://meta.wikimedia.org/wiki/Data_dumps/What%27s_available_for_download)、[Wikidata data model](https://www.wikidata.org/wiki/Wikidata:Data_model)
- **概要**：本文リンク、被リンク、カテゴリ、Wikidata、クリック、語彙類似を別typeの有向・無向辺として並存させる。
- **メリット**：「なぜ関連するか」を失わず、目的ごとに辺型を選択・集約できる。
- **デメリット**：単純グラフ用算法へ渡す前の射影規則が増え、UIも複雑になる。
- **実装要点**：正本は `MultiDiGraph` 相当とし、分析ごとに版管理された派生グラフを生成する。

### 042. 相互リンク重み — 97点

- **評価**：97/100（カテゴリ順位2/40）
- **典拠**：[Agirre et al., Wikipedia graph variants](https://arxiv.org/abs/1503.01655)
- **概要**：`A→B`と`B→A`が共存する辺を強い関係として加点するか、相互リンクだけの無向グラフを作る。
- **メリット**：一方向の説明用リンクより、双方が編集上重要とした関係を優先できる。
- **デメリット**：新規・専門記事や自然な上下関係に多い有用な片方向辺を落とす。
- **実装要点**：除外条件にせず `reciprocal` 特徴として保持し、他の根拠と合成する。

### 043. 本文位置・節・画面位置による重み — 96点

- **評価**：96/100（カテゴリ順位3/40）
- **典拠**：[Dimitrov et al., “What Makes a Link Successful on Wikipedia?”](https://arxiv.org/abs/1611.02508)
- **概要**：導入部、最初の有効リンク、関連項目、本文後半、注釈など、記事内位置を辺特徴へ変換する。
- **メリット**：編集上の強調と読者が実際に見つけやすい関係を二値リンクより細かく表せる。
- **デメリット**：記事形式、言語、画面幅に依存し、固定係数は普遍的でない。
- **実装要点**：位置を生の正規化値と節typeで保存し、係数は検証データから調整する。

### 044. Milne–Witten Link-based Measure（WLM） — 95点

- **評価**：95/100（カテゴリ順位4/40）
- **典拠**：[Milne & Witten, 2008](https://www.coli.uni-saarland.de/courses/WebAsCorpus-12/papers/Milne-Witten-08.pdf)
- **概要**：二記事の共通被リンク数を全体記事数と各被リンク数で対数正規化し、Wikipedia固有の意味関連度を求める。
- **メリット**：本文全文を使わず、一般的ハブの影響を補正した説明可能な関連度を得られる。
- **デメリット**：被リンク集合の取得が重く、新規・低被リンク記事では不安定になる。
- **実装要点**：ゼロ共通集合の扱い、全体記事数、スナップショット時点を固定し、被リンク集合をcacheする。

### 045. Personalized PageRank（PPR） — 94点

- **評価**：94/100（カテゴリ順位5/40）
- **典拠**：[WikiWalk](https://aclanthology.org/W09-3206/)、[Personalized PageRank to a target](https://arxiv.org/abs/1304.4658)
- **概要**：開始記事へteleportするランダムウォークの定常確率で、取得済みグラフ内の記事を順位づけする。
- **メリット**：直接リンクだけでなく複数hopの支持を統合し、開始点への局所性を明示的に保てる。
- **デメリット**：取得部分グラフの境界と高次数記事に影響され、全Wikipediaでの値とは一致しない。
- **実装要点**：辺型別重み、dangling処理、damping、収束許容差、探索版を保存する。

### 046. クリック遷移確率 — 93点

- **評価**：93/100（カテゴリ順位6/40）
- **典拠**：[Wikimedia Clickstream](https://meta.wikimedia.org/wiki/Research:Wikipedia_clickstream)、[Wikipedia navigation study](https://arxiv.org/abs/2201.00812)
- **概要**：`P(B|A)=count(A→B)/Σx count(A→x)` を実利用に基づく有向辺重みとする。
- **メリット**：その記事を読んだ人が次に選ぶ関連先を反映し、ナビゲーション用途に直結する。
- **デメリット**：低頻度欠落、季節性、UI配置、外部流入の影響を受け、意味関連ではない。
- **実装要点**：月、language、流入typeを保持し、平滑化と `log1p` を使い、編集リンク重みと別表示する。

### 047. リンク＋本文類似のハイブリッドRandom Walk — 92点

- **評価**：92/100（カテゴリ順位7/40）
- **典拠**：[Yazdani & Popescu-Belis, 2013](https://doi.org/10.1016/j.artint.2012.06.004)
- **概要**：ハイパーリンク遷移と語彙・意味ベクトル類似を一つの遷移行列または複数層walkへ統合する。
- **メリット**：リンク不足と同音語的な本文類似の双方を補い、単一情報源より頑健になりやすい。
- **デメリット**：尺度と係数の選択が結果を大きく左右し、説明が一段難しくなる。
- **実装要点**：各成分スコアを個別保存し、ablationで寄与を確認してから校正する。

### 048. LambdaMARTによる学習順位 — 91点

- **評価**：91/100（カテゴリ順位8/40）
- **典拠**：[LambdaMART / learning-to-rank実証整理](https://arxiv.org/abs/2204.01500)
- **概要**：相互リンク、位置、WLM、本文類似、深度、人気度などを特徴にし、人手関連判定またはクリックからGBDT順位器を学習する。
- **メリット**：非線形な特徴相互作用を扱い、目的指標のnDCGを直接改善しやすい。
- **デメリット**：十分で偏りの少ない教師データが必要で、版・言語・開始記事型のdomain shiftを受ける。
- **実装要点**：query groupを開始記事単位にし、記事や時間を跨いだdata leakageを防ぐ。

### 049. Sentence-BERT cosine関連度 — 90点

- **評価**：90/100（カテゴリ順位9/40）
- **典拠**：[Reimers & Gurevych, Sentence-BERT](https://aclanthology.org/D19-1410/)
- **概要**：記事の導入要約を文埋め込みへ変換し、開始記事または親記事とのcosine類似で順位づけする。
- **メリット**：語彙が異なる意味的近さを捉え、ベクトル索引で候補再順位づけを高速化できる。
- **デメリット**：モデル言語・長文切詰めに依存し、関係方向や「対立」も近くなり得る。
- **実装要点**：多言語モデル、入力revision、pooling方法、embedding versionを固定する。

### 050. BM25関連度 — 89点

- **評価**：89/100（カテゴリ順位10/40）
- **典拠**：[Robertson & Zaragoza, “BM25 and Beyond”](https://www.staff.city.ac.uk/~sbrp622/papers/foundations_bm25_review.pdf)
- **概要**：開始記事の重要語をquery、候補記事本文をdocumentとして、語頻度飽和と文書長補正を含むBM25で採点する。
- **メリット**：高速で説明しやすく、専門語・固有名詞が一致する関係に強い。
- **デメリット**：同義語や翻訳に弱く、開始記事全文をqueryにすると焦点がぼける。
- **実装要点**：導入部または上位語だけをqueryにし、言語別tokenizerとIDF corpus時点を保存する。

### 051. TF–IDF cosine関連度 — 88点

- **評価**：88/100（カテゴリ順位11/40）
- **典拠**：[scikit-learn `TfidfVectorizer`](https://scikit-learn.org/stable/modules/generated/sklearn.feature_extraction.text.TfidfVectorizer.html)
- **概要**：記事本文のTF–IDFベクトル間cosineを、語彙的な対称関連度として利用する。
- **メリット**：軽量・再現可能で、重みの根拠語を表示できる強いbaselineになる。
- **デメリット**：表記揺れ、同義語、短文、言語横断に弱く、語彙行列が大きい。
- **実装要点**：本文と参照・テンプレートを分離し、sublinear TF、min_df、ngramを検証する。

### 052. Wikidataプロパティ型別重み — 87点

- **評価**：87/100（カテゴリ順位12/40）
- **典拠**：[Wikidata data model](https://www.wikidata.org/wiki/Wikidata:Data_model)
- **概要**：`subclass of`、`part of`、`author`、`location`等を別関係として、目的に応じた型別係数を与える。
- **メリット**：方向と意味を説明でき、人物・場所・作品など記事型に合う関係を優先できる。
- **デメリット**：プロパティ数が多く、同じpropertyでも領域により情報量が違う。
- **実装要点**：記事型ごとのallowlistと係数表を版管理し、qualifierとstatement rankも特徴に含める。

### 053. カテゴリIDF関連度 — 86点

- **評価**：86/100（カテゴリ順位13/40）
- **典拠**：[MediaWiki API:Categories](https://www.mediawiki.org/wiki/API:Categories)、[TF–IDF原理](https://scikit-learn.org/stable/modules/feature_extraction.html#tfidf-term-weighting)
- **概要**：共有カテゴリごとに `log(N/|category|)` を与え、希少で識別的なカテゴリの一致を高く評価する。
- **メリット**：「存命人物」等の巨大カテゴリを自動減衰し、説明可能な類似度を作れる。
- **デメリット**：カテゴリの保守用途・階層距離・言語差を十分には扱わない。
- **実装要点**：hidden category除外とカテゴリサイズcacheを使い、親カテゴリ伝播には深度減衰を入れる。

### 054. PageRank — 85点

- **評価**：85/100（カテゴリ順位14/40）
- **典拠**：[Brin & Page, Web search engine](https://research.google/pubs/the-anatomy-of-a-large-scale-hypertextual-web-search-engine/)、[NetworkX PageRank](https://networkx.org/documentation/stable/reference/algorithms/generated/networkx.algorithms.link_analysis.pagerank_alg.pagerank.html)
- **概要**：リンクされる記事ほど、かつ重要記事からリンクされるほど高くなる全体的権威度を計算する。
- **メリット**：有向リンク構造を自然に使い、主要記事の表示サイズや補助順位に適する。
- **デメリット**：開始記事への関連性を直接測らず、部分グラフでは境界効果が強い。
- **実装要点**：主順位はPPRにし、PageRankはglobal importance特徴として弱く合成する。

### 055. HITS（hub／authority） — 84点

- **評価**：84/100（カテゴリ順位15/40）
- **典拠**：[Kleinberg, HITS](https://doi.org/10.1145/324133.324140)
- **概要**：良い一覧・概説へ相当するhubと、良いhubから参照されるauthorityを相互強化で別々に算出する。
- **メリット**：Wikipediaの索引的記事と中心概念を異なる役割として表示できる。
- **デメリット**：局所サブグラフ選択に敏感で、密な主題へtopic driftしやすい。
- **実装要点**：開始点周辺のbase setを固定し、hubとauthorityを一つの値へ潰さない。

### 056. SALSA — 83点

- **評価**：83/100（カテゴリ順位16/40）
- **典拠**：[Lempel & Moran, SALSA](https://doi.org/10.1016/S1389-1286%2800%2900034-7)
- **概要**：hub側とauthority側の二部表現上のランダムウォークで、HITSと確率的順位づけを結ぶ。
- **メリット**：HITSよりdegree効果を確率的に扱え、役割別順位を得られる。
- **デメリット**：実装と説明がPageRankより馴染みにくく、base set依存は残る。
- **実装要点**：弱連結成分ごとの定常分布と正規化を確認し、方向を誤って転置しない。

### 057. CycleRank — 82点

- **評価**：82/100（カテゴリ順位17/40）
- **典拠**：[CycleRank](https://pmc.ncbi.nlm.nih.gov/articles/PMC7544349/)
- **概要**：開始記事を含む短い有向cycleへの参加度を使い、戻って来られる意味的まとまりを順位づけする。
- **メリット**：Wikipediaの循環リンク構造を利用し、単なる被リンク数と違う開始点局所性を表せる。
- **デメリット**：cycle列挙または近似が重く、DAG的関係を過小評価する。
- **実装要点**：最大cycle長を小さく制限し、相互リンク2-cycleと長いcycleを別特徴にする。

### 058. SimRank — 81点

- **評価**：81/100（カテゴリ順位18/40）
- **典拠**：[Jeh & Widom, “SimRank”](https://doi.org/10.1145/775047.775126)
- **概要**：「似た記事から参照される二記事は似る」という再帰定義で構造的類似度を求める。
- **メリット**：直接リンクがなくても参照文脈が似る記事を発見できる。
- **デメリット**：全対反復が高コストで、自己類似と減衰係数に敏感。
- **実装要点**：開始記事対だけの近似、低rank化、候補集合制限を使い、全対行列を避ける。

### 059. Katz指数 — 80点

- **評価**：80/100（カテゴリ順位19/40）
- **典拠**：[Katz centrality documentation and source](https://networkx.org/documentation/stable/reference/algorithms/generated/networkx.algorithms.centrality.katz_centrality.html)
- **概要**：二記事を結ぶ全walkを長さに応じて `α^l` で減衰し、直接・間接経路を合計する。
- **メリット**：最短路一つでなく複数の弱い経路を関連度へ反映できる。
- **デメリット**：`α < 1/λmax` が必要で、ハブ間のwalk爆発と有向方向選択に注意が要る。
- **実装要点**：スペクトル半径から安全なαを選び、開始点からのKatz proximityとして計算する。

### 060. 時間安定性・リンク継続期間重み — 79点

- **評価**：79/100（カテゴリ順位20/40）
- **典拠**：[MediaWiki API:Revisions](https://www.mediawiki.org/wiki/API:Revisions)、[REST page history](https://www.mediawiki.org/wiki/API:REST_API/Reference)
- **概要**：複数revisionでリンクが存在した期間、追加・削除回数、最終更新からの時間を関係の安定度として使う。
- **メリット**：一時的編集やニュース由来の揺れと、長期に維持された関係を区別できる。
- **デメリット**：履歴取得と各版parseが非常に高価で、単なる古さを品質と誤認し得る。
- **実装要点**：全履歴でなく定期snapshotまたは変更点だけを蓄積し、安定度と新規性を別特徴にする。

### 061. Heat Kernel PageRank — 78点

- **評価**：78/100（カテゴリ順位21/40）
- **典拠**：[Chung, “The heat kernel as the pagerank of a graph”](https://pmc.ncbi.nlm.nih.gov/articles/PMC2148367/)
- **概要**：walk長をPoisson型に重み付けする熱拡散で、seedから時間 `t` に応じた局所順位を計算する。
- **メリット**：拡散尺度を連続的に制御でき、局所クラスタ抽出と順位を一体化できる。
- **デメリット**：PPRより実装例が少なく、`t`の意味を利用者へ説明しにくい。
- **実装要点**：複数の`t`で安定上位を比較し、疎行列指数の近似または局所Monte Carloを使う。

### 062. Random Walk with Restart関連度 — 77点

- **評価**：77/100（カテゴリ順位22/40）
- **典拠**：[Random Walk with Restart](https://pmc.ncbi.nlm.nih.gov/articles/PMC6426185/)
- **概要**：開始記事から辺重みに従って遷移し、一定確率で開始点へ戻る訪問確率を関連度にする。
- **メリット**：直近と多hopの関係を一つの確率にまとめ、増分power iterationを使いやすい。
- **デメリット**：PPRとほぼ同系統で、restart率と部分グラフ境界に敏感。
- **実装要点**：探索用sampling版と、取得後の確定行列版を分けて同じ名称で混同しない。

### 063. 共引用（co-citation） — 76点

- **評価**：76/100（カテゴリ順位23/40）
- **典拠**：[Small, 1973](https://doi.org/10.1002/asi.4630240406)
- **概要**：二記事へ同時にリンクする第三の記事数を、両者が同じ文脈で参照される強さとみなす。
- **メリット**：二記事間に直接リンクがなくても、共通の利用文脈から関係を得られる。
- **デメリット**：巨大な被リンク集合が必要で、高人気記事同士を過大評価する。
- **実装要点**：生数だけでなくWLM、Jaccard、Adamic–Adarの正規化値も併記する。

### 064. 書誌結合型の共通出リンク — 75点

- **評価**：75/100（カテゴリ順位24/40）
- **典拠**：[Kessler, bibliographic coupling](https://doi.org/10.1002/asi.5090140103)
- **概要**：二記事が共通して参照するWikipedia記事の数または重みを、説明対象の近さとして使う。
- **メリット**：被リンクよりオンライン取得しやすく、同じ基礎概念を説明する記事を結べる。
- **デメリット**：一般概念へのリンクが多い記事同士を過大評価し、リンク方針差に左右される。
- **実装要点**：共通先の逆次数IDFを掛け、脚注・関連項目など節typeを区別する。

### 065. Adamic–Adar指数 — 74点

- **評価**：74/100（カテゴリ順位25/40）
- **典拠**：[NetworkX Link Prediction](https://networkx.org/documentation/stable/reference/algorithms/link_prediction.html)
- **概要**：共通近傍 `z` の寄与を `1/log degree(z)` で減衰し、希少な共通文脈を強く評価する。
- **メリット**：Jaccardよりハブ共通近傍を抑え、Wikipediaの一般語記事による雑音を減らせる。
- **デメリット**：通常は無向グラフ前提で、関係方向と辺型を失う。
- **実装要点**：被リンク共通と出リンク共通を別々に計算し、低degreeの例外処理を入れる。

### 066. Resource Allocation指数 — 73点

- **評価**：73/100（カテゴリ順位26/40）
- **典拠**：[NetworkX `resource_allocation_index`](https://networkx.org/documentation/stable/reference/algorithms/link_prediction.html)
- **概要**：各共通近傍の寄与を `1/degree(z)` とし、高次数ハブをAdamic–Adarより強く減衰する。
- **メリット**：一覧・年・国のような巨大ハブに頑健で、式が単純である。
- **デメリット**：希少近傍一つの影響が大きく、誤リンクや小規模カテゴリに敏感。
- **実装要点**：最低degree、辺type、方向ごとに計算し、単独でなく他スコアへ加える。

### 067. Jaccard係数 — 72点

- **評価**：72/100（カテゴリ順位27/40）
- **典拠**：[NetworkX Jaccard coefficient](https://networkx.org/documentation/stable/reference/algorithms/link_prediction.html)
- **概要**：二記事の近傍集合について `|A∩B|/|A∪B|` を計算し、共有比率を関連度にする。
- **メリット**：0〜1で解釈しやすく、記事ごとの近傍規模差を正規化できる。
- **デメリット**：共通近傍の重要度を区別せず、低次数記事の小さな一致で高値になり得る。
- **実装要点**：出・入・相互・カテゴリの各集合に別々に適用し、集合サイズも表示する。

### 068. Sørensen–Dice係数 — 71点

- **評価**：71/100（カテゴリ順位28/40）
- **典拠**：[SciPy Dice dissimilarity](https://docs.scipy.org/doc/scipy/reference/generated/scipy.spatial.distance.dice.html)
- **概要**：`2|A∩B|/(|A|+|B|)` で近傍集合の重なりを測り、Jaccardより共通部分をやや強く評価する。
- **メリット**：対称で0〜1、短い近傍集合の重なりを直観的に扱える。
- **デメリット**：ハブ近傍の識別力を補正せず、Jaccardと情報が重複する。
- **実装要点**：どちらかが空集合の場合の規則を固定し、同一候補集合でJaccardとのnDCGを比較する。

### 069. Overlap coefficient — 70点

- **評価**：70/100（カテゴリ順位29/40）
- **典拠**：[Stanford IR book: vector and set similarity](https://nlp.stanford.edu/IR-book/html/htmledition/dot-products-1.html)
- **概要**：`|A∩B|/min(|A|,|B|)` とし、小さい方の近傍が大きい方にどれだけ包含されるかを測る。
- **メリット**：専門記事が概説記事の近傍部分集合になる関係を高く評価できる。
- **デメリット**：近傍が一つしかない記事でも一致すれば1になり、過大評価しやすい。
- **実装要点**：最小集合サイズの下限と信頼度補正を加え、包含方向も併記する。

### 070. Common Neighbors生数 — 69点

- **評価**：69/100（カテゴリ順位30/40）
- **典拠**：[NetworkX common neighbors and link prediction](https://networkx.org/documentation/stable/reference/algorithms/link_prediction.html)
- **概要**：二記事が共有する隣接記事数を、そのまま局所構造関連度として使う。
- **メリット**：最も単純で高速、具体的な共通記事を根拠として列挙できる。
- **デメリット**：次数に強く依存し、著名記事ほど高くなりやすい。
- **実装要点**：生数は表示説明に使い、順位には正規化指標またはdegree統制を併用する。

### 071. PMI／NPMI共起関連度 — 68点

- **評価**：68/100（カテゴリ順位31/40）
- **典拠**：[Bouma, Normalized PMI](https://arxiv.org/abs/0901.2575)
- **概要**：記事対が同じ文書、節、閲覧session等で共起する確率を周辺確率で割り、偶然以上の結びつきを測る。
- **メリット**：人気記事同士の単なる高頻度共起を補正し、NPMIは比較可能な範囲に収まる。
- **デメリット**：低頻度対で不安定になり、sessionデータは公開範囲とprivacy制約がある。
- **実装要点**：共起単位と平滑化を固定し、最小support未満を順位対象から外す。

### 072. 重み付き最短路関連度 — 67点

- **評価**：67/100（カテゴリ順位32/40）
- **典拠**：[NetworkX shortest paths](https://networkx.org/documentation/stable/reference/algorithms/shortest_paths.html)
- **概要**：辺の不関連度を距離へ変換し、開始記事からの最短累積距離または `1/(1+d)` で順位づけする。
- **メリット**：経路をそのまま説明でき、方向・関係型・深度を扱いやすい。
- **デメリット**：一本の偶然なshort-cutに敏感で、複数経路の支持を無視する。
- **実装要点**：距離が非負になる校正を行い、上位には最短路と代替路を表示する。

### 073. Communicability — 66点

- **評価**：66/100（カテゴリ順位33/40）
- **典拠**：[NetworkX communicability](https://networkx.org/documentation/stable/reference/algorithms/communicability_alg.html)
- **概要**：隣接行列指数 `exp(A)` により、短いwalkを強く、長いwalkを階乗で減衰して全経路を集約する。
- **メリット**：直接・間接の多数経路を連続的に統合し、bridge以外の冗長な結びつきを反映する。
- **デメリット**：大規模・有向グラフで計算が重く、高密度部分の値が支配的になる。
- **実装要点**：候補誘導部分グラフでKrylov近似を使い、対称化方法を明記する。

### 074. Current-flow／Resistance distance — 65点

- **評価**：65/100（カテゴリ順位34/40）
- **典拠**：[NetworkX current-flow centrality](https://networkx.org/documentation/stable/reference/algorithms/centrality.html#current-flow-closeness)
- **概要**：辺を抵抗とみなし、二記事間を通る全経路の電気的有効抵抗または電流で近さを測る。
- **メリット**：最短路だけでなく代替経路全体を反映し、橋の重要性を自然に表す。
- **デメリット**：連結無向グラフと線形方程式が基本で、大規模有向グラフには高価。
- **実装要点**：関係重みをconductanceへ正しく変換し、小さな連結成分または近似solverで使う。

### 075. 媒介中心性による橋記事順位 — 64点

- **評価**：64/100（カテゴリ順位35/40）
- **典拠**：[NetworkX betweenness centrality](https://networkx.org/documentation/stable/reference/algorithms/centrality.html)
- **概要**：取得グラフの最短路に記事が現れる割合を、主題間をつなぐbridge度として使う。
- **メリット**：単純な人気度で見落とす分野横断記事を発見できる。
- **デメリット**：全対最短路は重く、サンプル境界や偶然の細い接続で値が大きくなる。
- **実装要点**：seed集合に限定したsubset版またはsampling近似を使い、関連度とbridge度を別軸表示する。

### 076. Closeness centrality — 63点

- **評価**：63/100（カテゴリ順位36/40）
- **典拠**：[NetworkX closeness centrality](https://networkx.org/documentation/stable/reference/algorithms/centrality.html)
- **概要**：取得部分グラフ内の他記事への平均最短距離の逆数で、局所群の中心記事を順位づけする。
- **メリット**：クラスタの案内役・中心ラベルを選ぶ用途に分かりやすい。
- **デメリット**：開始記事との関連度ではなく、非連結性と探索境界に非常に敏感。
- **実装要点**：harmonic variantまたは成分内正規化を使い、主順位とは分離する。

### 077. Eigenvector centrality — 62点

- **評価**：62/100（カテゴリ順位37/40）
- **典拠**：[NetworkX eigenvector centrality](https://networkx.org/documentation/stable/reference/algorithms/centrality.html)
- **概要**：高得点記事と接続する記事を高くする主固有ベクトルで、取得群の構造的威信を表す。
- **メリット**：degreeより隣接先の質を反映し、表示サイズの補助尺度になる。
- **デメリット**：開始点非依存で、非連結・有向グラフの向きと収束に注意が必要。
- **実装要点**：最大成分、左／右固有ベクトル、重みの意味を固定し、PPRと混同しない。

### 078. Reciprocal Rank Fusion（RRF） — 61点

- **評価**：61/100（カテゴリ順位38/40）
- **典拠**：[Cormack, Clarke & Büttcher, RRF](https://research.google/pubs/reciprocal-rank-fusion-outperforms-condorcet-and-individual-rank-learning-methods/)
- **概要**：WLM、BM25、SBERT、PPR等の各順位を `Σ 1/(k+rank)` で教師なし統合する。
- **メリット**：異なる尺度のscore校正をせず、外れた一手法に比較的頑健な合成順位を作れる。
- **デメリット**：順位間隔と信頼度を捨て、似た手法を複数入れると重複投票になる。
- **実装要点**：相関の高いrankerをまとめ、RRF定数と欠損候補の扱いを版管理する。

### 079. Maximal Marginal Relevance（MMR） — 60点

- **評価**：60/100（カテゴリ順位39/40）
- **典拠**：[Goldstein & Carbonell, MMR](https://aclanthology.org/X98-1025/)
- **概要**：開始記事への関連度を高めつつ、既に選んだ記事との最大類似を罰して冗長性を抑える。
- **メリット**：上位が同じ小主題で埋まるのを防ぎ、限られた表示枠で主題幅を出せる。
- **デメリット**：多様性係数に敏感で、必要な密なクラスタまで間引くことがある。
- **実装要点**：順位づけ後の表示選抜に使い、元scoreとMMR後scoreを両方保存する。

### 080. xQuADによるaspect多様化 — 59点

- **評価**：59/100（カテゴリ順位40/40）
- **典拠**：[Santos, Explicit search result diversification](https://theses.gla.ac.uk/4106/)
- **概要**：カテゴリ、Wikidata型、節トピックをquery aspectとし、関連度と未充足aspectの被覆確率で再順位づけする。
- **メリット**：多様性を単なる距離でなく、説明可能な話題側面のcoverageとして制御できる。
- **デメリット**：aspect抽出と事前確率が必要で、誤分類が順位を直接歪める。
- **実装要点**：aspectなしのMMRをbaselineにし、coverageと関連度のトレードオフをUIで調整可能にする。

---

## 3. 点群化と粒度フィルタ（40件）

### 081. Sentence-BERT記事埋め込み — 98点

- **評価**：98/100（カテゴリ順位1/40）
- **典拠**：[Reimers & Gurevych, Sentence-BERT](https://aclanthology.org/D19-1410/)
- **概要**：記事の導入文、節要約、タイトルをdense vectorへ変換し、各記事を高次元点として表現する。
- **メリット**：語彙差を越えた意味近傍を作れ、逐次追加記事も再学習なしで埋め込める。
- **デメリット**：長文切詰め、model bias、言語品質に依存し、リンク構造を直接は使わない。
- **実装要点**：タイトル＋導入部を基本入力にし、モデル名、版、最大token、正規化有無を座標metadataに残す。

### 082. TADW（リンク＋本文の共同埋め込み） — 97点

- **評価**：97/100（カテゴリ順位2/40）
- **典拠**：[Yang et al., Text-Associated DeepWalk](https://www.ijcai.org/Proceedings/15/Papers/299.pdf)
- **概要**：DeepWalkの行列分解解釈へ記事テキスト特徴を加え、リンク構造と内容を同じ潜在空間へ写す。
- **メリット**：Wikipediaで重要なリンクと本文の両方を保持し、片方が疎でも補完できる。
- **デメリット**：全体再学習が必要で、逐次記事追加と大規模語彙行列が難しい。
- **実装要点**：小規模版でablationを行い、オンライン表示には定期snapshot学習＋新規記事の近似射影を使う。

### 083. UMAP — 96点

- **評価**：96/100（カテゴリ順位3/40）
- **典拠**：[McInnes, Healy & Melville, UMAP](https://arxiv.org/abs/1802.03426)、[UMAP documentation](https://umap-learn.readthedocs.io/)
- **概要**：記事ベクトルの近傍グラフをfuzzy simplicial setとして構成し、低次元で近傍関係を再現する。
- **メリット**：高速で2D・3Dを選べ、t-SNEよりglobal structureを保つ場合があり、transformも可能。
- **デメリット**：`n_neighbors`と`min_dist`で見た目が大きく変わり、軸に意味はない。
- **実装要点**：cosine距離、固定seed、複数parameterでtrustworthinessを比較し、グラフ辺は別途描く。

### 084. LSA／Truncated SVD — 95点

- **評価**：95/100（カテゴリ順位4/40）
- **典拠**：[Deerwester et al., Latent Semantic Analysis](https://ideas.repec.org/a/bla/jamest/v41y1990i6p391-407.html)
- **概要**：記事–語TF–IDF行列を低rank分解し、記事を潜在語彙空間の点として表す。
- **メリット**：疎行列に高速で、決定的なbaselineを作りやすく、上位寄与語を説明できる。
- **デメリット**：線形で多義性を粗く混合し、新語・別言語・非線形構造に弱い。
- **実装要点**：TF–IDF設定とrankを固定し、explained varianceと近傍precisionを記録する。

### 085. GraphSAGE — 94点

- **評価**：94/100（カテゴリ順位5/40）
- **典拠**：[Hamilton, Ying & Leskovec, GraphSAGE](https://proceedings.neurips.cc/paper_files/paper/2017/hash/5dd9db5e033da9c6fb5ba83c7a7ebea9-Abstract.html)
- **概要**：記事特徴とsampled neighborsを集約する関数を学習し、未学習の新記事にも埋め込みを生成する。
- **メリット**：オンライン逐次追加に適したinductive性があり、本文とグラフの双方を使える。
- **デメリット**：学習データと目的関数が必要で、深層化するとoversmoothingと高次数sampling biasが起きる。
- **実装要点**：1〜2層、型別neighbor sampling、リンク予測lossから始め、SBERT単体と比較する。

### 086. node2vec — 93点

- **評価**：93/100（カテゴリ順位6/40）
- **典拠**：[Grover & Leskovec, node2vec](https://pmc.ncbi.nlm.nih.gov/articles/PMC5108654/)
- **概要**：BFS的／DFS的探索を `p,q` で調整する二次random walkを生成し、skip-gramで記事点を学習する。
- **メリット**：局所コミュニティと構造役割のどちらを重視するか調整でき、実装が豊富。
- **デメリット**：有向・多重・型付き辺をそのまま表せず、parameterと乱数への依存が強い。
- **実装要点**：辺typeごとの遷移重みを設計し、複数seedで近傍安定性を測る。

### 087. DeepWalk — 92点

- **評価**：92/100（カテゴリ順位7/40）
- **典拠**：[Perozzi, Al-Rfou & Skiena, DeepWalk](https://research.google/pubs/deepwalk-online-learning-of-social-representations/)
- **概要**：グラフrandom walkを文、記事を語とみなし、skip-gramで近接記事のvectorを学習する。
- **メリット**：単純・スケーラブルで、局所高次近接をdense vectorへ変換できる。
- **デメリット**：本文を使わず、未見記事のinductive追加と方向・辺型表現が弱い。
- **実装要点**：walk length、window、walk数、seedを固定し、追加時は定期再学習の揺れを監視する。

### 088. Laplacian Eigenmaps — 91点

- **評価**：91/100（カテゴリ順位8/40）
- **典拠**：[Belkin & Niyogi, Laplacian Eigenmaps](https://newtraell.cs.uchicago.edu/research/publications/techreports/TR-2002-01)
- **概要**：近接記事を近く置くLaplacian二次形式を最小化し、低い非自明固有ベクトルを座標にする。
- **メリット**：グラフ構造との対応が明確で、spectral clusteringと同じ数学を共有する。
- **デメリット**：大規模固有値計算、非連結成分、hubによる座標圧縮に弱い。
- **実装要点**：normalized Laplacian、成分処理、固有vectorの符号整合を固定する。

### 089. Variational Graph Autoencoder（VGAE） — 90点

- **評価**：90/100（カテゴリ順位9/40）
- **典拠**：[Kipf & Welling, VGAE](https://arxiv.org/abs/1611.07308)
- **概要**：GCN encoderで記事ごとの潜在分布を学習し、内積decoderでリンクを再構成する。
- **メリット**：本文特徴を併用でき、不確実性を持つ滑らかな点群とリンク予測を同時に得られる。
- **デメリット**：通常は無向静的グラフ前提で、負例samplingと学習costが必要。
- **実装要点**：潜在次元をまず64程度で学習し、2D/3Dへ別途射影して直接3次元学習と比較する。

### 090. metapath2vec — 89点

- **評価**：89/100（カテゴリ順位10/40）
- **典拠**：[Dong, Chawla & Swami, metapath2vec](https://doi.org/10.1145/3097983.3098036)
- **概要**：記事–カテゴリ–記事、記事–人物–作品など型列を指定したwalkで異種グラフを埋め込む。
- **メリット**：関係型を潰さず、目的に合う意味経路を点群へ反映できる。
- **デメリット**：metapathを人手で設計し、型頻度と長さの偏りを調整する必要がある。
- **実装要点**：各metapathの意味と遷移確率を表示し、本文リンクだけのnode2vecをbaselineにする。

### 091. LINE — 88点

- **評価**：88/100（カテゴリ順位11/40）
- **典拠**：[Tang et al., LINE](https://arxiv.org/abs/1503.03578)
- **概要**：辺の一次近接と共有近傍の二次近接をnegative samplingで別々に最適化する。
- **メリット**：大規模・重み付き・有向ネットワークへ適用でき、計算量が辺数に近い。
- **デメリット**：長い多hop構造と本文意味を直接扱わず、一次・二次vectorの統合が必要。
- **実装要点**：方向付き二次近接を使い、edge weightの極端値を対数圧縮する。

### 092. HOPE — 87点

- **評価**：87/100（カテゴリ順位12/40）
- **典拠**：[Ou et al., HOPE](https://www.kdd.org/kdd2016/papers/files/rfp0184-ouA.pdf)
- **概要**：Katz、Rooted PageRank等の非対称高次近接行列を一般化SVDで分解し、source／target vectorを得る。
- **メリット**：有向Wikipediaリンクの非対称性と長距離関係を保持できる。
- **デメリット**：近接行列構築と分解が大規模で重く、左右二vectorの表示方法が難しい。
- **実装要点**：outgoing用とincoming用を連結するか別点群にし、対称化で意味を失わない。

### 093. Explicit Semantic Analysis（ESA） — 86点

- **評価**：86/100（カテゴリ順位13/40）
- **典拠**：[Gabrilovich & Markovitch, ESA](https://gabrilovich.com/publications/papers/Gabrilovich2007CSR.pdf)
- **概要**：語や記事をWikipedia概念軸の疎vectorへ写し、概念重みから記事間関連を表す。
- **メリット**：次元が人間可読な記事概念で、語彙一致を越えた説明可能な意味空間を作れる。
- **デメリット**：次元が非常に大きく、index構築と更新が重く、対象Wikipediaへの循環的依存がある。
- **実装要点**：top-k概念だけ保持し、概念軸の版と重みを表示根拠として保存する。

### 094. Doc2Vec／Paragraph Vector — 85点

- **評価**：85/100（カテゴリ順位14/40）
- **典拠**：[Le & Mikolov, Paragraph Vector](https://proceedings.mlr.press/v32/le14.html)
- **概要**：記事IDvectorと語vectorを同時学習し、各記事を固定長dense pointにする。
- **メリット**：可変長記事を一vectorへまとめ、TF–IDFより語順・局所文脈を一部捉える。
- **デメリット**：品質がtraining corpusとparameterに敏感で、最新transformerより性能が劣る場合が多い。
- **実装要点**：短い導入部と全文を比較し、未見記事infer時の反復とseedを固定する。

### 095. GraphTSNE — 84点

- **評価**：84/100（カテゴリ順位15/40）
- **典拠**：[GraphTSNE](https://arxiv.org/abs/1904.06915)
- **概要**：グラフ構造と頂点属性をgraph neural networkで統合し、t-SNE型目的で可視化座標を学習する。
- **メリット**：リンクと本文特徴を直接2D/3D配置へ反映し、クラスタ分離を狙える。
- **デメリット**：学習・parameter調整が重く、座標の再現性とout-of-sample追加が難しい。
- **実装要点**：評価用に構造のみ、属性のみ、統合の3条件で近傍保存を比較する。

### 096. Isomap — 83点

- **評価**：83/100（カテゴリ順位16/40）
- **典拠**：[Tenenbaum, de Silva & Langford, Isomap](https://doi.org/10.1126/science.290.5500.2319)
- **概要**：記事vectorのkNN graph上の最短路をmanifold geodesicとみなし、MDSで低次元化する。
- **メリット**：曲がった大域構造を線形PCAより展開でき、距離保存の意味が明瞭。
- **デメリット**：short-circuit辺と非連結近傍graphに弱く、全対最短路が重い。
- **実装要点**：kの連結性を検査し、landmark Isomapで大規模化する。

### 097. Locally Linear Embedding（LLE） — 82点

- **評価**：82/100（カテゴリ順位17/40）
- **典拠**：[Roweis & Saul, LLE](https://doi.org/10.1126/science.290.5500.2323)
- **概要**：各記事を近傍記事の線形結合で再構成する重みを求め、その重みを保つ低次元点を計算する。
- **メリット**：局所manifold形状を保ち、非線形な曲面状分布を展開できる。
- **デメリット**：近傍数、noise、密度差に敏感で、global distanceは保証しない。
- **実装要点**：regularizationと近傍数をsweepし、孤立点を別処理する。

### 098. struc2vec — 81点

- **評価**：81/100（カテゴリ順位18/40）
- **典拠**：[Ribeiro, Saverese & Figueiredo, struc2vec](https://arxiv.org/abs/1704.03165)
- **概要**：直接近接より次数列などの構造役割が似た記事を近く埋め込む。
- **メリット**：互いに遠くても「一覧」「概説」「橋」など同じ役割の記事を発見できる。
- **デメリット**：意味主題の近さを表さず、ユーザーが期待する関連記事点群とはずれる場合がある。
- **実装要点**：主題embeddingと別のrole viewとして提供し、同一平面に無説明で混ぜない。

### 099. Poincaré embeddings — 80点

- **評価**：80/100（カテゴリ順位19/40）
- **典拠**：[Nickel & Kiela, Poincaré Embeddings](https://proceedings.neurips.cc/paper_files/paper/2017/hash/59dfa2df42d9e3d41f5b02bfc32229dd-Abstract.html)
- **概要**：カテゴリやWikidataの階層を負曲率のPoincaré ballへ埋め込み、半径方向に一般→具体を配置する。
- **メリット**：木状・scale-free構造を少ない次元で低歪みに表しやすい。
- **デメリット**：一般リンクグラフのcycleや非階層関係には適合せず、Euclidean描画へ変換すると歪む。
- **実装要点**：階層辺だけで学習し、双曲距離と表示投影の双方を明示する。

### 100. PHATE — 79点

- **評価**：79/100（カテゴリ順位20/40）
- **典拠**：[Moon et al., PHATE](https://www.nature.com/articles/s41587-019-0336-3)
- **概要**：近傍affinityの拡散をpotential distanceへ変換し、局所と大域の連続構造を点群へ写す。
- **メリット**：分岐・連続遷移・クラスタを同時に見せる設計で、noiseに比較的頑健。
- **デメリット**：記事グラフでの標準実証は少なく、diffusion timeとkNN設定が必要。
- **実装要点**：SBERTまたはgraph embeddingを入力し、UMAP・PCAとtrustworthiness／continuityを比較する。

### 101. TriMap — 78点

- **評価**：78/100（カテゴリ順位21/40）
- **典拠**：[Amid & Warmuth, TriMap](https://arxiv.org/abs/1910.00204)
- **概要**：記事三つ組の相対距離制約を満たすよう低次元化し、局所だけでなく大域配置の維持を狙う。
- **メリット**：t-SNE／UMAPよりglobal structureを保つ設計で、主題群同士の相対位置を見やすくできる。
- **デメリット**：triplet samplingと重みで結果が変わり、局所クラスタ分離が弱い場合がある。
- **実装要点**：同じ前処理・seedでUMAPと比較し、global distance correlationも測る。

### 102. PaCMAP — 77点

- **評価**：77/100（カテゴリ順位22/40）
- **典拠**：[Wang et al., PaCMAPを含む次元削減比較](https://www.jmlr.org/papers/v22/20-1061.html)
- **概要**：近傍、mid-near、遠方の点対を段階的に最適化し、局所と大域の均衡を取る。
- **メリット**：クラスタ内部とクラスタ間配置の両方を比較的安定して見せやすい。
- **デメリット**：Wikipediaグラフ固有の辺方向を使わず、pair samplingと初期値に依存する。
- **実装要点**：SBERT、node2vec双方を入力し、近傍保持とクラスタ間距離の再現を別々に評価する。

### 103. LargeVis — 76点

- **評価**：76/100（カテゴリ順位23/40）
- **典拠**：[Tang et al., LargeVis](https://arxiv.org/abs/1602.00370)
- **概要**：近似kNN graphを構築し、negative sampling付き確率modelを非同期SGDで低次元化する。
- **メリット**：大量記事の2D/3D点群に対応し、計算量をほぼ線形へ抑える設計である。
- **デメリット**：近似近傍誤差と非決定性があり、最新実装の保守性を確認する必要がある。
- **実装要点**：同規模のUMAPと時間・memory・trustworthinessを比較し、乱数を固定する。

### 104. Minimum-Distortion Embedding（PyMDE） — 75点

- **評価**：75/100（カテゴリ順位24/40）
- **典拠**：[Agrawal, Ali & Boyd, MDE](https://arxiv.org/abs/2103.02559)
- **概要**：近づけたい記事対、離したい記事対、標準化制約を明示し、総distortionを最小化する。
- **メリット**：リンク・非リンク・意味類似を同じ枠組みで設計でき、目的関数が明確。
- **デメリット**：pairとdistortion関数の選定責任が大きく、局所最適と計算costがある。
- **実装要点**：相互リンクをattractive、hard negativeをrepulsiveにし、distortion内訳を保存する。

### 105. PPR上位kノードフィルタ — 74点

- **評価**：74/100（カテゴリ順位25/40）
- **典拠**：[Personalized PageRank](https://arxiv.org/abs/1304.4658)
- **概要**：開始記事にpersonalizeしたPPR上位 `k` 頂点だけを表示点群に残す。
- **メリット**：開始点との多hop関連を保ったまま、表示規模を直接制御できる。
- **デメリット**：橋や少数主題が順位外へ落ち、境界辺が途切れて見える。
- **実装要点**：top-kに加え、各上位への代表最短路とコミュニティquotaを保持する。

### 106. Leidenコミュニティ縮約 — 73点

- **評価**：73/100（カテゴリ順位26/40）
- **典拠**：[Traag, Waltman & van Eck, Leiden](https://doi.org/10.1038/s41598-019-41695-z)
- **概要**：Leidenで得たコミュニティをsupernodeへ集約し、ズーム時だけ構成記事を展開する。
- **メリット**：内部非連結問題を改善した高品質clusterで、大規模グラフを段階表示できる。
- **デメリット**：resolutionで粒度が変わり、communityを実在する分類と誤認しやすい。
- **実装要点**：複数resolutionの階層を保存し、superedgeには集約辺数・重み・type内訳を持たせる。

### 107. seed中心ego-radiusフィルタ — 72点

- **評価**：72/100（カテゴリ順位27/40）
- **典拠**：[NetworkX ego graph](https://networkx.org/documentation/stable/reference/generated/networkx.generators.ego.ego_graph.html)
- **概要**：開始記事からhop距離または重み付き距離 `r` 以内の記事だけを表示する。
- **メリット**：基準が直観的で、探索深度と表示範囲を一致させやすい。
- **デメリット**：境界直外の重要記事と、radius内の巨大ハブを適切に扱えない。
- **実装要点**：hopとweighted radiusを切替え可能にし、切断辺数を境界badgeで示す。

### 108. k-coreフィルタ — 71点

- **評価**：71/100（カテゴリ順位28/40）
- **典拠**：[NetworkX core decomposition](https://networkx.org/documentation/stable/reference/algorithms/core.html)
- **概要**：各頂点の内部次数が少なくとも `k` となる最大部分グラフを残し、密な核を抽出する。
- **メリット**：高速・入れ子構造で、周縁ノイズを段階的に除ける。
- **デメリット**：意味的に重要なleaf、経路端、橋記事を落とす。
- **実装要点**：core numberを属性として残し、leafを消去せず必要時に展開できるようにする。

### 109. 親ごとのtop-k辺フィルタ — 70点

- **評価**：70/100（カテゴリ順位29/40）
- **典拠**：[Large graph sparsification for visualization](https://arxiv.org/abs/1708.08659)
- **概要**：各記事から重み上位 `k` 本の辺だけを残し、ノードごとの最低接続を保証する。
- **メリット**：単純なglobal thresholdより低次数頂点を孤立させにくく、辺数を `O(kn)` にできる。
- **デメリット**：非対称選択になり、ハブ側の多数の意味ある辺を強く削る。
- **実装要点**：union top-kとmutual top-kを比較し、元degreeと削除辺数を表示する。

### 110. Mutual kNNグラフ — 69点

- **評価**：69/100（カテゴリ順位30/40）
- **典拠**：[UMAPの近傍グラフ構成](https://arxiv.org/abs/1802.03426)
- **概要**：記事AとBが互いに上位k近傍に入る場合だけ辺を残し、対称で強い点群近傍を作る。
- **メリット**：一方向だけの近傍誤差とhubnessを抑え、cluster境界が明瞭になりやすい。
- **デメリット**：密度差のある領域で疎な側が分断され、元の有向リンクを失う。
- **実装要点**：表示用近傍graphとして作り、元のWikipedia辺は別レイヤーに保持する。

### 111. Disparity Filter — 68点

- **評価**：68/100（カテゴリ順位31/40）
- **典拠**：[Serrano, Boguñá & Vespignani, 2009](https://doi.org/10.1073/pnas.0808904106)
- **概要**：各頂点のstrengthを基準とするnull modelに対し、局所的に有意な重み辺だけをbackboneへ残す。
- **メリット**：一つのglobal thresholdで消える弱い地域的関係を保ち、多尺度性を維持できる。
- **デメリット**：辺重みの分布仮定と有意水準に依存し、多重検定問題がある。
- **実装要点**：方向別・辺type別に適用し、alphaと残存率を記録する。

### 112. スペクトル・スパース化 — 67点

- **評価**：67/100（カテゴリ順位32/40）
- **典拠**：[Spielman & Teng, spectral sparsification](https://doi.org/10.1137/08074489X)
- **概要**：有効抵抗等に基づき辺をsamplingし、Laplacian二次形式を近似保存する疎グラフを作る。
- **メリット**：cut、diffusion、spectral layoutの性質を理論的に保ちながら辺数を減らせる。
- **デメリット**：実装が複雑で、有向・型付き・意味重みへの直接保証は弱い。
- **実装要点**：解析用無向射影に適用し、元辺typeへ復元可能なedge mapを保持する。

### 113. Steiner tree近似 — 66点

- **評価**：66/100（カテゴリ順位33/40）
- **典拠**：[NetworkX Steiner tree](https://networkx.org/documentation/stable/reference/algorithms/generated/networkx.algorithms.approximation.steinertree.steiner_tree.html)
- **概要**：選んだ重要記事群を接続する低cost部分木を近似し、必要な中継記事だけ加える。
- **メリット**：上位記事間の文脈経路を残した非常に細い説明骨格を作れる。
- **デメリット**：terminal選択に依存し、cycleや代替経路を捨てる。
- **実装要点**：terminalをPPR上位＋各community代表にし、木外の強辺を少数戻す。

### 114. Minimum Spanning Tree／Forest — 65点

- **評価**：65/100（カテゴリ順位34/40）
- **典拠**：[NetworkX minimum spanning edges](https://networkx.org/documentation/stable/reference/algorithms/generated/networkx.algorithms.tree.mst.minimum_spanning_edges.html)
- **概要**：関連度を距離へ変換し、全頂点を最小総costでつなぐ木またはforestを表示骨格にする。
- **メリット**：辺数がほぼ `n-1` となり、全ノードへの一意な経路を示せる。
- **デメリット**：cycleと冗長性を全て落とし、一辺の誤重みが大きく影響する。
- **実装要点**：MSTを常時表示し、追加の強辺をinteraction時に重ねる二層表示にする。

### 115. k-trussフィルタ — 64点

- **評価**：64/100（カテゴリ順位35/40）
- **典拠**：[NetworkX `k_truss`](https://networkx.org/documentation/stable/reference/algorithms/generated/networkx.algorithms.core.k_truss.html)
- **概要**：各辺が少なくとも `k-2` 個のtriangleに属する部分グラフを残す。
- **メリット**：k-coreより辺の局所結束を強く要求し、密な意味clusterの核を得やすい。
- **デメリット**：二部・階層・有向的な関係や橋をほぼ消し、triangleの少ない正当な構造に不向き。
- **実装要点**：相互リンク無向graphで使い、triangle不足を無関係の証拠とみなさない。

### 116. Infomapコミュニティ縮約 — 63点

- **評価**：63/100（カテゴリ順位36/40）
- **典拠**：[Rosvall & Bergstrom, Infomap](https://www.mapequation.org/assets/publications/RosvallBergstromPNAS2008Full.pdf)
- **概要**：random walkを短い符号で記述できるmoduleへ分け、moduleをsuperpoint化する。
- **メリット**：有向・重み付きの流れを反映し、読者遷移やリンクwalkと相性がよい。
- **デメリット**：強いflow trapが細粒度moduleを作り、結果がteleport設定に依存する。
- **実装要点**：階層Infomapを使い、module内外flowと代表記事をsupernode属性にする。

### 117. Louvainコミュニティ縮約 — 62点

- **評価**：62/100（カテゴリ順位37/40）
- **典拠**：[Blondel et al., Louvain](https://perso.uclouvain.be/vincent.blondel/research/louvain.html)
- **概要**：modularityを局所改善してclusterを統合し、各levelのcommunityを段階表示する。
- **メリット**：高速で実装が広く、大規模グラフの粗視化baselineに向く。
- **デメリット**：非連結communityとresolution limitがあり、Leidenより品質上の弱点がある。
- **実装要点**：第一候補はLeidenとし、Louvainは速度・互換性baselineとして同一seedで比較する。

### 118. HDBSCAN点群クラスタ／noise除外 — 61点

- **評価**：61/100（カテゴリ順位38/40）
- **典拠**：[McInnes, Healy & Astels, hdbscan](https://joss.theoj.org/papers/10.21105/joss.00205)
- **概要**：記事embedding上の密度階層から安定clusterを選び、低密度点をnoiseとして表示抑制する。
- **メリット**：cluster数を事前指定せず、密度差と外れ記事を扱える。
- **デメリット**：低次元投影後に行うと投影artifactをcluster化し、辺構造を無視する。
- **実装要点**：元の高次元embeddingでclusterし、2D/3D座標は表示だけに使う。

### 119. Stochastic Block Model（SBM）縮約 — 60点

- **評価**：60/100（カテゴリ順位39/40）
- **典拠**：[Abbe, Community Detection and SBM](https://jmlr.org/papers/volume18/16-480/16-480.pdf)
- **概要**：頂点block間の辺生成確率を推定し、統計的に似た接続patternを持つ記事をまとめる。
- **メリット**：assortative community以外のhub・bridge・二部的役割もmodel化できる。
- **デメリット**：model選択と推論が重く、部分グラフsampling biasでblockが歪む。
- **実装要点**：degree-corrected・nested SBMを候補にし、posterior不確実性を表示へ反映する。

### 120. 多段quotient graph — 59点

- **評価**：59/100（カテゴリ順位40/40）
- **典拠**：[NetworkX quotient graph](https://networkx.org/documentation/stable/reference/algorithms/generated/networkx.algorithms.minors.quotient_graph.html)
- **概要**：任意のpartitionをsupernodeへ写し、level 0記事、level 1小cluster、level 2大clusterという多段粒度を作る。
- **メリット**：cluster手法と描画手法を分離し、同じUIでsemantic zoomを実装できる。
- **デメリット**：partition品質を改善する算法ではなく、集約属性の設計を誤ると情報を隠す。
- **実装要点**：superedgeへ辺数、総重み、方向、type分布を保持し、展開時に元IDへ完全に戻せるようにする。

---

## 4. 2次元レイアウト（40件）

### 121. ForceAtlas2 — 98点

- **評価**：98/100（カテゴリ順位1/40）
- **典拠**：[Jacomy et al., ForceAtlas2](https://doi.org/10.1371/journal.pone.0098679)、[NetworkX `forceatlas2_layout`](https://networkx.org/documentation/stable/reference/generated/networkx.drawing.layout.forceatlas2_layout.html)
- **概要**：引力・斥力、degree依存repulsion、gravity、Barnes–Hut近似を備えたネットワーク探索向け力学配置である。
- **メリット**：コミュニティを見つけやすく、Gephi等の成熟実装があり、Wikipedia関連記事の汎用初期候補に向く。
- **デメリット**：座標軸自体に意味はなく、parameter・初期値・局所最適で見た目が変わる。
- **実装要点**：LinLog mode、prevent overlap、scalingを検証し、seedと停止条件を固定する。

### 122. Yifan Hu／Graphviz `sfdp` — 97点

- **評価**：97/100（カテゴリ順位2/40）
- **典拠**：[Hu, Efficient and High Quality Force-Directed Drawing](https://yifanhu.net/PUB/graph_draw.pdf)、[Graphviz `sfdp`](https://graphviz.org/docs/layouts/sfdp/)
- **概要**：multilevel coarseningとBarnes–Hut近似を組み合わせ、大規模疎グラフを粗から細へ配置する。
- **メリット**：数万規模へ伸ばしやすく、全体構造と速度の釣合いがよい。
- **デメリット**：小さな局所関係と有向性は見えにくく、parameter tuningが必要。
- **実装要点**：大規模時の第一候補とし、component packingとedge sparsificationを前処理する。

### 123. Fruchterman–Reingold — 96点

- **評価**：96/100（カテゴリ順位3/40）
- **典拠**：[Fruchterman & Reingold, 1991](https://doi.org/10.1002/spe.4380211102)、[NetworkX spring layout](https://networkx.org/documentation/stable/reference/generated/networkx.drawing.layout.spring_layout.html)
- **概要**：辺をspring引力、頂点対を電荷斥力として反復し、均衡位置を求める代表的force-directed法である。
- **メリット**：理解・実装・調整が容易で、中小規模の強いbaselineになる。
- **デメリット**：素朴実装は `O(n²)` で、密集・局所最適・非決定性がある。
- **実装要点**：spectral初期値、固定seed、quadtree近似、反復上限を使う。

### 124. Stress Majorization／SMACOF — 95点

- **評価**：95/100（カテゴリ順位4/40）
- **典拠**：[Gansner, Koren & North, 2004](https://doi.org/10.1007/978-3-540-31843-9_25)、[OGDF StressMinimization](https://ogdf.github.io/doc/ogdf/classogdf_1_1_stress_minimization.html)
- **概要**：グラフ距離と描画上のEuclidean距離の差を重み付きstressとして反復最小化する。
- **メリット**：目的関数が明確で単調改善し、距離保存品質を数値比較できる。
- **デメリット**：全対最短距離と線形系が重く、大規模ではlandmark近似が要る。
- **実装要点**：開始記事・community代表をpivotにし、最終stressと近傍保存率を記録する。

### 125. Sugiyama／Graphviz `dot` — 94点

- **評価**：94/100（カテゴリ順位5/40）
- **典拠**：[Sugiyama, Tagawa & Toda, 1981](https://doi.org/10.1109/TSMC.1981.4308636)、[Graphviz dot](https://graphviz.org/docs/layouts/dot/)、[OGDF SugiyamaLayout](https://ogdf.github.io/doc/ogdf/classogdf_1_1_sugiyama_layout.html)
- **概要**：cycle除去、rank割当、交差削減、座標決定を行い、有向グラフを階層状に描く。
- **メリット**：開始点からの方向と深度、因果・包含に近い流れを読みやすくできる。
- **デメリット**：Wikipediaのcycleが多い一般グラフでは逆向き辺と横長化が増える。
- **実装要点**：BFS深度をrank constraintにし、feedback edgesを色分けする。

### 126. fCoSE — 93点

- **評価**：93/100（カテゴリ順位6/40）
- **典拠**：[Cytoscape.js layouts](https://js.cytoscape.org/)、[fCoSE implementation](https://github.com/iVis-at-Bilkent/cytoscape.js-fcose)
- **概要**：compound Spring Embedderを高速化し、cluster包含、相対位置、alignment制約を扱う。
- **メリット**：コミュニティをcompound nodeとして見せるWeb UIに強く、品質と速度が高い。
- **デメリット**：制約設定が複雑で、非compound graphにはForceAtlas2より利点が小さい。
- **実装要点**：Leiden communityをparent nodeへ写し、quality presetとrandomize設定を固定する。

### 127. Eclipse Layout Kernel（ELK Layered） — 92点

- **評価**：92/100（カテゴリ順位7/40）
- **典拠**：[The Eclipse Layout Kernel](https://arxiv.org/abs/2311.00533)
- **概要**：port、compound、方向、layer constraintを扱える産業的なlayered layout frameworkである。
- **メリット**：複雑な型付き有向グラフとUI box配置を細かく制御でき、elkjsも利用できる。
- **デメリット**：optionが多く、一般的な密グラフでは大きく広がりやすい。
- **実装要点**：主要関係だけをhierarchy edgeにし、他辺はlayout後のoverlayとしてroutingする。

### 128. OGDF FMMM — 91点

- **評価**：91/100（カテゴリ順位8/40）
- **典拠**：[OGDF layout example](https://ogdf.github.io/doc/ogdf/ex-layout.html)
- **概要**：fast multipole法とmultilevel法を組み合わせた大規模force-directed layoutである。
- **メリット**：高性能C++実装があり、非常に大きなグラフで高品質配置を狙える。
- **デメリット**：Webへの組込みがJavaScript系より難しく、WASM化やserver計算が必要。
- **実装要点**：offline/server座標計算に使い、クライアントは固定座標と増分微調整だけ行う。

### 129. OpenOrd — 90点

- **評価**：90/100（カテゴリ順位9/40）
- **典拠**：[OpenOrd implementation](https://github.com/SciTechStrategies/OpenOrd)
- **概要**：複数phaseの温度・cutting戦略を使い、大規模グラフのcluster構造を高速に分離する。
- **メリット**：数十万規模の概観とcommunity発見に向き、Gephi plugin実績がある。
- **デメリット**：局所形状と辺長は粗く、細部を正確に読む最終配置には不向き。
- **実装要点**：全体overview用とし、選択communityは別layoutで再配置する。

### 130. LinLog energy model — 89点

- **評価**：89/100（カテゴリ順位10/40）
- **典拠**：[Noack, Energy Models for Graph Clustering](https://jgaa.info/index.php/jgaa/article/download/paper154/2814/2621)
- **概要**：線形引力と対数斥力などにより、辺cutの少ないcommunityを空間的に分離するenergy modelである。
- **メリット**：クラスタ構造を通常FRより強調し、ForceAtlas2のLinLog modeとも対応する。
- **デメリット**：cluster間距離と座標軸を意味距離として解釈できず、leafが遠く飛びやすい。
- **実装要点**：modularity communityと一致度を見つつ、node degreeに応じたrepulsionを検証する。

### 131. Spectral layout — 88点

- **評価**：88/100（カテゴリ順位11/40）
- **典拠**：[NetworkX spectral layout](https://networkx.org/documentation/stable/reference/drawing.html)
- **概要**：Laplacianの第2・第3固有ベクトルをx,y座標にし、隣接頂点の座標差を小さくする。
- **メリット**：force法の良い決定的初期値になり、clusterの大域分離を高速に得られる。
- **デメリット**：固有値重複、非連結成分、hubで潰れ、node overlapを直接避けない。
- **実装要点**：成分ごとに計算・packingし、Procrustesで更新間の向きを揃える。

### 132. Kamada–Kawai — 87点

- **評価**：87/100（カテゴリ順位12/40）
- **典拠**：[Kamada & Kawai, 1989](https://doi.org/10.1016/0020-0190%2889%2990102-6)
- **概要**：全対グラフ距離を理想spring長にし、総energyをNewton型更新で減らす。
- **メリット**：小中規模で対称性とグラフ距離をきれいに表しやすい。
- **デメリット**：全対距離と反復が重く、大規模・高次数では重なりや局所解が増える。
- **実装要点**：数百頂点程度の詳細viewか、coarsened graphに限定する。

### 133. D3 force simulation — 86点

- **評価**：86/100（カテゴリ順位13/40）
- **典拠**：[D3 force](https://d3js.org/d3-force)
- **概要**：link、many-body、center、collision、position forceを組み合わせ、ブラウザで増分的に座標を更新する。
- **メリット**：対話・drag・固定点・逐次ノード追加へ柔軟で、UI実装が成熟している。
- **デメリット**：CPU上の大規模simulationは重く、既定parameterだけでは品質が安定しない。
- **実装要点**：Web Worker化、alpha再加熱の制限、quadtree、`forceCollide`を使う。

### 134. CoSE-Bilkent — 85点

- **評価**：85/100（カテゴリ順位14/40）
- **典拠**：[Cytoscape.js CoSE layouts](https://js.cytoscape.org/)
- **概要**：compound graphをspring embedderで配置し、入れ子clusterと通常nodeを同時に扱う。
- **メリット**：記事community、カテゴリ、記事を入れ子表示する構造に適する。
- **デメリット**：compound境界で大きく空白ができ、fCoSEより遅い場合がある。
- **実装要点**：fCoSEのbaselineとして同じcompound入力・停止条件で比較する。

### 135. WebCola制約付きforce layout — 84点

- **評価**：84/100（カテゴリ順位15/40）
- **典拠**：[WebCola](https://ialab.it.monash.edu/webcola/)
- **概要**：force／stress系配置にalignment、separation、flow、non-overlap等の制約を加える。
- **メリット**：開始記事固定、言語列、型別領域など設計意図を保ちながら自動配置できる。
- **デメリット**：制約が衝突すると不自然な間隔や収束遅延を招く。
- **実装要点**：hard constraintを最小化し、違反量をdebug表示できるようにする。

### 136. Graphviz `neato` — 83点

- **評価**：83/100（カテゴリ順位16/40）
- **典拠**：[Graphviz layout engines](https://graphviz.org/docs/layouts/)
- **概要**：Kamada–Kawai系またはstress系のspring modelで、一般無向グラフを配置する成熟CLI実装である。
- **メリット**：導入が容易で再現可能なSVG等を生成でき、prototypeと静的報告に強い。
- **デメリット**：大規模対話UIと逐次更新には向かず、固定図になりやすい。
- **実装要点**：`-Gstart`、overlap、splines、既知pos入力を固定し、server側renderに使う。

### 137. Graphviz `fdp` — 82点

- **評価**：82/100（カテゴリ順位17/40）
- **典拠**：[Graphviz layout engines](https://graphviz.org/docs/layouts/)
- **概要**：一般無向グラフをforce-directed placementで配置するGraphviz engineである。
- **メリット**：静的生成pipelineへ組み込みやすく、cluster属性等のGraphviz機能と連携できる。
- **デメリット**：規模が増すと`sfdp`に劣り、ブラウザ内の継続simulationには使えない。
- **実装要点**：小中規模の比較候補とし、同入力をneato・sfdpで自動評価する。

### 138. DrL（Distributed Recursive Layout） — 81点

- **評価**：81/100（カテゴリ順位18/40）
- **典拠**：[igraph layout manual](https://igraph.org/c/html/main/igraph-Layout.html)、[igraph DrL reference](https://r.igraph.org/reference/layout.drl.html)
- **概要**：大規模グラフ用に複数phaseで温度と引力を変えるforce-directed実装である。
- **メリット**：igraphから利用でき、千〜大規模のoverviewを高速に作りやすい。
- **デメリット**：parameterが多く、結果が粗く非決定的で、方向性を表さない。
- **実装要点**：default、coarsen後、edge-weightedの三条件を比較し、detail viewには別layoutを使う。

### 139. GRIP — 80点

- **評価**：80/100（カテゴリ順位19/40）
- **典拠**：[Gajer & Kobourov, GRIP](https://jgaa.info/index.php/jgaa/article/view/paper52)
- **概要**：recursive coarseningとintelligent placementで複数次元のforce配置を高速化する。
- **メリット**：近線形の時間・空間を目指し、大規模グラフと2D/3Dの共通基盤になる。
- **デメリット**：現代的なWeb実装が少なく、既存libraryへ直接載せにくい。
- **実装要点**：独自実装より、手法比較とspectral／multilevel初期化の設計参考にする。

### 140. GEM — 79点

- **評価**：79/100（カテゴリ順位20/40）
- **典拠**：[igraph GEM layout](https://igraph.org/c/html/main/igraph-Layout.html)、[igraph `layout_with_gem`](https://r.igraph.org/reference/layout_with_gem.html)
- **概要**：局所温度、振動・回転検出、gravityを使うadaptive force-directed algorithmである。
- **メリット**：単純FRより収束を制御し、一般無向グラフに美しい配置を得られる場合がある。
- **デメリット**：初期順序とparameterへ敏感で、大規模では新しいmultilevel法に劣る。
- **実装要点**：数百〜千頂点の比較候補にし、igraphの固定seedで測る。

### 141. Davidson–Harel simulated annealing — 78点

- **評価**：78/100（カテゴリ順位21/40）
- **典拠**：[igraph Davidson-Harel layout](https://igraph.org/c/html/main/igraph-Layout.html)、[igraph `layout_with_dh`](https://r.igraph.org/reference/layout_with_dh.html)
- **概要**：辺長、交差、node-edge距離など複数の美的costをsimulated annealingで同時最適化する。
- **メリット**：単純forceでは直接扱わない交差やnode-edge proximityを目的に入れられる。
- **デメリット**：計算が重く、cost係数が多く、規模が増すと実用性が下がる。
- **実装要点**：小さな代表subgraphの高品質静的図に限定し、時間上限を設ける。

### 142. LGL（Large Graph Layout） — 77点

- **評価**：77/100（カテゴリ順位22/40）
- **典拠**：[igraph LGL layout](https://igraph.org/c/html/main/igraph-Layout.html)、[igraph `layout_with_lgl`](https://r.igraph.org/reference/layout_with_lgl.html)
- **概要**：rootから層状に頂点を追加し、局所forceで大規模graphを段階配置する。
- **メリット**：非常に大きな疎graphへ適用でき、中心から周辺への概観を作れる。
- **デメリット**：選択rootに強く依存し、非連結・密graphで品質が粗い。
- **実装要点**：開始記事をrootにし、component別処理と強辺backboneを併用する。

### 143. Pivot MDS — 76点

- **評価**：76/100（カテゴリ順位23/40）
- **典拠**：[Brandes & Pich, Eigensolver methods for graph drawing](https://doi.org/10.1007/978-3-540-24595-7_24)、[OGDF PivotMDS](https://ogdf.github.io/doc/ogdf/)
- **概要**：少数pivotから全頂点への距離だけを使い、古典MDSを近似して座標を得る。
- **メリット**：全対距離を避けながらglobal distanceをある程度保存し、stress法の初期値にもなる。
- **デメリット**：pivot選択とgraph diameterに敏感で、局所近傍は潰れる場合がある。
- **実装要点**：開始点、遠点、community代表をfarthest-point samplingでpivotにする。

### 144. 放射状／Graphviz `twopi` — 75点

- **評価**：75/100（カテゴリ順位24/40）
- **典拠**：[Graphviz twopi](https://graphviz.org/docs/layouts/twopi/)
- **概要**：開始記事を中心に、graph距離またはBFS深度ごとの同心円へ記事を配置する。
- **メリット**：開始点からのhopを即座に読み取れ、説明・教育用途に強い。
- **デメリット**：同一ring内のcrossingとoverlapが増え、非tree辺を表しにくい。
- **実装要点**：ring内順序をcommunityまたはbarycentric heuristicで最適化する。

### 145. Circular／Graphviz `circo` — 74点

- **評価**：74/100（カテゴリ順位25/40）
- **典拠**：[Graphviz circo and layout engines](https://graphviz.org/docs/layouts/)
- **概要**：頂点を円周または複数円へ置き、順序最適化でcrossingを減らす。
- **メリット**：全頂点を均等に見せ、cycle・相互リンク・community間接続を比較しやすい。
- **デメリット**：頂点数が増えると辺が中心に密集し、距離に意味がない。
- **実装要点**：community単位のcircleとedge bundlingを使い、順序規則を明記する。

### 146. BFS layered layout — 73点

- **評価**：73/100（カテゴリ順位26/40）
- **典拠**：[NetworkX drawing layouts](https://networkx.org/documentation/stable/reference/drawing.html)
- **概要**：開始記事からのBFS depthを一軸、同層内順序を他軸として配置する。
- **メリット**：探索結果と図の階層が一致し、オンライン取得の進行も可視化しやすい。
- **デメリット**：同深度内の意味関係を表さず、多数のcross edgeで混雑する。
- **実装要点**：層内をcommunity→barycenterの順に並べ、戻り辺を曲線で分離する。

### 147. Reingold–Tilford tidy tree — 72点

- **評価**：72/100（カテゴリ順位27/40）
- **典拠**：[Reingold & Tilford, 1981](https://doi.org/10.1109/TSE.1981.234519)
- **概要**：BFS treeまたは代表親treeを、部分木の幅を保ちながら対称でcompactに配置する。
- **メリット**：探索親子と経路を非常に読みやすく、決定的配置を作れる。
- **デメリット**：各頂点に親を一つ選ぶ必要があり、非tree辺の意味を弱める。
- **実装要点**：代表親の選択scoreを保存し、非tree辺はon-demand overlayにする。

### 148. Balloon tree layout — 71点

- **評価**：71/100（カテゴリ順位28/40）
- **典拠**：[igraph tree and radial layouts](https://igraph.org/c/html/main/igraph-Layout.html)
- **概要**：各部分木の子を親の周囲へ円形に配置し、入れ子のballoonとしてtreeを描く。
- **メリット**：幅広い階層を矩形treeよりcompactに収め、subtree境界を見やすくする。
- **デメリット**：深さと角度の比較が難しく、非tree辺がballoonを横切る。
- **実装要点**：subtree sizeで半径を決め、選択時のみcross edgeを表示する。

### 149. Shell／Concentric layout — 70点

- **評価**：70/100（カテゴリ順位29/40）
- **典拠**：[NetworkX shell layout](https://networkx.org/documentation/stable/reference/drawing.html)、[Cytoscape.js concentric layout](https://js.cytoscape.org/)
- **概要**：PPR、core number、記事type等の離散rankで同心shellを作り、各shellへ記事を並べる。
- **メリット**：選んだ尺度を位置に直接符号化でき、force layoutより再現性が高い。
- **デメリット**：shell内距離に意味がなく、記事数の偏りで外周が過密になる。
- **実装要点**：ringの意味をlegendへ明示し、shell内順序をcommunityで安定化する。

### 150. Bipartite layout — 69点

- **評価**：69/100（カテゴリ順位30/40）
- **典拠**：[NetworkX bipartite layout](https://networkx.org/documentation/stable/reference/drawing.html)
- **概要**：記事–カテゴリ、記事–Wikidata entityなど二種頂点を左右二列へ分けて配置する。
- **メリット**：異種関係と共有先が明確で、記事–記事への巨大clique射影を避けられる。
- **デメリット**：一般の多型・多hop関係には列が増え、crossingが急増する。
- **実装要点**：次数順またはbarycentric orderingで各列を並べ、巨大categoryを折り畳む。

### 151. Arc diagram — 68点

- **評価**：68/100（カテゴリ順位31/40）
- **典拠**：[Wattenberg, Arc Diagrams](https://doi.org/10.1109/INFVIS.2002.1173155)
- **概要**：記事を一本の軸上に並べ、リンクを軸の上下のarcとして描く。
- **メリット**：順序、連続区間、long-range edgeを読みやすく、screen spaceを節約する。
- **デメリット**：良い並び順が不可欠で、一般graphの経路探索はnode-link平面より難しい。
- **実装要点**：community→PPRまたはseriationで順序付けし、辺typeを上下へ分ける。

### 152. Hive plot — 67点

- **評価**：67/100（カテゴリ順位32/40）
- **典拠**：[Krzywinski et al., Hive Plots](https://doi.org/10.1093/bib/bbr069)
- **概要**：頂点をtypeやmetricで複数axisへ割り当て、値に応じた位置とaxis間の辺で構造を描く。
- **メリット**：forceの偶然な座標を避け、再現可能な規則で型付き関係を比較できる。
- **デメリット**：局所近傍探索が直感的でなく、axis割当規則の設計が必要。
- **実装要点**：記事typeをaxis、PPRまたはdegreeを半径にし、node-link viewと切替える。

### 153. 並べ替え付きadjacency matrix — 66点

- **評価**：66/100（カテゴリ順位33/40）
- **典拠**：[Henry & Fekete, MatrixExplorer](https://doi.org/10.1109/TVCG.2006.160)
- **概要**：行列の行・列をcommunity、spectral seriation、hierarchical clusteringで並べ、辺をcellへ表示する。
- **メリット**：密graphでもedge overlapがなく、community blockと方向を正確に比較できる。
- **デメリット**：経路と個別nodeの隣接追跡が難しく、疎graphでは空白が多い。
- **実装要点**：node-link図とlinked selectionし、順序とaggregation levelをUIで切替える。

### 154. Orthogonal layout — 65点

- **評価**：65/100（カテゴリ順位34/40）
- **典拠**：[OGDF orthogonal layout modules](https://ogdf.github.io/doc/ogdf/group__gd-orthogonal.html)
- **概要**：辺を水平・垂直segmentでroutingし、bendとcrossingを減らすdiagram型配置である。
- **メリット**：型付き有向edgeとlabelを整理しやすく、小さな説明graphに適する。
- **デメリット**：一般密graphでは面積とbendが爆発し、自然なcommunity形状を失う。
- **実装要点**：表示をSteiner backbone等へ絞り、portとedge labelを制約に含める。

### 155. Tutte barycentric planar layout — 64点

- **評価**：64/100（カテゴリ順位35/40）
- **典拠**：[Tutte, “How to Draw a Graph”](https://doi.org/10.1112/plms/s3-13.1.743)
- **概要**：外周を凸多角形へ固定し、内部頂点を近傍の重心に置いてplanar graphを交差なし直線描画する。
- **メリット**：3-connected planar graphでは強い理論保証があり、決定的で高速。
- **デメリット**：Wikipedia graphは非planarで、planar backbone抽出が必要。
- **実装要点**：planar subgraphまたはtree-like backboneの説明図に限定する。

### 156. Graphviz `osage` cluster packing — 63点

- **評価**：63/100（カテゴリ順位36/40）
- **典拠**：[Graphviz osage](https://www.graphviz.org/docs/layouts/osage/)
- **概要**：clusterを矩形として再帰的にpackingし、その内部nodeとcluster間edgeをroutingする。
- **メリット**：Leiden等のcommunity境界を安定した領域として明確に見せられる。
- **デメリット**：packing時にedgeを重視せず、cluster間関係の幾何が不自然になり得る。
- **実装要点**：community比較viewに使い、通常のforce viewを併設する。

### 157. Graphviz `patchwork` treemap — 62点

- **評価**：62/100（カテゴリ順位37/40）
- **典拠**：[Graphviz layout engines](https://graphviz.org/docs/layouts/)
- **概要**：cluster階層とnode weightをsquarified treemapの矩形面積へ写す。
- **メリット**：community規模と階層を面積としてcompactに比較できる。
- **デメリット**：辺を主役にするnetwork layoutではなく、記事間経路を読みづらい。
- **実装要点**：overviewの集約viewとして使い、矩形選択でnode-link詳細へ遷移する。

### 158. Connected-component packing — 61点

- **評価**：61/100（カテゴリ順位38/40）
- **典拠**：[Graphviz graph attributes: pack](https://www.graphviz.org/docs/graph/)
- **概要**：各連結成分を独立layoutした後、bounding boxを重ならないよう2Dへpackingする。
- **メリット**：弱く接続または分断された候補群を無理にforceで混ぜず、空間利用を改善する。
- **デメリット**：成分間の相対位置に意味がなく、更新時に大きく再配置されやすい。
- **実装要点**：成分をPPR・sizeで安定順序付けし、位置の意味がないことを明示する。

### 159. 2D UMAP layout — 60点

- **評価**：60/100（カテゴリ順位39/40）
- **典拠**：[UMAP](https://arxiv.org/abs/1802.03426)
- **概要**：記事embeddingを2次元へ落とし、近傍点をnode位置として元graph edgeをoverlayする。
- **メリット**：semantic point cloudとlink graphを同じ画面で比較でき、大量nodeにも比較的速い。
- **デメリット**：投影距離とgraph距離が異なり、edgeが長く交差しやすい。
- **実装要点**：semantic viewとtopology layoutを切替え、trustworthinessとedge-length分布を表示する。

### 160. 2D t-SNE layout — 59点

- **評価**：59/100（カテゴリ順位40/40）
- **典拠**：[van der Maaten & Hinton, t-SNE](https://jmlr.csail.mit.edu/beta/papers/v9/vandermaaten08a.html)
- **概要**：高次元記事embeddingの局所確率を2D Student-t分布で再現し、clusterを分離する。
- **メリット**：局所的な意味clusterを発見しやすく、点群explorationに実績がある。
- **デメリット**：cluster間距離・大きさを解釈できず、逐次追加と再現性が弱く、graph辺を使わない。
- **実装要点**：PCA初期化、複数perplexity、固定seedを比較し、主network layoutにはしない。

---

## 5. 3次元レイアウト（40件）

### 161. 3D Fruchterman–Reingold — 98点

- **評価**：98/100（カテゴリ順位1/40）
- **典拠**：[NetworkX `spring_layout(dim=3)`](https://networkx.org/documentation/stable/reference/generated/networkx.drawing.layout.spring_layout.html)、[igraph 3D layouts](https://igraph.org/c/html/main/igraph-Layout.html)
- **概要**：FRの引力・斥力を3次元vectorで計算し、記事を空間内の点、リンクを線分として配置する。
- **メリット**：2D実装を自然に拡張でき、自由度増加でnode overlapと真のedge crossingを減らせる。
- **デメリット**：画面投影では遮蔽と見かけの交差が戻り、良いviewpointなしでは読みにくい。
- **実装要点**：固定seed、octree近似、orbit操作、focus、2D同期viewを必須にする。

### 162. Walshaw multilevel 3D force layout — 97点

- **評価**：97/100（カテゴリ順位2/40）
- **典拠**：[Walshaw, Multilevel Force-Directed Graph-Drawing](https://jgaa.info/index.php/jgaa/article/view/paper70)
- **概要**：隣接頂点を反復coarsenし、最粗graphを3D配置してからlevelごとに展開・force refinementする。
- **メリット**：論文で2D・3Dと大規模graphを明示的に扱い、global qualityと速度を両立する。
- **デメリット**：coarseningで型付き・方向付き関係が失われやすく、実装が単純FRより複雑。
- **実装要点**：community-aware matchingと辺type集約を使い、各levelの座標をsemantic zoomに再利用する。

### 163. GRIP multidimensional force layout — 96点

- **評価**：96/100（カテゴリ順位3/40）
- **典拠**：[Gajer & Kobourov, GRIP](https://jgaa.info/index.php/jgaa/article/view/paper52)
- **概要**：recursive coarseningとintelligent vertex placementを使い、3Dを含む多次元でforce energyを高速最小化する。
- **メリット**：大規模graphに近線形で、random初期値より最終位置に近い配置から始められる。
- **デメリット**：古いC/OpenGL系実装で、現在のWeb stackへ統合する作業が必要。
- **実装要点**：アルゴリズム設計をserver/GPU実装へ移植し、2Dと同一coarseningを共有する。

### 164. 3D Stress Majorization — 95点

- **評価**：95/100（カテゴリ順位4/40）
- **典拠**：[Graph drawing by stress majorization](https://doi.org/10.1007/978-3-540-31843-9_25)、[3D viewpoint evaluation](https://doi.org/10.1111/cgf.15077)
- **概要**：graph距離と3D Euclidean距離の差をstressとして反復最小化する。
- **メリット**：目的が明確で、3D化によるstress低下を2Dと定量比較できる。
- **デメリット**：全対距離が重く、3Dで保存された距離を2D screenから正確に読めない。
- **実装要点**：Pivot MDS初期化、landmark stress、投影後stressを併記する。

### 165. 3D UMAP — 94点

- **評価**：94/100（カテゴリ順位5/40）
- **典拠**：[UMAP](https://arxiv.org/abs/1802.03426)、[2D/3D network cartographs](https://www.nature.com/articles/s43588-022-00199-z)
- **概要**：記事embeddingのfuzzy近傍graphを `n_components=3` で三次元点群へ射影する。
- **メリット**：意味embeddingを大量記事で高速表示でき、2Dよりcluster重なりを減らせる場合がある。
- **デメリット**：原Wikipedia edgeを直接最適化せず、回転で印象が変わり、軸に意味がない。
- **実装要点**：2D/3Dで同じ近傍parameterを使い、trustworthinessとviewpoint別overlapを比較する。

### 166. 3D Spectral layout — 93点

- **評価**：93/100（カテゴリ順位6/40）
- **典拠**：[igraph 3D layouts](https://igraph.org/c/html/main/igraph-Layout.html)、[NetworkX spectral layout](https://networkx.org/documentation/stable/reference/drawing.html)
- **概要**：Laplacianの第2〜第4固有vectorをx,y,zへ割り当て、低周波のgraph構造を三軸へ写す。
- **メリット**：決定的でforceの初期配置になり、clusterの大域分離を得やすい。
- **デメリット**：固有値重複で軸が回転・交換し、非連結成分とhubで座標が潰れる。
- **実装要点**：版間で符号・軸をProcrustes整合し、componentごとにsphere packingする。

### 167. Spherical MDS — 92点

- **評価**：92/100（カテゴリ順位7/40）
- **典拠**：[Miller, Huroyan & Kobourov, Spherical Graph Drawing by MDS](https://arxiv.org/abs/2209.00191)
- **概要**：記事を球面上に拘束し、球面geodesic距離とgraph距離のstressをSGDで最小化する。
- **メリット**：境界のないoverviewと回転navigationを提供し、一部graphでEuclideanより低歪みになる。
- **デメリット**：裏面遮蔽が常にあり、中心・周辺の意味をEuclidean viewと同じに扱えない。
- **実装要点**：focus点を手前へ回転し、裏面edge透過、great-circle routing、2D展開図を用意する。

### 168. H3 3D hyperbolic layout — 91点

- **評価**：91/100（カテゴリ順位8/40）
- **典拠**：[Munzner, H3](https://graphics.stanford.edu/papers/h3/html.nosplit/)
- **概要**：spanning treeを3D hyperbolic spaceのcone treeとして配置し、focus周辺を射影で拡大する。
- **メリット**：体積が指数的に増える空間を利用し、大きな階層とfocus+context表示に強い。
- **デメリット**：tree外辺はlayoutへ寄与せず、双曲射影・navigationの学習負担がある。
- **実装要点**：PPRまたはWikidata階層でspanning treeを選び、cross edgeは選択時だけ描く。

### 169. Poincaré ball 3D embedding — 90点

- **評価**：90/100（カテゴリ順位9/40）
- **典拠**：[Nickel & Kiela, Poincaré Embeddings](https://proceedings.neurips.cc/paper_files/paper/2017/hash/59dfa2df42d9e3d41f5b02bfc32229dd-Abstract.html)
- **概要**：階層記事を3次元Poincaré ballへ学習し、中心を一般概念、境界を多数の具体概念として配置する。
- **メリット**：tree-likeなカテゴリ／subclass構造を低次元で表現し、階層depthを半径へ反映できる。
- **デメリット**：一般本文リンクのcycleと多義関係には合わず、境界付近が画面上で密になる。
- **実装要点**：階層辺と通常辺を分離し、Möbius変換でfocusを中心へ移動できるようにする。

### 170. PCA 3D point cloud — 89点

- **評価**：89/100（カテゴリ順位10/40）
- **典拠**：[scikit-learn PCA](https://scikit-learn.org/stable/modules/generated/sklearn.decomposition.PCA.html)
- **概要**：SBERT、TADW等の高次元記事vectorを分散最大の三主成分へ線形射影する。
- **メリット**：高速・決定的で、新規記事を同じ三軸へtransformでき、説明分散比を示せる。
- **デメリット**：非線形近傍やgraph distanceを保存せず、最大分散が主題上の重要軸とは限らない。
- **実装要点**：標準化規則、各軸の寄与語・特徴、累積説明分散を表示する。

### 171. 3D t-SNE — 88点

- **評価**：88/100（カテゴリ順位11/40）
- **典拠**：[van der Maaten & Hinton, t-SNE](https://jmlr.csail.mit.edu/beta/papers/v9/vandermaaten08a.html)、[3D viewpoint evaluation](https://doi.org/10.1111/cgf.15077)
- **概要**：高次元記事近傍確率を3D Student-t分布上で再現し、局所clusterを空間分離する。
- **メリット**：2Dより自由度が増え、密な意味clusterの重なりを減らせる場合がある。
- **デメリット**：cluster間距離を解釈できず、out-of-sampleと再現性が弱い。
- **実装要点**：PCA初期化、複数perplexity・seed、2D版との近傍保持差を測る。

### 172. 3D Isomap — 87点

- **評価**：87/100（カテゴリ順位12/40）
- **典拠**：[Tenenbaum, de Silva & Langford, Isomap](https://doi.org/10.1126/science.290.5500.2319)
- **概要**：記事vectorのkNN graph上geodesic距離を古典MDSで3Dへ埋め込む。
- **メリット**：曲がった大域manifoldを三次元で展開し、経路距離との対応を持たせられる。
- **デメリット**：近傍graphのshort circuit・非連結性・全対距離costに弱い。
- **実装要点**：landmark版とk sweepを用い、元Wikipedia graphとembedding kNN graphを区別する。

### 173. 3D Locally Linear Embedding — 86点

- **評価**：86/100（カテゴリ順位13/40）
- **典拠**：[Roweis & Saul, LLE](https://doi.org/10.1126/science.290.5500.2323)
- **概要**：各記事の局所再構成重みを保つよう、三次元座標を固有値問題から求める。
- **メリット**：局所manifoldの曲がりを保ち、線形PCAより複雑な形を表せる。
- **デメリット**：noise、密度差、近傍数へ敏感で、大域cluster位置を信頼できない。
- **実装要点**：regularizationと近傍数を検証し、孤立点は別のinitializationへ逃がす。

### 174. 3D PHATE — 85点

- **評価**：85/100（カテゴリ順位14/40）
- **典拠**：[PHATE original paper](https://www.nature.com/articles/s41587-019-0336-3)、[PHATE documentation](https://phate.readthedocs.io/)
- **概要**：記事vector近傍の拡散potential距離を作り、metric MDSで3Dへ配置する。
- **メリット**：branch、continuum、clusterの同時表現を狙え、三次元で分岐を追いやすい。
- **デメリット**：Wikipedia graphでの標準benchmarkが少なく、拡散parameterの意味づけが必要。
- **実装要点**：semantic vector入力とgraph diffusion入力を分け、UMAPとDEMaP／trustworthinessを比較する。

### 175. 3D TriMap — 84点

- **評価**：84/100（カテゴリ順位15/40）
- **典拠**：[TriMap](https://arxiv.org/abs/1910.00204)
- **概要**：記事三つ組の相対近さを三次元で満たし、global orderingを保つ点群を作る。
- **メリット**：cluster間の相対構造を2Dより余裕ある空間へ置き、大域関係を維持しやすい。
- **デメリット**：triplet samplingに依存し、原リンク方向・edge crossingを直接最適化しない。
- **実装要点**：同じtripletとseedで2D/3Dを計算し、距離順位相関を比較する。

### 176. 3D PaCMAP — 83点

- **評価**：83/100（カテゴリ順位16/40）
- **典拠**：[PaCMAPを含むJMLR比較](https://www.jmlr.org/papers/v22/20-1061.html)
- **概要**：near、mid-near、far pairを段階的に最適化し、記事を三次元へ配置する。
- **メリット**：局所clusterと大域配置の均衡を取り、3Dの追加自由度を利用できる。
- **デメリット**：pair scheduleと初期値で変化し、追加軸が必ず解釈性を上げるわけではない。
- **実装要点**：2Dとのtask studyを行い、最良viewpointだけで評価しない。

### 177. 3D LargeVis — 82点

- **評価**：82/100（カテゴリ順位17/40）
- **典拠**：[Tang et al., LargeVis](https://arxiv.org/abs/1602.00370)
- **概要**：近似kNN graphとnegative samplingにより、大量記事を3D低次元空間へ配置する。
- **メリット**：linear-timeを目指した設計で、大規模点群を実用時間に収めやすい。
- **デメリット**：非決定的で実装保守性に注意が要り、元リンク構造を直接は保たない。
- **実装要点**：UMAP 3Dと同じ入力・規模でtime、memory、trustworthinessをbenchmarkする。

### 178. 3D Minimum-Distortion Embedding — 81点

- **評価**：81/100（カテゴリ順位18/40）
- **典拠**：[Minimum-Distortion Embedding](https://arxiv.org/abs/2103.02559)
- **概要**：相互リンク等をattractive pair、hard negativeをrepulsive pairとして、標準化された3D座標を最適化する。
- **メリット**：何を近づけ何を離すかを明示でき、Wikipedia固有の複合関係を直接目的化できる。
- **デメリット**：pair selectionとdistortion設計が結果を作り込み、計算・調整負担がある。
- **実装要点**：各辺typeのloss寄与をlogし、2D同一目的との改善量を測る。

### 179. 3D Laplacian Eigenmaps — 80点

- **評価**：80/100（カテゴリ順位19/40）
- **典拠**：[Belkin & Niyogi, Laplacian Eigenmaps](https://newtraell.cs.uchicago.edu/research/publications/techreports/TR-2002-01)
- **概要**：重み付き近傍graphのLaplacian第2〜第4固有vectorを三次元点にする。
- **メリット**：局所近接を保つ目的が明瞭で、spectral graph layoutとpoint-cloud embeddingを統一できる。
- **デメリット**：大規模固有値計算、density差、非連結性とout-of-sampleが課題。
- **実装要点**：Nyström近似とnormalized Laplacianを検証し、符号・軸を版間整合する。

### 180. 3D Variational Graph Autoencoder — 79点

- **評価**：79/100（カテゴリ順位20/40）
- **典拠**：[Kipf & Welling, VGAE](https://arxiv.org/abs/1611.07308)
- **概要**：VGAEのlatent dimensionを3にして、リンク再構成lossから直接三次元記事座標を学習する。
- **メリット**：別の射影段階がなく、座標でlink predictionを直接説明でき、不確実性も得られる。
- **デメリット**：3次元bottleneckで再構成性能が落ち、通常のVGAE前提は無向静的graphである。
- **実装要点**：高次元VGAE→UMAP 3Dと直接3D VGAEをAUC・近傍保持・可読性で比較する。

### 181. 3D Kamada–Kawai — 78点

- **評価**：78/100（カテゴリ順位21/40）
- **典拠**：[igraph layout API (`dim=3`)](https://python.igraph.org/en/main/api/igraph.layout.html)、[Kamada & Kawai](https://doi.org/10.1016/0020-0190%2889%2990102-6)
- **概要**：graph全対距離を理想長とするspring energyを三次元で最小化する。
- **メリット**：小中規模でgraph距離と対称性を保ち、2Dより低energyを得られる場合がある。
- **デメリット**：全対距離と反復が重く、遮蔽・局所解・projection依存は残る。
- **実装要点**：community縮約graphまたは数百記事のdetail viewに限定し、2D KKとstressを比較する。

### 182. GEM-3D — 77点

- **評価**：77/100（カテゴリ順位22/40）
- **典拠**：[Bruss & Frick, Fast Interactive 3-D Graph Visualization](https://doi.org/10.1007/BFb0021794)
- **概要**：GEMのadaptive local temperature、gravity、oscillation・rotation検出を三次元へ拡張したspring embedderである。
- **メリット**：interactiveな3D graph配置を明示的に設計し、単純FRより局所適応的に収束する。
- **デメリット**：古い手法で現行libraryが少なく、大規模ではmultilevel／GPU法が有利。
- **実装要点**：比較実装とし、temperature制御の考え方を現在の3D force engineへ移植する。

### 183. 高次元force計算後の3D projection — 76点

- **評価**：76/100（カテゴリ順位23/40）
- **典拠**：[A multi-dimensional approach to force-directed layouts](https://doi.org/10.1016/j.comgeo.2004.03.014)
- **概要**：3次元より高い空間でforce layoutを最適化し、PCA等で最終的に3Dへ射影する。
- **メリット**：低次元の局所最適から逃れ、より滑らかで対称な配置を得られる場合がある。
- **デメリット**：高次元energyが投影時に再び重なり、計算量と説明困難性が増す。
- **実装要点**：直接3D forceとのstress、crossing、近傍保持を同一seed群で比較する。

### 184. Cone Tree — 75点

- **評価**：75/100（カテゴリ順位24/40）
- **典拠**：[Robertson, Mackinlay & Card, Cone Trees](https://doi.org/10.1145/108844.108883)
- **概要**：各親の子を円錐底面の円周へ並べ、階層を連続する三次元coneとして配置する。
- **メリット**：幅広いtreeをcompactに見せ、選択nodeを正面へ回すanimationで経路を追える。
- **デメリット**：spanning tree選択が必要で、Wikipediaの多重親・cross edgeはclutterになる。
- **実装要点**：代表親treeだけを常時表示し、他辺はhover時に描く。

### 185. 同心球BFS／3D radial layers — 74点

- **評価**：74/100（カテゴリ順位25/40）
- **典拠**：[Kwon et al., Spherical Layout and Rendering](https://doi.org/10.1109/PACIFICVIS.2015.7156357)
- **概要**：開始記事を中心に置き、BFS深度ごとの球殻へ記事を分散配置する。
- **メリット**：開始点からのhopを半径として直接読め、2D同心円より各層の面積を広く使える。
- **デメリット**：内部球殻が外側に遮られ、同じshell内の順序とedge routingが難しい。
- **実装要点**：shellを半透明にし、communityごとのsolid angle割当と切断面viewを提供する。

### 186. Spherical force-directed layout — 73点

- **評価**：73/100（カテゴリ順位26/40）
- **典拠**：[Kobourov & Wampler, Non-Euclidean Spring Embedders](https://doi.org/10.1109/TVCG.2005.103)
- **概要**：球面の接空間でforceを計算し、exponential map等で記事を球面上に更新する。
- **メリット**：境界のない均等なoverviewと任意方向focusを実現し、一般graphへ適用できる。
- **デメリット**：geodesic計算が複雑で、antipodal点と裏面遮蔽を扱う必要がある。
- **実装要点**：単位球へ正規化するだけの擬似手法と区別し、great-circle距離で評価する。

### 187. Hyperbolic Riemannian force layout — 72点

- **評価**：72/100（カテゴリ順位27/40）
- **典拠**：[Kobourov & Wampler, Non-Euclidean Spring Embedders](https://doi.org/10.1109/TVCG.2005.103)
- **概要**：双曲空間の距離・角度・接空間へforce-directed法を一般化し、任意graphを負曲率空間へ置く。
- **メリット**：階層・scale-free graphの指数的成長を収め、focus+context探索に適する。
- **デメリット**：curvature、model、projectionが視覚結果を大きく変え、利用者の理解負担が高い。
- **実装要点**：Poincaré ball表示とhyperboloid計算を分け、数値境界clampを入れる。

### 188. GeoGraphViz地理制約3D force — 71点

- **評価**：71/100（カテゴリ順位28/40）
- **典拠**：[Wang, Li & Gu, GeoGraphViz](https://arxiv.org/abs/2304.09864)
- **概要**：3D force graphへ地図上の位置への引力を加え、意味networkとgeolocationの釣合いを調整する。
- **メリット**：場所・出来事記事で、地理的分布と関係構造を一画面に統合できる。
- **デメリット**：非地理記事と同地点記事の配置が難しく、地図とgraphの距離尺度が競合する。
- **実装要点**：地理force係数をUIで調整し、座標なし記事はsemantic layerへ分離する。

### 189. 関係型別2.5D積層 — 70点

- **評価**：70/100（カテゴリ順位29/40）
- **典拠**：[Kerracher et al., Temporal Graph Visualisation design space](https://doi.org/10.2312/eurovisshort.20141149)
- **概要**：本文リンク、カテゴリ、Wikidata、クリック等を平行planeに分け、同一記事を垂直edgeで結ぶ。
- **メリット**：異質な関係を混ぜず、どのlayerがclusterや経路を作るか比較できる。
- **デメリット**：layer数とcross-layer edgeで遮蔽が増え、Z距離を関連度と誤認しやすい。
- **実装要点**：Z軸は離散layerと明示し、solo、small multiples、透過、layer間隔調整を備える。

### 190. Space–time cube — 69点

- **評価**：69/100（カテゴリ順位30/40）
- **典拠**：[TimeLighting / 2D+t temporal networks](https://doi.org/10.1109/TVCG.2024.3514858)
- **概要**：各時点の2D graph layoutをZ時間軸へ積み、記事の座標変化をtrajectoryとして描く。
- **メリット**：Wikipedia revisionや月次clickstreamによるcluster・中心性の変化を連続的に追える。
- **デメリット**：空間と時間のclutterが大きく、Z距離は意味距離でない。
- **実装要点**：shared-node位置の安定化、time slicing、trajectory選択、animation viewを併設する。

### 191. 言語版別Z-layer layout — 68点

- **評価**：68/100（カテゴリ順位31/40）
- **典拠**：[Wikidata sitelinks and data access](https://www.wikidata.org/wiki/Wikidata:Data_access)、[multilayer graph visualization design](https://doi.org/10.2312/eurovisshort.20141149)
- **概要**：日本語、英語など言語版ごとに2D planeを作り、同じQIDの記事をZ方向に接続する。
- **メリット**：言語別リンク構造、記事欠落、中心性差を直接比較できる。
- **デメリット**：言語数が増えると読みづらく、同じQIDでも記事範囲が一致しない。
- **実装要点**：共通QIDをanchorにshared layoutを計算し、対象言語を少数選択する。

### 192. Community sphere packing — 67点

- **評価**：67/100（カテゴリ順位32/40）
- **典拠**：[Spherical layout and rendering methods](https://vis.cs.ucdavis.edu/papers/ImmersiveGraphVis.pdf)
- **概要**：各communityをsphereまたはballへ局所配置し、community間graphを外側の3D packingで配置する。
- **メリット**：cluster境界を明示し、局所detailと全体networkを分離して計算できる。
- **デメリット**：ball境界が強すぎて連続的関係を隠し、sphere同士のpackingに意味がない場合がある。
- **実装要点**：sphere半径をnode数の立方根で決め、inter-community edgeで中心配置を微調整する。

### 193. Torus／任意曲面拘束layout — 66点

- **評価**：66/100（カテゴリ順位33/40）
- **典拠**：[State of the Art of Graph Visualization in non-Euclidean Spaces](https://diglib.cgv.tugraz.at/items/13b67e12-e536-486a-a29e-78bb77692a2c)
- **概要**：forceまたはbarycentric更新をtorus、cylinder、複数surface上へ拘束し、境界や周期性を変える。
- **メリット**：cycleやperiodic clusterに合う位相を選べ、平面境界artifactを減らせる。
- **デメリット**：Wikipedia関係に対応する曲面選択根拠が弱く、navigationと距離解釈が難しい。
- **実装要点**：分析仮説がある場合だけ使い、Euclidean・球面とのdistortion比較を必須にする。

### 194. Anchored incremental 3D layout — 65点

- **評価**：65/100（カテゴリ順位34/40）
- **典拠**：[Dynamic graph visualisation design space](https://doi.org/10.2312/eurovisshort.20141149)
- **概要**：既存記事座標を固定または弱いspringでanchorし、オンライン追加記事と局所近傍だけ3D再配置する。
- **メリット**：逐次取得時のmental mapを保ち、毎回全空間が回転・反転するのを防ぐ。
- **デメリット**：初期配置の悪さを固定し、追加が続くと局所歪みと密集が蓄積する。
- **実装要点**：局所更新と定期global relayoutを分け、Procrustes整合後にanimationする。

### 195. Interactive 3D force-directed edge bundling — 64点

- **評価**：64/100（カテゴリ順位35/40）
- **典拠**：[Zielasko et al., Interactive 3D Force-Directed Edge Bundling](https://www.graphics.rwth-aachen.de/publication/02107/)
- **概要**：近い向き・clusterの3D edgeを相互引力で束ね、空間内の線clutterを減らす。
- **メリット**：大量edgeのmacro flowを見せ、cluster間接続patternを把握しやすい。
- **デメリット**：個々の辺と方向を追いにくく、存在しない共有経路のように見える。
- **実装要点**：selection時は原辺へ戻し、bundle strength、方向particle、edge countを表示する。

### 196. Barnes–Hut octree 3D force近似 — 63点

- **評価**：63/100（カテゴリ順位36/40）
- **典拠**：[Barnes & Hut, hierarchical N-body algorithm](https://doi.org/10.1038/324446a0)、[3d-force-graph](https://github.com/vasturiano/3d-force-graph)
- **概要**：遠方node群をoctree cellの代表質量にまとめ、3D斥力計算を `O(n²)` から概ね `O(n log n)` へ近似する。
- **メリット**：数千〜数万記事のinteractive 3D forceを現実的にする中核的高速化である。
- **デメリット**：近似角度で配置品質が変わり、密度の偏ったgraphではtree更新costが大きい。
- **実装要点**：theta、更新頻度、worker/GPU利用を計測し、近傍forceは正確に保つ。

### 197. Viewpoint-optimized 3D graph drawing — 62点

- **評価**：62/100（カテゴリ順位37/40）
- **典拠**：[Wageningen et al., Viewpoint-Based 3D Graph Drawing](https://doi.org/10.1111/cgf.15077)
- **概要**：3D座標を固定した後、球面上の多数camera方向を評価し、投影crossing、overlap、angular resolution等が良い視点を選ぶ。
- **メリット**：悪い投影だけで3D手法を評価する誤りを防ぎ、静止画にも良いviewを与えられる。
- **デメリット**：layout自体を改善せず、metric間の最良視点が一致せず、sampling costがある。
- **実装要点**：Fibonacci sphereでviewpointをsampleし、複数metricのPareto候補をUIへ提示する。

### 198. `3d-force-graph`実装 — 61点

- **評価**：61/100（カテゴリ順位38/40）
- **典拠**：[3d-force-graph](https://github.com/vasturiano/3d-force-graph)
- **概要**：Three.js/WebGLとd3-force-3dまたはngraph physicsを統合した対話的3D graph componentである。
- **メリット**：orbit、picking、labels、particles、VR/AR派生を短期間で実装でき、prototypeに最適。
- **デメリット**：layout品質より描画componentであり、大規模label・accessibility・bundleは追加実装が要る。
- **実装要点**：第一prototypeに採用し、座標計算をworkerへ分離、固定座標の再読込を可能にする。

### 199. Three.js `Points`＋`BufferGeometry` — 60点

- **評価**：60/100（カテゴリ順位39/40）
- **典拠**：[Three.js Points](https://threejs.org/docs/pages/Points.html)、[BufferGeometry](https://threejs.org/docs/pages/BufferGeometry.html)
- **概要**：事前計算した3D座標をGPU bufferのpoint cloudとして描き、必要なedgeだけline bufferで重ねる。
- **メリット**：大量点を低overheadで描け、shader、LOD、color、pickingを自由に最適化できる。
- **デメリット**：layout計算、label、interaction、edge管理を自作する必要がある。
- **実装要点**：位置・色・sizeをtyped arrayへ格納し、dirty range更新、frustum culling、ID pickingを実装する。

### 200. deck.gl `PointCloudLayer`／Plotly `scatter3d` — 59点

- **評価**：59/100（カテゴリ順位40/40）
- **典拠**：[deck.gl layer catalog](https://deck.gl/docs/api-reference/layers)、[Plotly 3D scatter](https://plotly.com/javascript/3d-scatter-plots/)
- **概要**：計算済み3D記事座標を、deck.glの大規模point layerまたはPlotlyの対話scatterとして表示する実装選択肢である。
- **メリット**：前者は大量dataとpicking、後者は少量dataの迅速な分析・共有に強い。
- **デメリット**：いずれもgraph layout算法ではなく、複雑なedge routingとsemantic zoomは別実装になる。
- **実装要点**：production大規模viewはdeck.gl、研究notebook・比較図はPlotlyと使い分ける。

---

## 推奨する初期実装

最初の実装は、次の順が費用対効果に優れる。

1. `prop=links`＋`generator=links`で逐次取得し、被リンクを上限付きで補う。
2. pageid・redirect・revisionを正規化し、取得状態とcontinue tokenをcheckpointする。
3. 相互リンク、本文位置、WLM、SBERT、PPRを個別特徴として保存し、初期は重み付き和、教師が集まったらLambdaMARTへ移す。
4. PPR上位＋community quota＋代表経路で表示規模を絞り、Leidenの多段quotientを作る。
5. 2DはForceAtlas2を基準、巨大graphは`sfdp`、方向説明はSugiyamaを切替える。
6. 3Dは3D FRと3D UMAPを基準に、3D Stress、Spherical MDS、H3を目的別に比較する。
7. 3D表示には自動viewpoint候補、2D同期view、focus、on-demand edge、透過、LODを必須にする。

擬似コードの最小形は次のとおりである。

```text
enqueue(startPage, priority=1)
while queue not empty and budgets remain:
    page := pop highest priority
    response := fetch links + pageprops + revision using continuation checkpoints
    normalize redirects and pageids
    for each candidate link:
        features := reciprocal, sectionPosition, WLM, textSimilarity,
                    categoryIDF, clickProbability, depth, novelty
        priority := calibrated relevance(features) - depthPenalty
        accept by per-parent cap, beam width, and diversity quota
        upsert typed edge with source revision and evidence
        enqueue unseen accepted page

G := typed directed multigraph
rank := personalizedPageRank(aggregate(G), seed=startPage)
H := retain top rank + community representatives + connecting paths
communities := Leiden(H, multiple resolutions)
layout2d := ForceAtlas2(H, fixedSeed)
layout3d := compare(3D-FR(H), UMAP3D(articleEmbeddings), Stress3D(H))
render progressively with synchronized 2D/3D selection
```

## 検証項目

- **取得**：継続tokenの完走、重複pageid、redirect、namespace、revision固定、再開時の冪等性、API etiquette。
- **関連度**：開始記事ごとの人手gold setに対するprecision@k、recall@k、nDCG、MRR。特徴ablationと時間・言語別holdout。
- **点群**：trustworthiness、continuity、kNN overlap、distance correlation、cluster安定性。座標軸に意味を与えない。
- **2D**：stress、edge crossing、node/label overlap、angular resolution、計算時間、更新時移動量。
- **3D**：3D内のstressだけでなく、100〜1000視点の投影crossing・overlap・遮蔽、視点探索時間、path／cluster発見taskの正答率を測る。
- **再現性**：Wikipedia revision、取得日時、model/library version、全parameter、乱数seed、棄却理由を保存する。

## 結論

200方式の比較から、中心となる設計は「オンライン逐次取得」「型付き関係の保存」「開始点personalization」「多段粒度」「2Dを基準view、3Dを探索viewとして同期」の組合せである。特に3Dは自由度が増えるだけで自動的に良くならず、viewpoint、遮蔽、interaction、2D対照表示までを一つの手法として評価しなければならない。

最初から一方式へ固定せず、同じ取得済みgraphと記事embeddingに対して上位手法を差替え可能にすることが重要である。座標・cluster・順位は事実ではなく、選択した入力関係と目的関数の結果であるため、画面上で根拠、parameter、版を追跡できる設計を採る。
