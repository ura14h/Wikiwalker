# Wikiwalker 設計仕様書 (index_claude.html)

作成日：2026年9月19日
対象：`index_claude.html`（単一 HTML・バニラ JavaScript）
上位文書：[アプリのコンセプト](cencept.md)、[ストーリーボード](storyboard.md)、[Wikipedia API 仕様書](refs/wikipedia-api.md)、[関連記事の可視化手法](refs/wikipedia-graph.md)

本書は `index_claude.html` が「何を」「どのように」実現しているかを、コードを読む人の手引きとして記す。設定値は本文中の表に集約し、コード上は `CONFIG` オブジェクトに対応する。

---

## 1. 概要

Wikiwalker は、Wikipedia の記事を「星」、記事間のリンクを「関係線」として 3D 空間に描き、マウスで散策できるグラフィカルブラウザーである。

| 項目 | 内容 |
| --- | --- |
| 形態 | 単一 HTML ファイル。サーバー不要、`file://` で開いて動作する |
| 実装 | バニラ JavaScript（ES modules）、責務ごとのクラス分割 |
| 外部ライブラリ | `3d-force-graph`、`three-spritetext`、`three`（いずれも esm.sh から読み込み） |
| データ源 | 各言語版 Wikipedia の MediaWiki Action API、Meta-Wiki の Site matrix |
| テーマ | 宇宙・SF・辞典・シンプル（濃紺の星空、明朝体の見出し、半透明パネル） |

### 1.1 用語

| 用語 | 意味 |
| --- | --- |
| 記事星（article ノード） | Wikipedia の 1 記事に対応するノード（球体） |
| 星団（cluster ノード） | 同じカテゴリを共有する複数の関連記事を 1 つに集約したノード（環付き球体） |
| その他（pool ノード） | どの星団にも入らなかった関連記事をまとめたノード（灰色の環付き球体） |
| 直接ノード | 選択記事の導入部に現れるリンク先。集約せず記事星として個別に表示する |
| 選択記事 | 現在中央に据え、詳細を表示している記事 |
| 収集（run） | 選択記事の関連記事をバックグラウンドで取得し、グラフへ反映する一連の処理 |

---

## 2. 画面仕様

画面は 3D グラフを全面に敷き、その上に固定配置の UI を重ねる。ストーリーボードの場面番号と対応付けて示す。

```
┌──────────────────────────────────────────────────────────────┐
│ [JA] [検索キーワード入力欄            ]        [Prev] [Next] │
│  WIKIWALKER                                                  │
│                                                              │
│                       ○ 記事星                               │
│                  ◎ 星団   ● 選択記事 ── ○                   │
│                       ○         ◎                            │
│                                         ┌──────────────────┐ │
│                                         │ 選択記事詳細パネル │ │
│                                         │ (スクロール可)     │ │
│ [通知]                                  └──────────────────┘ │
│ テキストは Wikipedia「記事名」の記事を CC BY-SA 4.0 の下で…   │
└──────────────────────────────────────────────────────────────┘
```

### 2.1 構成要素

| 要素 | 位置 | 仕様 | 場面 |
| --- | --- | --- | --- |
| 言語選択ボタン | 左上 | 現在の言語コードを大文字で表示（例 `JA`）。クリックで言語選択ダイアログを開く | #1 |
| 検索キーワード入力欄 | 上部中央寄り | インクリメンタル検索。入力 250ms 後に検索し、候補をすぐ下にポップアップ表示。`/` キーでフォーカス | #1, #2 |
| 検索該当記事リスト | 入力欄の直下 | タイトルと抜粋（検索語を強調、2 行で省略）。↑↓ で選択移動、Enter で決定、Esc で閉じる。マウスクリックでも決定 | #2 |
| Prev / Next ボタン | 右上 | 選択履歴スタックを戻る／進む。端に達したボタンは無効化 | #4 |
| 3D グラフ | 全面 | ドラッグで回転、Shift+ドラッグで平行移動、ホイールでズーム。ノードクリックで選択 | #3〜#5 |
| 選択記事詳細パネル | 右下固定 | 記事見出し（Wikipedia へのリンク）、主なカテゴリ、導入部の抜粋。本文はパネル内スクロール。下段に収集の進捗 | #3, #5 |
| 通知 | 下部中央 | エラーや待機の一時的な通知（4 秒で消える） | — |
| 権利表示 | 左下 | 常時表示。表示中の記事へのリンクと CC BY-SA 4.0 へのリンクを含む | 全場面 |
| 言語選択ダイアログ | モーダル | 言語名・コードで絞り込める一覧。現在の言語を強調 | #1 |

### 2.2 場面遷移

| 遷移 | 契機 | 動作 |
| --- | --- | --- |
| #1 → #2 | キーワード入力 | 候補リストを表示。入力が変わるたびに前回の検索を中断して再検索 |
| #2 → #3 | 候補を決定 | 宇宙（グラフ・履歴）を初期化し、記事を原点に置いて選択。詳細を表示し、収集を開始 |
| #3 → #4 | 収集の進行 | 直接ノード → 星団ノードの順に、取得でき次第グラフへ逐次追加 |
| #4 → #5 | 記事星をクリック | グラフを保ったまま選択記事を変更。カメラが移動し、詳細が切り替わり、その記事の収集を開始。前の収集は中断 |
| #5 → #3/#4 | — | 以降は #3・#4 と同じ振る舞い |
| 星団をクリック | — | 詳細パネルにメンバー一覧を表示し、先頭 24 件を記事星として星団の周囲へ展開（一覧の項目クリックで選択可） |
| Prev / Next | — | 履歴上の記事を再選択（履歴には積まない）。中断されていた収集があれば再開 |
| 言語切替 | ダイアログで決定 | 選択記事があれば `langlinks` で同じ記事の他言語版を探し、あればその記事から再出発。なければ初期画面へ戻り通知 |

### 2.3 グラフの見た目

| 種類 | 形状 | 色 | ラベル |
| --- | --- | --- | --- |
| 選択記事 | 大きい球 + 半透明の光輪 | 白 | 大きめ |
| 記事星 | 球（展開済みはやや大きい） | 深さ（選択起点からのホップ数）ごとの色 | 記事名 |
| 星団 | 球 + 傾いた環 | 金色 | `カテゴリ名 (件数)` |
| その他 | 球 + 環 | 灰色 | `その他 (件数)` |

- 選択記事は位置を固定し、レイアウトの揺れの影響を受けない。
- 選択記事から遠いノードほど霞む（フォグ：距離 80 で 0.75、600 以上で 0.18 の不透明度）。
- 新規ノードは親ノードの位置から 90ms 間隔で 1 つずつ出現する。
- 背景に 2,500 個の点群で星空を描く。

---

## 3. 記事データの取得仕様

### 3.1 使用する API

すべて GET・`format=json`・`formatversion=2`・`origin=*` を付与し、`Api-User-Agent` ヘッダーで識別する。

| 用途 | 呼び出し | 主なパラメーター | 優先度 |
| --- | --- | --- | --- |
| 記事検索 | `list=search` | `srnamespace=0`, `srlimit=10`, `srprop=snippet` | 2（最優先） |
| 記事詳細 | `prop=info\|extracts\|pageprops\|categories` | `inprop=url`, `exintro=1`, `ppprop=disambiguation`, `clshow=!hidden`, `cllimit=50`, `redirects=1` | 1 |
| 導入部リンク | `action=parse` | `section=0`, `prop=text`, `redirects=1` | 1 |
| タイトル解決 | `titles=…`（最大 50 件） | `prop=info\|pageprops\|categories`, `redirects=1` | 1 |
| 全出リンク | `generator=links` | `gplnamespace=0`, `gpllimit=50`, `prop=categories\|pageprops`, `clshow=!hidden`, `cllimit=max`, `redirects=1` | 0（背景） |
| 他言語版 | `prop=langlinks` | `lllang=<言語コード>`, `lllimit=1` | 1 |
| 言語一覧 | `action=sitematrix`（meta.wikimedia.org） | `smtype=language`, `smlimit=max` | 0 |

- 応答の `continue` はそのまま次のリクエストへ渡す。`generator=links` では `batchcomplete` が返るまで同じページの属性（カテゴリ）を束ねてから 1 バッチとして扱う。
- 応答ページの `missing` / `invalid` / `disambiguation` はそれぞれ除外条件として扱う。
- 言語版のホスト名は Site matrix の `site[].url` を採用する。起動直後の暫定表（主要 18 言語）も同じ値を保持しており、初回のダイアログ表示時に一覧を Site matrix で置き換える。

### 3.2 利用制限の遵守（`RequestQueue`）

| 規則 | 実装 |
| --- | --- |
| 同時実行 1 件 | 全リクエストを 1 本の直列キューで処理する |
| 5 件/秒未満・200 件/分 | リクエスト開始間隔を最短 320ms にする（約 187 件/分） |
| 1 秒超の応答の後は 5 秒待機 | 背景タスク（優先度 0）に限り 5 秒の休止を入れる。検索・記事表示は間隔 320ms のまま割り込める |
| HTTP 429 / 503 | `Retry-After`（なければ 5 秒）だけキュー全体を止め、最大 3 回まで再試行 |
| `error.code = maxlag` | 5 秒待って再試行 |
| キャッシュ | パラメーターをソートした URL をキーに応答 Promise をキャッシュ。失敗した要求はキャッシュから外す |
| 中断 | 記事を切り替えると前の収集の `AbortController` を中断し、未開始の要求はキューから捨てる。同じ URL を共有していた別の呼び出し元は自身の signal で出し直す |

待機はキュー内で 100ms 刻みに行い、待機中に優先度の高いタスクが来たときはそれを先に通す。これにより、収集中でも検索や記事表示が止まらない。

### 3.3 コンテンツの再利用条件

- 表示する抜粋は `prop=extracts` の HTML を、`script`/`style`/`img`/`iframe` と `on*`・`href`・`src`・`style` 属性を落としてから描画する。
- 画面左下に、表示中の記事（`fullurl`）と CC BY-SA 4.0 へのリンクを常時掲示する。記事未選択時は Wikipedia トップへのリンクになる。

---

## 4. 関連記事の意味変換（記事データ → 記事星）

高次数の記事をそのまま描くと画面が飽和するため、関連記事を次の 3 種に振り分ける。処理は決定論的で、同じ記事に対しては同じ結果になる。

### 4.1 手順

1. **導入部リンク → 直接ノード**
   `action=parse&section=0` の HTML から、本文中の出現順で内部リンクを抽出する（[可視化手法](refs/wikipedia-graph.md) 043「本文位置による重み」、020「First-Link Network」の簡略版）。
   除外：`table`（情報ボックス）、`figure`、`sup`/`.reference`（脚注）、`cite`/`ol.references`/`.mw-references-wrap`（自動付与される出典一覧）、`.hatnote`、`.navbox`、`.noprint`、赤リンク、ファイルリンク。
   先頭 32 件をタイトル解決（`redirects=1`）し、名前空間 0・非欠落・非曖昧さ回避のものを先頭から最大 16 件、直接ノードとして追加する。
2. **全出リンク → 星団**
   `generator=links` で 50 件ずつ最大 8 リクエスト（最大 400 リンク）取得し、バッチごとに `RelatedAggregator` へ渡す。直接ノード・選択記事自身・曖昧さ回避ページは除外する。
3. **星団・その他の逐次反映**
   バッチごとに、影響のあった星団ノードを追加または件数更新し、「その他」ノードの件数を更新する。

### 4.2 集約アルゴリズム（`RelatedAggregator`）

[可視化手法](refs/wikipedia-graph.md) 008「カテゴリ二部グラフ」・106/120「コミュニティ縮約・quotient graph」を、オンライン逐次取得向けに単純化したものである。

```
addCandidates(pages):
  fresh ← 未処理の候補をタイトル順に整列
  for page in fresh:
    home ← page のカテゴリを持つ既存星団のうちメンバー数最大のもの（同数ならカテゴリ名順）
    home があれば home に追加、なければ pool に追加
  while 星団数 < maxClusters:
    category ← pool 内で共有メンバー数が最大のカテゴリ（minClusterSize 未満なら終了、同数なら名前順）
    category を持つ pool の記事を新しい星団へ移す
```

| 設定 | 値 | 意味 |
| --- | --- | --- |
| `minClusterSize` | 3 | 星団を作るのに必要な共有記事数 |
| `maxClusters` | 14 | 1 記事あたりの星団数上限。超過分は「その他」に残る |
| `ignoredCategories` | `存命人物`, `Living people`, `…ページ`, `Pages …`, `Articles …` | 汎用すぎるカテゴリ、非表示指定のない保守用カテゴリ |
| `expandLimit` | 24 | 星団を開いたときにグラフへ展開する記事数 |

性質：
- 既存の星団は縮まない（バッチが増えても表示がちらつかない）。
- 同じ順序で同じ候補が届けば同じ星団構成になる（API の列挙順は決定的）。
- 星団はカテゴリ名をそのままラベルにするため、集約の根拠が画面上で説明できる。

---

## 5. ソフトウェア構成

単一ファイル内を「設定 → 共通ユーティリティ → 通信 → 意味変換 → 表示 → 統括」の順に並べ、各責務を 1 クラスにする。

```
Wikiwalker（統括）
 ├─ WikiApi ──── RequestQueue        通信
 ├─ RelatedAggregator                意味変換
 ├─ GraphView（3d-force-graph）      表示：宇宙
 ├─ SearchBox                        表示：検索
 ├─ LanguageDialog                   表示：言語選択
 ├─ DetailPanel                      表示：記事詳細・進捗
 ├─ HistoryNav                       表示：Prev / Next
 └─ StatusBar                        表示：通知
```

### 5.1 クラスの責務

| クラス | 責務 | 主なメソッド |
| --- | --- | --- |
| `RequestQueue` | 直列・優先度付きのリクエスト実行、間隔制御、休止 | `enqueue(run, {priority, signal})`, `holdFor(ms)` |
| `WikiApi` | Action API の呼び出し、応答の正規化、キャッシュ | `search`, `page`, `leadLinkTitles`, `resolveTitles`, `linkedPages`（async generator）, `langlinkTitle`, `loadSites` |
| `RelatedAggregator` | 候補記事を星団／その他へ決定論的に振り分ける | `exclude(pageid)`, `addCandidates(pages)` |
| `GraphView` | ノード・リンクの保持、3D 描画、逐次出現、選択・フォーカス、フォグ | `add`, `addNow`, `link`, `rename`, `select`, `focus`, `reset` |
| `SearchBox` | 入力のデバウンス、検索の中断、候補描画、キーボード操作 | `schedule`, `run`, `choose`, `setValue` |
| `LanguageDialog` | 言語一覧の表示・絞り込み・決定、ボタン表示 | `open`, `setSites`, `setCurrent` |
| `DetailPanel` | 記事・星団の詳細描画、進捗表示、権利表示の更新 | `showArticle`, `showCluster`, `setProgress`, `clear` |
| `HistoryNav` | 選択履歴スタックとボタン状態 | `push`, `move`, `reset` |
| `StatusBar` | 一時的な通知 | `notice` |
| `Wikiwalker` | 場面遷移、選択、収集 run の管理、言語切替 | `startFrom`, `selectArticle`, `collectRelated`, `openCluster`, `changeLanguage` |

### 5.2 データモデル

**記事レコード（`WikiApi.normalizePage` の出力）**

| フィールド | 型 | 内容 |
| --- | --- | --- |
| `pageid` | number | ページ ID（言語版内でのみ一意） |
| `title` | string | 正規化・リダイレクト解決後のタイトル |
| `ns` | number | 名前空間 |
| `url` | string \| null | `fullurl` |
| `extract` | string \| null | 導入部の HTML 抜粋（詳細取得時のみ） |
| `categories` | string[] | 非表示以外の所属カテゴリ（`Category:` 接頭辞を除去） |
| `disambiguation` | boolean | 曖昧さ回避ページか |
| `missing` | boolean | 存在しないページか |

**グラフノード**

| フィールド | article | cluster | pool |
| --- | --- | --- | --- |
| `id` | `a:<pageid>` | `c:<親pageid>:<カテゴリ>` | `p:<親pageid>` |
| `kind` | `'article'` | `'cluster'` | `'pool'` |
| `name` | 記事名 | `カテゴリ (件数)` | `その他 (件数)` |
| `page` / `members` | `page` | `category`, `members[]` | `members[]` |
| `depth` | 選択起点からのホップ数（色に使用） | 同左 | 同左 |
| `expanded` | 収集済みか | 展開済みか | 展開済みか |
| `summary`, `run` | 収集結果の要約、収集 run の ID | — | — |

ノード ID にページ ID を使うため、同じ記事が複数の経路で現れても 1 つのノードに統合され、関係線だけが追加される。

### 5.3 選択と収集の流れ（`selectArticle`）

```
selectArticle(ref):
  前の収集を中断（AbortController.abort, runId++）
  ref に pageid があれば、応答を待たずにノードを置き、選択・カメラ移動・パネル・履歴を更新
  page ← api.page(ref)                      # 優先度 1
  runId が変わっていれば終了（古い run）
  ノードの page を抜粋付きで置き換え、パネルを更新
  collectRelated(node, runId, signal)       # await しない（背景）
    ├─ 導入部リンク → 直接ノード（優先度 1）
    ├─ 出リンクをバッチ取得 → 星団・その他を逐次反映（優先度 0）
    └─ 中断・失敗時は node.expanded を戻し、次回選択時に再開
```

- `runId` は収集 run の識別子で、中断できなかった応答が後から届いても古い run の結果は捨てる。
- 収集中に別の記事を選ぶと前の run は中断される。Prev で戻ったときはキャッシュにより短時間で再開する。

---

## 6. 設定値一覧（`CONFIG`）

| キー | 値 | 意味 |
| --- | --- | --- |
| `userAgent` | `Wikiwalker/1.0 (single-file local browser app)` | `Api-User-Agent` の値 |
| `request.minGap` | 320 ms | リクエスト開始の最短間隔 |
| `request.slowThreshold` | 1000 ms | これを超える応答の後に背景タスクを休止 |
| `request.slowGap` | 5000 ms | 背景タスクの休止時間 |
| `search.debounce` | 250 ms | 入力から検索までの待ち |
| `search.limit` | 10 | 候補件数 |
| `related.directMax` | 16 | 直接ノードの上限 |
| `related.linkBatch` | 50 | `gpllimit` |
| `related.maxLinkRequests` | 8 | 出リンク列挙の上限リクエスト数 |
| `related.minClusterSize` | 3 | 星団の最小サイズ |
| `related.maxClusters` | 14 | 星団数の上限 |
| `related.expandLimit` | 24 | 星団展開時の記事数 |
| `related.ignoredCategories` | 正規表現の配列 | 集約に使わないカテゴリ |
| `graph.addInterval` | 90 ms | ノード出現間隔 |
| `graph.focusDistance` | 280 | カメラと選択ノードの距離 |
| `graph.focusDuration` | 900 ms | カメラ移動時間 |
| `graph.chargeStrength` | −90 | ノード間の反発 |
| `graph.linkDistance` | 55 | 関係線の基準長 |
| `graph.depthColors` | 7 色 | 深さごとの記事星の色 |
| `graph.clusterColor` / `poolColor` | 金 / 灰 | 星団・その他の色 |
| `graph.fogNear` / `fogFar` | 80 / 600 | フォグの距離範囲 |
| `graph.baseOpacity` / `minOpacity` | 0.75 / 0.18 | フォグの不透明度範囲 |
| `graph.starCount` | 2500 | 背景の星の数 |
| `seedSites` | 18 言語 | Site matrix 読み込み前の暫定言語表 |

---

## 7. 非機能要件と対応

| 要件 | 対応 |
| --- | --- |
| 操作への応答が常に軽快 | 通信はすべて非同期。検索・記事表示は優先度で割り込み、収集の待機はいつでも中断できる。pageid が既知なら応答を待たずに画面を切り替える |
| 取得中の操作禁止区間をなくす | UI をロックしない。進捗は詳細パネル下段に表示するだけ |
| 画面の飽和防止 | 直接ノード最大 16 + 星団最大 14 + その他 1 に抑え、残りは星団展開または一覧で辿る |
| 決定論的な集約 | 4.2 節。乱数を使わず、順序はタイトル順・カテゴリ名順で固定 |
| 過度なエンジニアリングをしない | ビルド不要、フレームワーク不使用、クラスは 10 個。設定は `CONFIG` に集約 |
| 認知負荷の低減 | ファイル内をセクション見出しで区切り、各クラスの先頭に責務コメント、非自明な判断に理由コメントを付す |

---

## 8. 既知の制約

- 出リンクは API の列挙順（名前空間・タイトル順）で先頭 400 件までを対象にする。超過分は要約に「（先頭のみ）」と示す。
- 1 秒を超える応答の後は背景収集を 5 秒休止するため、大きな記事では星団が出そろうまで数十秒かかる。体感を優先する場合は `CONFIG.request.slowGap` を下げる。
- 星団のラベルはカテゴリ名そのままであり、意味的なまとめ直し（同義カテゴリの統合、階層の利用）は行わない。
- 導入部の抜粋のみを表示し、記事全文は Wikipedia へのリンクで参照する。
- 外部ライブラリはインターネットから読み込むため、オフラインでは動作しない。
- 言語切替で他言語版に同じ記事がない場合は初期画面に戻る（グラフは引き継がない）。
