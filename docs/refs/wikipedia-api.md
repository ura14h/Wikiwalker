# Wikipedia API 仕様書

作成日：2026年9月16日

対象サービス：各言語版WikipediaのMediaWiki Action API、および記事の取得・探索・更新検知に利用できるWikimedia公開API

## 1. 文書の目的と対象

本書は、Wikipediaが公開する記事データ取得サービスについて、記事検索、記事情報、記事間リンクおよびカテゴリの取得に関するインターフェースと利用制限を定義する。MediaWiki Action APIを中心に、版固定本文を扱うMediaWiki REST API、閲覧数を扱うWikimedia Analytics API、更新通知を扱うEventStreams、構造化データを扱うWikidata APIも、記事の取得・探索に必要な範囲で扱う。利用シナリオ上の留意事項は末尾の補足に記す。

APIは読み取り用途で利用でき、APIキーやログインを必須としない。ブラウザからの匿名リクエストにも対応しているため、記事検索、記事情報の取得、記事内の内部リンク取得は、APIの返すJSONを機械的に処理して実現できる。[API:Links](https://www.mediawiki.org/wiki/API:Links) [API:Cross-site requests](https://www.mediawiki.org/wiki/API:Cross-site_requests)

### 対応言語とホスト名の取得

対応する言語、Wikimediaプロジェクト、サイトURLの完全な一覧は、Meta-WikiのSite matrixから取得できる。人が確認する場合は`https://meta.wikimedia.org/wiki/Special:SiteMatrix`、プログラムから取得する場合は`https://meta.wikimedia.org/w/api.php`の`action=sitematrix`を使用する。[SiteMatrix API](https://www.mediawiki.org/wiki/API:Sitematrix)

```sh
curl --get --silent --show-error --compressed \
  --user-agent 'WikipediaApiExample/1.0 (https://example.org/contact)' \
  'https://meta.wikimedia.org/w/api.php' \
  --data-urlencode 'action=sitematrix' \
  --data-urlencode 'smtype=language' \
  --data-urlencode 'smlangprop=code|name|localname|site' \
  --data-urlencode 'smsiteprop=url|dbname|code|sitename' \
  --data-urlencode 'smlimit=max' \
  --data-urlencode 'format=json' \
  --data-urlencode 'formatversion=2'
```

応答例（HTTP 200、JSON構造）：

```json
{
  "limits": {
    "sitematrix": 5000
  },
  "sitematrix": {
    "count": 1072,
    "151": {
      "code": "ja",
      "name": "日本語",
      "site": [
        {
          "url": "https://ja.wikipedia.org",
          "dbname": "jawiki",
          "code": "wiki",
          "sitename": "Wikipedia"
        }
      ],
      "localname": "Japanese"
    }
  }
}
```

省略箇所：`$.sitematrix`：実応答から日本語の言語groupだけを掲載し、その他の言語groupを省略。`$.sitematrix["151"].site`：`code=wiki`のWikipediaサイトだけを掲載し、他のWikimediaプロジェクトを省略。`count`は言語数ではなく、Site matrixに含まれるwikiの総数である。数値形式のgroupキーは応答内の位置であり、言語識別子として保存しない。

各言語groupの`code`が言語コード、`site[]`がその言語に属するWikimediaプロジェクトである。Wikipediaだけを列挙する場合は`site[].code === "wiki"`を選び、`site[].url`をサイトの基準URLとして使用する。Action API URLはその末尾へ`/w/api.php`、MediaWiki REST API URLは`/w/rest.php/v1/`を付加して構成する。閉鎖済み等のサイトには`closed`、`private`、`fishbowl`等の状態キーが付き得るため、利用対象から除外するかを明示的に判断する。

#### 言語版ごとのエンドポイント例

Site matrixの`site[].url`から構成したAction APIおよびMediaWiki REST APIの基準エンドポイント例は次のとおりである。

| 言語版 | Action API | MediaWiki REST API |
| --- | --- | --- |
| 日本語版 | `https://ja.wikipedia.org/w/api.php` | `https://ja.wikipedia.org/w/rest.php/v1/` |
| 英語版 | `https://en.wikipedia.org/w/api.php` | `https://en.wikipedia.org/w/rest.php/v1/` |
| フランス語版 | `https://fr.wikipedia.org/w/api.php` | `https://fr.wikipedia.org/w/rest.php/v1/` |
| その他 | …（言語版を省略） | …（言語版を省略） |

例えば英語版の記事を検索する場合は`https://en.wikipedia.org/w/api.php`、フランス語版の記事を検索する場合は`https://fr.wikipedia.org/w/api.php`へ同じAction APIパラメーターを送る。ページID、名前空間の名称、記事タイトル、記事内容、検索結果は言語版ごとに異なるため、ある言語版の`pageid`を別のホストへ渡してはならない。

Analytics APIの記事別閲覧数では、URLの`project`に日本語版は`ja.wikipedia.org`、英語版は`en.wikipedia.org`、フランス語版は`fr.wikipedia.org`を指定する。EventStreamsの`https://stream.wikimedia.org/`とWikidataの`https://www.wikidata.org/`、`https://query.wikidata.org/`は言語版ごとにホストを置き換えるサービスではない。

#### Webブラウザの言語設定からホスト名を決定する

次の例は、Webブラウザの`navigator.languages`を優先順に調べ、Site matrixが返した稼働中のWikipediaサイトへ対応付ける。`ja-JP`から`ja`、`en-US`から`en`のように末尾のregion・script subtagを段階的に除去し、一致しない場合は日本語版へフォールバックする。言語コードを文字列連結でホスト名へ変換せず、Site matrixの`site[].url`だけを採用する。

```js
const SITE_MATRIX_ENDPOINT = "https://meta.wikimedia.org/w/api.php";

function languageCandidates(languageTag) {
  const normalized = languageTag.trim().toLowerCase().replaceAll("_", "-");
  if (!normalized) return [];

  const parts = normalized.split("-");
  const candidates = [];
  for (let length = parts.length; length > 0; length -= 1) {
    candidates.push(parts.slice(0, length).join("-"));
  }

  // 古いブラウザや保存済み設定で現れ得る旧言語コード。
  const aliases = { iw: "he", in: "id", ji: "yi" };
  const alias = aliases[parts[0]];
  if (alias) candidates.push(alias);

  return [...new Set(candidates)];
}

async function loadWikipediaSites() {
  const sitesByLanguage = new Map();
  let smcontinue;

  do {
    const params = new URLSearchParams({
      action: "sitematrix",
      smtype: "language",
      smlangprop: "code|name|localname|site",
      smsiteprop: "url|dbname|code|sitename",
      smlimit: "max",
      format: "json",
      formatversion: "2",
      origin: "*",
    });
    if (smcontinue) params.set("smcontinue", smcontinue);

    const response = await fetch(`${SITE_MATRIX_ENDPOINT}?${params}`);
    if (!response.ok) {
      throw new Error(`Site matrix request failed: HTTP ${response.status}`);
    }

    const payload = await response.json();
    if (payload.error) {
      throw new Error(`Site matrix error: ${payload.error.code}`);
    }

    for (const group of Object.values(payload.sitematrix ?? {})) {
      if (!group || typeof group !== "object" || !Array.isArray(group.site)) {
        continue;
      }

      const wikipedia = group.site.find((site) =>
        site.code === "wiki" &&
        !Object.hasOwn(site, "closed") &&
        !Object.hasOwn(site, "private") &&
        !Object.hasOwn(site, "fishbowl")
      );
      if (!wikipedia) continue;

      const baseUrl = new URL(wikipedia.url);
      const isWikipediaHost =
        baseUrl.protocol === "https:" &&
        (baseUrl.hostname === "wikipedia.org" ||
          baseUrl.hostname.endsWith(".wikipedia.org"));
      if (!isWikipediaHost) continue;

      sitesByLanguage.set(group.code.toLowerCase(), {
        languageCode: group.code,
        baseUrl: baseUrl.origin,
        dbname: wikipedia.dbname,
      });
    }

    smcontinue =
      payload["query-continue"]?.sitematrix?.smcontinue ??
      payload.continue?.smcontinue;
  } while (smcontinue);

  return sitesByLanguage;
}

async function resolveWikipediaEndpoints({
  languages = globalThis.navigator?.languages?.length
    ? [...globalThis.navigator.languages]
    : [globalThis.navigator?.language ?? "ja"],
  fallbackCode = "ja",
} = {}) {
  const sites = await loadWikipediaSites();

  const requestedCodes = languages.flatMap(languageCandidates);
  requestedCodes.push(...languageCandidates(fallbackCode));

  const matched = requestedCodes
    .map((code) => sites.get(code))
    .find(Boolean);
  if (!matched) {
    throw new Error("No usable Wikipedia host was found");
  }

  return {
    ...matched,
    actionApi: `${matched.baseUrl}/w/api.php`,
    restApi: `${matched.baseUrl}/w/rest.php/v1/`,
    analyticsProject: new URL(matched.baseUrl).hostname,
  };
}

resolveWikipediaEndpoints()
  .then((endpoints) => console.log(endpoints))
  .catch((error) => console.error(error));
```

ブラウザの優先言語が`ja-JP`の場合、`languageCode`は`ja`、`actionApi`は`https://ja.wikipedia.org/w/api.php`、`restApi`は`https://ja.wikipedia.org/w/rest.php/v1/`となる。`navigator.languages`は利用者の優先順を表すため、配列の先頭だけでなく、一致するサイトが見つかるまで順に評価する。

Site matrixの継続値は、実応答では`query-continue.sitematrix.smcontinue`に入る。`smlimit=max`でも継続値がある場合は、同じクエリへ`smcontinue`として渡して完走する。対応言語とサイト状態は変更され得るため、一覧をソースコードへ固定せず、Site matrixの取得結果と取得日時をキャッシュする。このcurlと応答例は実送信により、日本語版の`url`、`dbname`、`code`を確認したものである。上のJavaScriptに`ja-JP`、`en-US`の順で言語コードを与えた場合に、日本語版のホスト名と両API URLが得られることも実送信で確認した。エンドポイント例に示した英語版とフランス語版についても、Action APIとMediaWiki REST APIがHTTP 200を返すことを確認した。

## 2. サービスの構成

MediaWiki Action APIは、`action`パラメーターで呼び出す機能を指定するHTTP APIである。本書の中心は`action=query`と`action=parse`である。版固定HTML、閲覧数、変更通知、構造化された概念関係は別の公開サービスから取得するため、同じ認証・継続・応答形式であるとは仮定しない。

| 機能 | 呼び出し | 取得できるもの |
| --- | --- | --- |
| 記事検索 | `action=query&list=search` | 検索結果、記事タイトル、ページID、抜粋 |
| ページ情報 | `action=query&prop=info` | ページID、タイトル、正規URL、最終版IDなど |
| 抜粋 | `action=query&prop=extracts` | 記事冒頭または指定量のテキスト・限定HTML |
| 出リンク | `action=query&prop=links` | 指定ページからリンクされているページ |
| ページ属性 | `action=query&prop=pageprops` | 曖昧さ回避などのページ属性 |
| 解析済みページ | `action=parse` | 解析済みHTML、内部リンク、版IDなど |
| 被リンク | `action=query&list=backlinks` | 指定ページへリンクしているページ |
| 所属カテゴリ・上位カテゴリ | `action=query&prop=categories` | 記事またはカテゴリが直接所属するカテゴリ |
| カテゴリ内の記事・下位カテゴリ | `action=query&list=categorymembers` | 指定カテゴリに直接所属するページ |
| リンク先を起点にした一括属性取得 | `action=query&generator=links` | 出リンク先のページ情報、属性、カテゴリ、抜粋、最新版ID |
| 版情報・Wikitext | `action=query&prop=revisions` | 版ID、時刻、本文ソース、版履歴 |
| 言語間リンク | `action=query&prop=langlinks` | 他言語版のタイトル、言語コード、URL |
| テンプレート関係 | `prop=templates`、`list=embeddedin` | 記事が使用するテンプレート、テンプレートを使用するページ |
| 外部URL関係 | `prop=extlinks`、`list=exturlusage` | 記事からの外部URL、同じURLを含むページ |
| 地理近傍 | `prop=coordinates`、`list=geosearch` | 記事座標、指定地点の近傍記事と距離 |
| 最新版・特定版HTML | MediaWiki REST API | HTML、ページID、版ID、ライセンス情報 |
| 記事別閲覧数 | Wikimedia Analytics API | 日次・月次のpageview時系列 |
| 更新通知 | Wikimedia EventStreams | recent change、revision作成等のSSEイベント |
| 概念・型付き関係 | Wikidata `wbgetentities`、Wikidata Query Service | QID、statement、sitelink、制約付きSPARQL結果 |

Action APIは記事データを返すためのインターフェースであり、記事間の意味的な近さや推薦順位を返すサービスではない。`prop=links`が返すのは「当該ページからリンクされているページ」である。[API:Backlinks](https://www.mediawiki.org/wiki/API:Backlinks)

### カテゴリによる分類

Wikipediaはカテゴリによって記事を分類する。記事は複数のカテゴリに所属でき、カテゴリ自体も別のカテゴリに所属できる。カテゴリページの名前空間IDは14であり、タイトルは`Category:宇宙`のように表される。カテゴリは複数の上位カテゴリを持ち得るため、単一の親だけを持つ分類ツリーではない。[Help:Categories](https://www.mediawiki.org/wiki/Help:Categories)

カテゴリへの所属と、本文中のリンクは別の関係である。カテゴリの取得には`prop=categories`または`list=categorymembers`を使用する。記事に付与された話題の分類のほか、保守・管理用のカテゴリも存在する。非表示カテゴリはAPIで識別・除外できるが、非表示かどうかだけで話題分類と管理分類を完全に区別することはできない。[API:Categories](https://www.mediawiki.org/wiki/API:Categories)

## 3. MediaWiki Action APIの共通リクエスト仕様

### 3.1 パラメーター

| パラメーター | 値 | 意味 |
| --- | --- | --- |
| `action` | `query` または `parse` | APIモジュールを指定する |
| `format` | `json` | JSON形式で応答を受ける |
| `formatversion` | `2` | 現行形式のJSON構造を指定する |
| `origin` | `*` | 匿名のクロスオリジンリクエストを許可する |
| `redirects` | `1` | 指定タイトルがリダイレクトならリンク先へ解決する |

読み取りリクエストはGETで実行できる。`formatversion=2`では、`query.pages`はページオブジェクトの配列で返る。旧形式ではページIDをキーとするオブジェクトであるため、両方の構造を同じものとして扱ってはならない。

リクエスト文字列にはURLエンコードを適用する。特に日本語のタイトル、空白、`|`で区切る複数タイトルは、URLライブラリで安全に組み立てる必要がある。

`action=query`では、`prop`に複数のモジュールを`|`区切りで指定し、同じ記事に関する複数種類のメタデータを1回のリクエストで取得できる。例えば`prop=info|pageprops|categories`は、ページ情報・ページ属性・所属カテゴリをまとめて取得する。各モジュールの追加パラメーター（`inprop=url`、`cllimit=50`など）も同じリクエストに指定でき、結果は対象記事の`query.pages[]`内にまとめて返る。[API:Properties](https://www.mediawiki.org/wiki/API:Properties)

複数指定でも各モジュールの取得件数上限は適用される。1回のリクエストで全件を取得できるとは限らず、`continue`が返った場合は第3.3節に従って継続取得する。[API:Continue](https://www.mediawiki.org/wiki/API:Continue)

### 3.2 CORS、認証、識別

Action APIは、匿名のクロスオリジンリクエストに対して`origin=*`を指定できる。この場合、応答は資格情報を伴わない匿名アクセスとして扱われる。ログイン、OAuth、APIキーは、ここで扱う公開記事の読み取りには不要である。[CORS仕様](https://www.mediawiki.org/wiki/API:Cross-site_requests)

ブラウザでは通常、`origin=*`を含むURLへCORSリクエストを送る。公開アプリケーションは、識別と連絡先を含む`Api-User-Agent`ヘッダーを設定することが推奨される。ブラウザが管理する`User-Agent`ヘッダーを変更することはできない。`Api-User-Agent`を使用する場合、ブラウザはプリフライトを送る場合がある。[API Etiquette](https://www.mediawiki.org/wiki/API:Etiquette) [User-Agent policy](https://foundation.wikimedia.org/wiki/Policy:User-Agent_policy)

`no-cors`モードでは応答本文をJavaScriptから読めないため、APIデータを利用する方法にはならない。

### 3.3 継続取得

検索、出リンク、被リンクなど、応答件数が上限を超えるAPIはトップレベルの`continue`オブジェクトを返す。続きのデータを取得するには、同じクエリ条件へ、その`continue`オブジェクトに含まれる**すべて**のパラメーターをそのまま渡す。`continue`がない場合に限り、そのクエリ条件での取得は完了している。

継続トークンは不透明な値である。アプリケーションは内容を解釈・加工せず、別のページや別条件のリクエストに再利用してはならない。[API:Continue](https://www.mediawiki.org/wiki/API:Continue)

継続取得は、列挙開始時点の内容を固定するsnapshot機能ではない。取得途中に記事やリンクテーブルが更新されると、異なる版の状態が混在し得る。再帰探索では開始時と完了時の`lastrevid`を比較し、変化していれば再取得するか、その走査を非固定snapshotとして記録する。厳密な再現が必要な本文リンクは、第4.6節または第4.14節で版を固定して抽出する。

### 3.4 応答とエラー

成功時でも、応答JSONに`error`オブジェクトが含まれることがある。HTTPステータスとJSONの両方を確認する必要がある。負荷・レート制限に関する応答がある場合は、`Retry-After`などの指示に従ってリクエストを遅延する。

指定ページが存在しない場合、ページオブジェクトには`missing`が含まれる。無効なタイトルでは`invalid`が含まれることがある。これらはHTTP成功とは別に処理する。

## 4. APIモジュール仕様

第4.1～4.14節および第4.16～4.19節のcurl例は、第3節の共通パラメーターを含む独立したAction APIのGETリクエストである。第4.15節と第4.20～4.23節は各サービス固有のURL・応答形式に従う。いずれもmacOS/Linuxのシェルで実行できる。`--data-urlencode`が日本語・空白・`|`をエンコードし、`--compressed`が圧縮された応答の受信・展開を行う。

応答例は各curlに対応するJSON本文を示す。200文字を超える応答では、長い文字列を`"…（文字列省略）"`で表し、配列要素を一部省略する。省略箇所は各例に明記する。記事内容・件数・返却順は変化し得る。`continue`は続きがある場合のみ存在し、`batchcomplete`は全件取得の完了を意味しない。

第4.6節の版固定例および第4.13～4.23節で追加したcurl例は、各例を1回ずつ実送信し、HTTP 200、JSONとしての解析可否、記載した主要パスと型を確認した実応答に基づく。固定期間・固定版以外の値は後日の再実行で変化し得る。

`--user-agent`内の`https://example.org/contact`は例示用であり、実利用時には自分の連絡先URLまたはメールアドレスへ置き換える。curlでは`User-Agent`、ブラウザでは`Api-User-Agent`を使う。各例は1回だけ送信し、自動再試行しない。HTTP応答ヘッダーも調べる場合は`--include`を追加し、第5節に従ってステータス・`Retry-After`・JSONの`error`を確認する。curlにはブラウザのCORS制約は適用されない。

### 4.1 記事検索：`list=search`

```sh
curl --get --silent --show-error --compressed \
  --user-agent 'WikipediaApiExample/1.0 (https://example.org/contact)' \
  'https://ja.wikipedia.org/w/api.php' \
  --data-urlencode 'action=query' \
  --data-urlencode 'list=search' \
  --data-urlencode 'srsearch=宇宙' \
  --data-urlencode 'srnamespace=0' \
  --data-urlencode 'srlimit=20' \
  --data-urlencode 'srprop=snippet' \
  --data-urlencode 'format=json' \
  --data-urlencode 'formatversion=2' \
  --data-urlencode 'origin=*'
```

応答例（HTTP 200、JSON構造）：

```json
{
  "batchcomplete": true,
  "continue": {
    "sroffset": 20,
    "continue": "-||"
  },
  "query": {
    "searchinfo": {
      "totalhits": 66556
    },
    "search": [
      {
        "ns": 0,
        "title": "宇宙",
        "pageid": 3957,
        "snippet": "…（文字列省略）"
      }
    ]
  }
}
```

省略箇所：`$.query.search`：先頭1要素のみ掲載。`$.query.search[0].snippet`：文字列を省略。

`query.search`が結果配列、`searchinfo.totalhits`が総ヒット数、`sroffset`が続きの取得位置を表す。

`srsearch`に検索語を指定する。`srnamespace=0`は標準の記事名前空間に限定する指定である。`srlimit`は返却を求める件数であり、検索結果全件数を保証しない。

| パス | 型 | 意味 |
| --- | --- | --- |
| `query.search[]` | array | 検索結果 |
| `query.search[].pageid` | number | ページID |
| `query.search[].title` | string | 記事タイトル |
| `query.search[].snippet` | string | 検索語周辺の抜粋HTML |
| `continue` | object | 次の結果を得る継続情報。存在する場合のみ |

`snippet`は強調用のHTMLを含み得る。これは記事本文そのものではない。[API:Search](https://www.mediawiki.org/wiki/API:Search)

検索バックエンドの数値scoreを返していた`srprop=score`は非推奨で、現在は無視される。既定の`srsort=relevance`で返る配列順位を検索由来の根拠として扱い、検索語、順位、snippet、取得日時を一組で保存する。公開APIから安定した絶対関連度scoreを得られるとは仮定しない。[API:Search](https://www.mediawiki.org/wiki/API:Search)

検索結果へページ属性を同時付与する場合は`generator=search`とし、`srsearch`・`srlimit`等を`gsrsearch`・`gsrlimit`のように`g` prefixへ置き換える。検索順位やsnippetを根拠として厳密に保存する必要がある場合は、まず`list=search`の応答を保存し、その`pageid`をまとめてproperty照会する方が対応関係を明示しやすい。[API:Query：Generators](https://www.mediawiki.org/wiki/API:Query#Generators)

### 4.2 ページ情報：`prop=info`

```sh
curl --get --silent --show-error --compressed \
  --user-agent 'WikipediaApiExample/1.0 (https://example.org/contact)' \
  'https://ja.wikipedia.org/w/api.php' \
  --data-urlencode 'action=query' \
  --data-urlencode 'pageids=3957' \
  --data-urlencode 'prop=info' \
  --data-urlencode 'inprop=url' \
  --data-urlencode 'format=json' \
  --data-urlencode 'formatversion=2' \
  --data-urlencode 'origin=*'
```

応答例（HTTP 200、JSON構造）：

```json
{
  "batchcomplete": true,
  "query": {
    "pages": [
      {
        "pageid": 3957,
        "ns": 0,
        "title": "宇宙",
        "contentmodel": "wikitext",
        "pagelanguage": "ja",
        "pagelanguagehtmlcode": "ja",
        "pagelanguagedir": "ltr",
        "touched": "2026-09-12T04:57:00Z",
        "lastrevid": 110379900,
        "length": 57562,
        "fullurl": "https://ja.wikipedia.org/wiki/%E5%AE%87%E5%AE%99",
        "editurl": "https://ja.wikipedia.org/w/index.php?title=%E5%AE%87%E5%AE%99&action=edit",
        "canonicalurl": "https://ja.wikipedia.org/wiki/%E5%AE%87%E5%AE%99"
      }
    ]
  }
}
```

`query.pages`の各要素がページ情報であり、`lastrevid`は最終版ID、`fullurl`と`canonicalurl`はURL文字列である。

ページは`pageids`または`titles`で指定する。`inprop=url`を指定すると、`fullurl`などのURL情報を取得できる。主な応答フィールドは`pageid`、`ns`、`title`、`fullurl`、`canonicalurl`、`lastrevid`である。

`pageid`は一つのWiki内でページを識別する数値であり、全言語版に共通の識別子ではない。`title`はページ名であり、正規化やリダイレクト解決によって入力と異なることがある。[API:Info](https://www.mediawiki.org/wiki/API:Info)

### 4.3 抜粋：`prop=extracts`

```sh
curl --get --silent --show-error --compressed \
  --user-agent 'WikipediaApiExample/1.0 (https://example.org/contact)' \
  'https://ja.wikipedia.org/w/api.php' \
  --data-urlencode 'action=query' \
  --data-urlencode 'pageids=3957' \
  --data-urlencode 'prop=extracts' \
  --data-urlencode 'exintro=1' \
  --data-urlencode 'explaintext=1' \
  --data-urlencode 'exchars=600' \
  --data-urlencode 'format=json' \
  --data-urlencode 'formatversion=2' \
  --data-urlencode 'origin=*'
```

応答例（HTTP 200）：

```json
{
  "batchcomplete": true,
  "query": {
    "pages": [
      {
        "pageid": 3957,
        "ns": 0,
        "title": "宇宙",
        "extract": "宇宙（うちゅう）について、本項では漢語（およびその借用語）としての「宇宙」と、「宇宙」と漢語訳される様々な概念を扱う。\n\n"
      }
    ]
  }
}
```

抜粋は`query.pages[].extract`に入る。この例では`explaintext=1`を指定しているため、値はプレーンテキスト文字列である。

`extracts`は記事から抜粋したテキストまたは限定HTMLを返す。`exintro=1`は最初の節より前の内容に限定する。`explaintext=1`はプレーンテキスト形式を指定する。`exchars`は返却を求める文字数であるが、厳密な上限ではない。

抜粋は要約の生成結果ではなく、記事本文から抽出された部分である。記事の構造によっては空の抜粋が返ることがある。複数ページの抜粋取得には`exintro=1`が必要であり、`exlimit`の上限は20である。[TextExtracts](https://www.mediawiki.org/wiki/Extension:TextExtracts#API)

### 4.4 出リンク：`prop=links`

```sh
curl --get --silent --show-error --compressed \
  --user-agent 'WikipediaApiExample/1.0 (https://example.org/contact)' \
  'https://ja.wikipedia.org/w/api.php' \
  --data-urlencode 'action=query' \
  --data-urlencode 'pageids=3957' \
  --data-urlencode 'prop=links' \
  --data-urlencode 'plnamespace=0' \
  --data-urlencode 'pllimit=50' \
  --data-urlencode 'format=json' \
  --data-urlencode 'formatversion=2' \
  --data-urlencode 'origin=*'
```

`prop=links`は指定ページからリンクしているページを返す。`plnamespace=0`で記事名前空間へのリンクに限定できる。一般クライアントにおける`pllimit`の許容範囲は1から500であり、既定値は10である。

応答例（HTTP 200、JSON構造）：

```json
{
  "continue": {
    "plcontinue": "3957|0|ウィクショナリー",
    "continue": "||"
  },
  "query": {
    "pages": [
      {
        "pageid": 3957,
        "ns": 0,
        "title": "宇宙",
        "links": [
          {
            "ns": 0,
            "title": "1915年"
          }
        ]
      }
    ]
  }
}
```

省略箇所：`$.query.pages[0].links`：先頭1要素のみ掲載。

`pages[]`はリンク元、`links[]`はリンク先を表す。次のリンク群は`plcontinue`を含む`continue`全体を渡して取得する。

`links[]`の要素には少なくとも`ns`と`title`が返る。このモジュールが返すリンク先は、個々のリンク先のページID、正規URL、存在可否を常に含むものではない。これらが必要な場合は、返されたタイトルを別の`action=query`で照会する。[API:Links](https://www.mediawiki.org/wiki/API:Links)

出リンクは記事の本文だけでなく、テンプレートなどのページ構成要素に由来するリンクを含み得る。また、返却順は意味的な関連度や記事内での重要度を表さない。

`prop=links`は現在のlink tableを列挙する機能であり、`oldid`を指定して過去版のリンクだけを取得する機能ではない。版固定の辺集合が必要な場合は、最新版IDを保存したうえで`action=parse&oldid=<版ID>`または`prop=revisions`で得たWikitextからリンクを抽出する。

### 4.5 タイトル解決とページ属性：`titles`、`redirects`、`pageprops`

```sh
curl --get --silent --show-error --compressed \
  --user-agent 'WikipediaApiExample/1.0 (https://example.org/contact)' \
  'https://ja.wikipedia.org/w/api.php' \
  --data-urlencode 'action=query' \
  --data-urlencode 'titles=NASA|水星 (曖昧さ回避)' \
  --data-urlencode 'prop=info|pageprops' \
  --data-urlencode 'inprop=url' \
  --data-urlencode 'ppprop=disambiguation' \
  --data-urlencode 'redirects=1' \
  --data-urlencode 'format=json' \
  --data-urlencode 'formatversion=2' \
  --data-urlencode 'origin=*'
```

応答例（HTTP 200、JSON構造）：

```json
{
  "batchcomplete": true,
  "query": {
    "redirects": [
      {
        "from": "NASA",
        "to": "アメリカ航空宇宙局"
      }
    ],
    "pages": [
      {
        "pageid": 1065,
        "ns": 0,
        "title": "アメリカ航空宇宙局",
        "contentmodel": "wikitext",
        "pagelanguage": "ja",
        "pagelanguagehtmlcode": "ja",
        "pagelanguagedir": "ltr",
        "touched": "2026-09-04T13:00:13Z",
        "lastrevid": 110792602,
        "length": 49443,
        "fullurl": "…（文字列省略）",
        "editurl": "…（文字列省略）",
        "canonicalurl": "…（文字列省略）"
      },
      {
        "pageid": 5188157,
        "ns": 0,
        "title": "水星 (曖昧さ回避)",
        "contentmodel": "wikitext",
        "pagelanguage": "ja",
        "pagelanguagehtmlcode": "ja",
        "pagelanguagedir": "ltr",
        "touched": "2026-01-08T03:36:26Z",
        "lastrevid": 107922859,
        "length": 396,
        "new": true,
        "fullurl": "https://ja.wikipedia.org/wiki/%E6%B0%B4%E6%98%9F_(%E6%9B%96%E6%98%A7%E3%81%95%E5%9B%9E%E9%81%BF)",
        "editurl": "…（文字列省略）",
        "canonicalurl": "https://ja.wikipedia.org/wiki/%E6%B0%B4%E6%98%9F_(%E6%9B%96%E6%98%A7%E3%81%95%E5%9B%9E%E9%81%BF)",
        "pageprops": {
          "disambiguation": ""
        }
      }
    ]
  }
}
```

省略箇所：`$.query.pages[0].fullurl`：文字列を省略。`$.query.pages[0].editurl`：文字列を省略。`$.query.pages[0].canonicalurl`：文字列を省略。`$.query.pages[1].editurl`：文字列を省略。

`redirects`がNASAから解決先への対応、`pages`が解決後のページ情報を表す。この例の2要素目は曖昧さ回避ページである。配列順は保証されない。`disambiguation`の空文字は省略表現ではなく実際に返り得る値。`normalized`は正規化が発生した場合のみ返るため、この呼び出し例には含めていない。

複数タイトルは`|`で指定できる。入力タイトルの正規化結果は`query.normalized`、リダイレクト解決結果は`query.redirects`に返る。`redirects=1`は、指定タイトルがリダイレクトである場合にリンク先ページを返す指定である。

`prop=pageprops&ppprop=disambiguation`を指定すると、曖昧さ回避ページは`pageprops.disambiguation`キーを持つ。値が空文字であっても、キーの存在で判定する。

なお、`prop=redirects`は「指定ページへ向かうリダイレクト一覧」を取得する別機能であり、ここで説明した`redirects=1`とは異なる。[API:Query](https://www.mediawiki.org/wiki/API:Query#Resolving_redirects) [API:Pageprops](https://www.mediawiki.org/wiki/API:Pageprops)

### 4.6 解析済み本文：`action=parse`

```sh
curl --get --silent --show-error --compressed \
  --user-agent 'WikipediaApiExample/1.0 (https://example.org/contact)' \
  'https://ja.wikipedia.org/w/api.php' \
  --data-urlencode 'action=parse' \
  --data-urlencode 'pageid=3957' \
  --data-urlencode 'prop=text|links|revid' \
  --data-urlencode 'redirects=1' \
  --data-urlencode 'format=json' \
  --data-urlencode 'formatversion=2' \
  --data-urlencode 'origin=*'
```

応答例（HTTP 200、JSON構造）：

```json
{
  "parse": {
    "title": "宇宙",
    "pageid": 3957,
    "revid": 110379900,
    "redirects": [

    ],
    "text": "…（文字列省略）",
    "links": [
      {
        "ns": 10,
        "title": "Template:Physical cosmology",
        "exists": true
      }
    ]
  }
}
```

省略箇所：`$.parse.text`：文字列を省略。`$.parse.links`：先頭1要素のみ掲載。

`query`ではなく`parse`がルート直下に返る。`text`はHTML文字列、`links`は内部リンクの配列であり、存在するリンク先には`exists: true`が付く。

`action=parse`はページをMediaWikiで解析した結果を返す。`text`はHTML文字列、`links`は解析時に得られた内部リンクの一覧、`revid`は解析対象の版IDである。

`parse.links[]`はリンク先の一覧であり、各出現の文字offset、節、画面位置を返す構造ではない。リンクの出現位置、重複回数、導入部・脚注・関連項目の区別を特徴にする場合は、同じ`revid`の`text` HTMLまたはWikitextを走査して根拠箇所を保存する。

これはWikitextをそのまま返すAPIではない。`text`はHTMLであり、API利用者側でページのスクリプトや表示様式が提供されるわけではない。HTMLの構造やテンプレートの展開結果は記事の編集により変化し得る。[API:Parsing wikitext](https://www.mediawiki.org/wiki/API:Parsing_wikitext)

記事内の節を識別する場合は、旧来の`prop=sections`ではなく`prop=tocdata`を使用する。`prop=sections`は非推奨である。`tocdata.sections[]`は見出し、目次番号、`anchor`、Wikitext上の`codepointOffset`等を返すが、`links`の各要素へ節番号を直接付与するものではない。節ごとのリンクを分けるには、`anchor`とHTML DOM上の見出しを対応付けるか、`codepointOffset`で版固定Wikitextを区切る。`tocdata.sections[].index`は編集用hashであり、`action=parse`の`section`へそのまま渡せる安定IDとはみなさない。[TOCData schema](https://www.mediawiki.org/wiki/API:Parsing_wikitext/TOCData)

特定版を再現する場合は、`pageid`と`oldid`を同時に指定せず、版IDだけを`oldid`へ指定する。

```sh
curl --get --silent --show-error --compressed \
  --user-agent 'WikipediaApiExample/1.0 (https://example.org/contact)' \
  'https://ja.wikipedia.org/w/api.php' \
  --data-urlencode 'action=parse' \
  --data-urlencode 'oldid=110379900' \
  --data-urlencode 'prop=text|links|tocdata|revid' \
  --data-urlencode 'format=json' \
  --data-urlencode 'formatversion=2'
```

応答例（HTTP 200、JSON構造）：

```json
{
  "parse": {
    "title": "宇宙",
    "pageid": 3957,
    "text": "…（文字列省略）",
    "links": [
      {
        "ns": 10,
        "title": "Template:Physical cosmology",
        "exists": true
      }
    ],
    "revid": 110379900,
    "tocdata": {
      "sections": [
        {
          "tocLevel": 1,
          "hLevel": 2,
          "line": "定義",
          "number": "1",
          "index": "1",
          "fromTitle": "宇宙",
          "codepointOffset": 4086,
          "anchor": "定義"
        }
      ],
      "extensionData": []
    }
  }
}
```

省略箇所：`$.parse.text`：203,347文字のHTMLを省略。`$.parse.links`：実応答444要素の先頭1要素のみ掲載。`$.parse.tocdata.sections`：実応答32要素の先頭1要素のみ掲載。

`oldid`は`page`・`pageid`・`text`と排他的である。応答の`parse.revid`が要求した版IDと一致することを検査し、HTML、抽出した辺、節位置を同じ版IDへ関連付けて保存する。`action=parse`による特定版の解析はサーバー負荷が高いため、一次選抜した記事だけに用いる。[API:Parsing wikitext](https://www.mediawiki.org/wiki/API:Parsing_wikitext) [API Etiquette：Parsing of revisions](https://www.mediawiki.org/wiki/API:Etiquette#Parsing_of_revisions)

### 4.7 被リンク：`list=backlinks`

```sh
curl --get --silent --show-error --compressed \
  --user-agent 'WikipediaApiExample/1.0 (https://example.org/contact)' \
  'https://ja.wikipedia.org/w/api.php' \
  --data-urlencode 'action=query' \
  --data-urlencode 'list=backlinks' \
  --data-urlencode 'bltitle=宇宙' \
  --data-urlencode 'blnamespace=0' \
  --data-urlencode 'bllimit=50' \
  --data-urlencode 'format=json' \
  --data-urlencode 'formatversion=2' \
  --data-urlencode 'origin=*'
```

応答例（HTTP 200、JSON構造）：

```json
{
  "batchcomplete": true,
  "continue": {
    "blcontinue": "0|6670",
    "continue": "-||"
  },
  "query": {
    "backlinks": [
      {
        "pageid": 12,
        "ns": 0,
        "title": "地理学"
      }
    ]
  }
}
```

省略箇所：`$.query.backlinks`：先頭1要素のみ掲載。

`query.backlinks`の各要素は「宇宙」へリンクするページを表す。次の取得には`blcontinue`を含む`continue`全体を引き継ぐ。

`list=backlinks`は、指定タイトルへリンクしているページを取得する。出リンクとは方向が逆の関係であり、`prop=links`と同一の結果にはならない。多件数の場合は`continue`を使用する。[API:Backlinks](https://www.mediawiki.org/wiki/API:Backlinks)

### 4.8 記事の所属カテゴリ：`prop=categories`

```sh
curl --get --silent --show-error --compressed \
  --user-agent 'WikipediaApiExample/1.0 (https://example.org/contact)' \
  'https://ja.wikipedia.org/w/api.php' \
  --data-urlencode 'action=query' \
  --data-urlencode 'titles=宇宙' \
  --data-urlencode 'prop=categories' \
  --data-urlencode 'clshow=!hidden' \
  --data-urlencode 'cllimit=50' \
  --data-urlencode 'format=json' \
  --data-urlencode 'formatversion=2' \
  --data-urlencode 'origin=*'
```

応答例（HTTP 200、JSON構造）：

```json
{
  "batchcomplete": true,
  "query": {
    "pages": [
      {
        "pageid": 3957,
        "ns": 0,
        "title": "宇宙",
        "categories": [
          {
            "ns": 14,
            "title": "Category:天体物理学"
          }
        ]
      }
    ]
  }
}
```

省略箇所：`$.query.pages[0].categories`：先頭1要素のみ掲載。

`query.pages[].categories`は記事が直接所属するカテゴリ。`clshow=!hidden`は非表示カテゴリを除外する指定である。非表示も含めて取得する場合はこの指定を外し、`clprop=hidden`を追加すると非表示カテゴリに`hidden`属性が付く。上位カテゴリへの所属は自動展開されない。続きは`clcontinue`を含む`continue`全体で取得する。[API:Categories](https://www.mediawiki.org/wiki/API:Categories)

### 4.9 カテゴリに所属する記事：`list=categorymembers`

```sh
curl --get --silent --show-error --compressed \
  --user-agent 'WikipediaApiExample/1.0 (https://example.org/contact)' \
  'https://ja.wikipedia.org/w/api.php' \
  --data-urlencode 'action=query' \
  --data-urlencode 'list=categorymembers' \
  --data-urlencode 'cmtitle=Category:宇宙' \
  --data-urlencode 'cmnamespace=0' \
  --data-urlencode 'cmprop=ids|title' \
  --data-urlencode 'cmlimit=50' \
  --data-urlencode 'format=json' \
  --data-urlencode 'formatversion=2' \
  --data-urlencode 'origin=*'
```

応答例（HTTP 200、JSON構造）：

```json
{
  "batchcomplete": true,
  "query": {
    "categorymembers": [
      {
        "pageid": 3957,
        "ns": 0,
        "title": "宇宙"
      }
    ]
  }
}
```

省略箇所：`$.query.categorymembers`：先頭1要素のみ掲載。

`query.categorymembers`は指定カテゴリに直接所属するページの配列。`cmtitle`には`Category:`を含め、`cmnamespace=0`で通常の記事だけに限定する。下位カテゴリ内の記事は自動では含まれない。名前空間による絞り込みでは、返却が指定件数未満または0件でも`continue`が残る場合があるため、空配列だけで取得完了と判定しない。[API:Categorymembers](https://www.mediawiki.org/wiki/API:Categorymembers)

### 4.10 下位カテゴリ：`list=categorymembers&cmtype=subcat`

```sh
curl --get --silent --show-error --compressed \
  --user-agent 'WikipediaApiExample/1.0 (https://example.org/contact)' \
  'https://ja.wikipedia.org/w/api.php' \
  --data-urlencode 'action=query' \
  --data-urlencode 'list=categorymembers' \
  --data-urlencode 'cmtitle=Category:宇宙' \
  --data-urlencode 'cmtype=subcat' \
  --data-urlencode 'cmprop=ids|title|type' \
  --data-urlencode 'cmlimit=50' \
  --data-urlencode 'format=json' \
  --data-urlencode 'formatversion=2' \
  --data-urlencode 'origin=*'
```

応答例（HTTP 200、JSON構造）：

```json
{
  "batchcomplete": true,
  "query": {
    "categorymembers": [
      {
        "pageid": 206696,
        "ns": 14,
        "title": "Category:宇宙空間",
        "type": "subcat"
      }
    ]
  }
}
```

省略箇所：`$.query.categorymembers`：先頭1要素のみ掲載。

`cmtype=subcat`で直接の下位カテゴリを取得する。`cmprop=type`による`type: "subcat"`は、結果が下位カテゴリであることを表す。孫以下を取得するには、返却されたカテゴリを対象に追加リクエストが必要である。`cmtype`は`cmsort=timestamp`指定時には無視されるため、この例では既定の並び順を使う。[API:Categorymembers](https://www.mediawiki.org/wiki/API:Categorymembers)

### 4.11 上位カテゴリ：カテゴリページへの`prop=categories`

```sh
curl --get --silent --show-error --compressed \
  --user-agent 'WikipediaApiExample/1.0 (https://example.org/contact)' \
  'https://ja.wikipedia.org/w/api.php' \
  --data-urlencode 'action=query' \
  --data-urlencode 'titles=Category:宇宙' \
  --data-urlencode 'prop=categories' \
  --data-urlencode 'clshow=!hidden' \
  --data-urlencode 'cllimit=50' \
  --data-urlencode 'format=json' \
  --data-urlencode 'formatversion=2' \
  --data-urlencode 'origin=*'
```

応答例（HTTP 200）：

```json
{
  "batchcomplete": true,
  "query": {
    "pages": [
      {
        "pageid": 138483,
        "ns": 14,
        "title": "Category:宇宙",
        "categories": [
          {
            "ns": 14,
            "title": "Category:自然"
          }
        ]
      }
    ]
  }
}
```

カテゴリページを`titles`で指定すると、`categories`にそのカテゴリが直接所属する上位カテゴリが返る。上位カテゴリは複数存在し得る。祖先全体を一度に取得する機能ではなく、さらに上位を取得する場合も同じAPIを繰り返す。[API:Categories](https://www.mediawiki.org/wiki/API:Categories)

### 4.12 複数種類のメタデータ：`prop=info|pageprops|categories`

```sh
curl --get --silent --show-error --compressed \
  --user-agent 'WikipediaApiExample/1.0 (https://example.org/contact)' \
  'https://ja.wikipedia.org/w/api.php' \
  --data-urlencode 'action=query' \
  --data-urlencode 'titles=宇宙' \
  --data-urlencode 'prop=info|pageprops|categories' \
  --data-urlencode 'inprop=url' \
  --data-urlencode 'cllimit=1' \
  --data-urlencode 'format=json' \
  --data-urlencode 'formatversion=2' \
  --data-urlencode 'origin=*'
```

応答例（HTTP 200、JSON構造）：

```json
{
  "continue": {
    "clcontinue": "3957|出典を必要とする節のある記事/2022年1月",
    "continue": "||info|pageprops"
  },
  "query": {
    "pages": [
      {
        "pageid": 3957,
        "ns": 0,
        "title": "宇宙",
        "contentmodel": "wikitext",
        "pagelanguage": "ja",
        "pagelanguagehtmlcode": "ja",
        "pagelanguagedir": "ltr",
        "touched": "2026-09-12T04:57:00Z",
        "lastrevid": 110379900,
        "length": 57562,
        "fullurl": "https://ja.wikipedia.org/wiki/%E5%AE%87%E5%AE%99",
        "editurl": "…（文字列省略）",
        "canonicalurl": "…（文字列省略）",
        "pageprops": {
          "defaultsort": "うちゆう",
          "page_image_free": "Hubble_ultra_deep_field.jpg",
          "wikibase_item": "Q1"
        },
        "categories": [
          {
            "ns": 14,
            "title": "Category:Reflistで3列を指定しているページ"
          }
        ]
      }
    ]
  }
}
```

省略箇所：`$.query.pages[0].editurl`：文字列を省略。`$.query.pages[0].canonicalurl`：文字列を省略。

`prop`を`|`で区切ると、ページ情報・ページ属性・所属カテゴリを1回のリクエストで取得できる。`titles`は対象記事、`inprop=url`はURL情報の追加、`cllimit=1`はカテゴリの1回あたりの取得件数を指定する。[API:Properties](https://www.mediawiki.org/wiki/API:Properties)

同じ`query.pages[]`要素に、`info`の`pageid`・`fullurl`など、`pageprops`のページ属性、`categories`の所属カテゴリが返る。`info`という入れ子のオブジェクトは作られない。`pageprops.wikibase_item`は対応するWikidata項目のIDである。

この例ではカテゴリを1件に制限しているため、続きの取得に使う`continue`が返っている。残りを取得するには、第3.3節に従って`clcontinue`と`continue`の両方を同じクエリ条件に追加する。複数種類の同時取得は、全件が1回で返ることを保証しない。[API:Continue](https://www.mediawiki.org/wiki/API:Continue)

### 4.13 出リンク先の一括属性取得：`generator=links`

```sh
curl --get --silent --show-error --compressed \
  --user-agent 'WikipediaApiExample/1.0 (https://example.org/contact)' \
  'https://ja.wikipedia.org/w/api.php' \
  --data-urlencode 'action=query' \
  --data-urlencode 'pageids=3957' \
  --data-urlencode 'generator=links' \
  --data-urlencode 'gplnamespace=0' \
  --data-urlencode 'gpllimit=20' \
  --data-urlencode 'prop=info|pageprops|categories|extracts|revisions' \
  --data-urlencode 'inprop=url' \
  --data-urlencode 'ppprop=disambiguation|wikibase_item' \
  --data-urlencode 'clshow=!hidden' \
  --data-urlencode 'cllimit=10' \
  --data-urlencode 'exintro=1' \
  --data-urlencode 'explaintext=1' \
  --data-urlencode 'exchars=400' \
  --data-urlencode 'exlimit=20' \
  --data-urlencode 'rvprop=ids|timestamp' \
  --data-urlencode 'redirects=1' \
  --data-urlencode 'format=json' \
  --data-urlencode 'formatversion=2' \
  --data-urlencode 'origin=*'
```

応答例（HTTP 200、JSON構造）：

```json
{
  "continue": {
    "clcontinue": "4145|書物",
    "continue": "||info|pageprops|extracts|revisions"
  },
  "query": {
    "pages": [
      {
        "pageid": 1929,
        "ns": 0,
        "title": "1965年",
        "lastrevid": 110628682,
        "length": 75210,
        "fullurl": "…（文字列省略）",
        "pageprops": {
          "wikibase_item": "Q2650"
        },
        "categories": [
          {
            "ns": 14,
            "title": "Category:1965年"
          }
        ],
        "extract": "1965年（1965 ねん）は、西暦（グレゴリオ暦）による、金曜日から始まる平年。昭和40年。\nこの項目では、国際的な視点に基づいた1965年について記載する。",
        "revisions": [
          {
            "revid": 110628682,
            "parentid": 110093515,
            "timestamp": "2026-08-13T07:17:24Z"
          }
        ]
      }
    ]
  }
}
```

省略箇所：`$.query.pages`：実応答20要素の先頭1要素のみ掲載。`$.query.pages[0].fullurl`：文字列を省略。ページ情報のうち`contentmodel`、言語、`touched`、`editurl`、`canonicalurl`を省略。

generatorは、あるモジュールが列挙したページを、`titles`や`pageids`の代わりに後続のpropertyモジュールへ渡す。`generator=links`では元ページの出リンク先が`query.pages[]`となり、各要素に`info`、`pageprops`、`categories`、`extract`、`revisions`の結果が併合される。generator自身のパラメーターには`g`を付けるため、`plnamespace`と`pllimit`ではなく`gplnamespace`と`gpllimit`を使う。[API:Query：Generators](https://www.mediawiki.org/wiki/API:Query#Generators)

generatorはリンク元とリンク先の辺を別構造で返すのではなく、リンク先ページをproperty照会の対象として返す。したがって、複数の親記事を一度に処理する場合は、どの親から候補が生じたかを失わないよう、元の`prop=links`結果またはワークキュー側で親子対応を保持する。

generatorとpropertyのいずれも継続を発生させ得る。応答に`continue`がある間は、`gplcontinue`、`clcontinue`、`continue`等の**すべて**を同じ条件へ渡す。`batchcomplete`は現在のgeneratorバッチに属する各ページについてpropertyデータが完了したことを示すが、クエリ全体に続きがないことを意味しない。`batchcomplete`前に同じ`pageid`が複数応答へ現れる場合に備え、ページ単位で属性配列を追記・重複除去する。[API:Query：Batchcomplete](https://www.mediawiki.org/wiki/API:Query#Batchcomplete) [API:Continue](https://www.mediawiki.org/wiki/API:Continue)

### 4.14 版ID・Wikitext・版履歴：`prop=revisions`

```sh
curl --get --silent --show-error --compressed \
  --user-agent 'WikipediaApiExample/1.0 (https://example.org/contact)' \
  'https://ja.wikipedia.org/w/api.php' \
  --data-urlencode 'action=query' \
  --data-urlencode 'pageids=3957' \
  --data-urlencode 'prop=revisions' \
  --data-urlencode 'rvprop=ids|timestamp|contentmodel|content' \
  --data-urlencode 'rvslots=main' \
  --data-urlencode 'rvlimit=1' \
  --data-urlencode 'format=json' \
  --data-urlencode 'formatversion=2' \
  --data-urlencode 'origin=*'
```

応答例（HTTP 200、JSON構造）：

```json
{
  "query": {
    "pages": [
      {
        "pageid": 3957,
        "ns": 0,
        "title": "宇宙",
        "revisions": [
          {
            "revid": 110379900,
            "parentid": 109979185,
            "timestamp": "2026-07-23T10:16:51Z",
            "slots": {
              "main": {
                "contentmodel": "wikitext",
                "content": "…（文字列省略）"
              }
            }
          }
        ]
      }
    ]
  }
}
```

省略箇所：`$.query.pages[0].revisions[0].slots.main.content`：30,137文字のWikitextを省略。

最新版の版IDは`query.pages[].revisions[0].revid`、親版IDは`parentid`、時刻は`timestamp`に入る。`rvslots=main`を指定した本文は`revisions[].slots.main.content`、内容モデルは`revisions[].slots.main.contentmodel`に入る。記事本文の全文特徴量やWikitext上のリンク位置を検証する場合は、`extracts`ではなくこの本文ソースを利用できる。ただし、Wikitextのテンプレートは展開前なので、画面上の本文順序を検証する場合はparse結果またはREST HTMLを使う。

特定版をIDで取得する場合は`titles`や`pageids`の代わりに`revids=<版ID>`を使う。`revids`指定では`redirects=1`を併用できず、版履歴向けの`rvlimit`、`rvstart`、`rvend`等とも併用しない。解析済みHTMLとリンクが必要なら第4.6節の`action=parse&oldid=<版ID>`、REST HTMLが必要なら第4.15節の版IDルートを使う。[API:Revisions](https://www.mediawiki.org/wiki/API:Revisions)

一つの記事の版履歴を列挙する場合は、対象を一ページに限定し、`rvprop=ids|timestamp`、`rvlimit=50`等を指定して`rvcontinue`を処理する。`rvprop=content`を含むと1回の上限は50版に制限される。時間安定性を検証する際は、版一覧だけからリンク差分を推定せず、必要な版のソースまたはparse結果を比較する。

### 4.15 最新版・特定版HTML：MediaWiki REST API

MediaWiki REST APIの日本語版Wikipediaにおける基準URLは次のとおりである。

```text
https://ja.wikipedia.org/w/rest.php/v1/
```

最新版のHTMLと版情報を一つのJSONで取得する例：

```sh
curl --silent --show-error --compressed \
  --user-agent 'WikipediaApiExample/1.0 (https://example.org/contact)' \
  'https://ja.wikipedia.org/w/rest.php/v1/page/%E5%AE%87%E5%AE%99/with_html'
```

応答例（HTTP 200、JSON構造）：

```json
{
  "id": 3957,
  "key": "宇宙",
  "title": "宇宙",
  "latest": {
    "id": 110379900,
    "timestamp": "2026-07-23T10:16:51Z"
  },
  "content_model": "wikitext",
  "license": {
    "url": "https://creativecommons.org/licenses/by-sa/4.0/deed.ja",
    "title": "Creative Commons Attribution-Share Alike 4.0"
  },
  "html": "…（文字列省略）"
}
```

省略箇所：`$.html`：255,321文字のHTMLを省略。

応答はページオブジェクトであり、`id`がページID、`key`と`title`がタイトル、`latest.id`が最新版ID、`latest.timestamp`が版時刻、`html`がHTML文字列、`license`がライセンス情報である。HTMLだけが必要なら`/page/{title}/html`を使用できる。タイトルはURLのpath segmentとしてエンコードし、サブページ名の`/`は`%2F`にする。[MediaWiki REST API reference](https://www.mediawiki.org/wiki/API:REST_API/Reference#Pages)

ページ取得ルートにリダイレクトタイトルを指定すると、既定ではHTTP 307と遷移先ルートを示す`Location`が返る。curlで解決先まで取得する場合は`--location`を指定して307を明示的に追跡する。入力したリダイレクトページ自体を監査する場合は`?redirect=no`を指定すると、そのページの版情報と内容をHTTP 200で取得できる。HTTP 301のタイトル正規化、HTTP 307のwiki redirect、HTTP 200の内容取得を区別し、要求した表記と最終応答の`id`・`key`、途中の`Location`を保存する。

最新版の本文ソースとメタデータは`/page/{title}`で取得でき、ページオブジェクトの`source`にcontent modelに応じたソースが入る。HTML解析とWikitext解析を比較する場合は、両応答の`latest.id`が同じであることを確認する。

特定版のHTMLは次のルートで取得する。

```text
https://ja.wikipedia.org/w/rest.php/v1/revision/{revision-id}/html
```

版情報とHTMLを同じJSONで必要とする場合は`/revision/{revision-id}/with_html`を使う。最新版URLと特定版URLを混在させず、保存したHTMLには応答の版IDまたは要求した版IDを必ず関連付ける。[MediaWiki REST API reference：History](https://www.mediawiki.org/wiki/API:REST_API/Reference#History)

REST応答が`ETag`を返す場合、再取得時にその値を`If-None-Match`へ渡せる。変更がなければHTTP 304、変更があればHTTP 200と新しい`ETag`が返る。`ETag`がなく`Last-Modified`があるendpointでは`If-Modified-Since`を使用する。すべてのendpointが同じvalidatorを返すとは仮定しない。[Wikimedia APIs：Conditional requests](https://www.mediawiki.org/wiki/Wikimedia_APIs/Conditional_requests)

### 4.16 言語間リンク：`prop=langlinks`

```sh
curl --get --silent --show-error --compressed \
  --user-agent 'WikipediaApiExample/1.0 (https://example.org/contact)' \
  'https://ja.wikipedia.org/w/api.php' \
  --data-urlencode 'action=query' \
  --data-urlencode 'pageids=3957' \
  --data-urlencode 'prop=langlinks' \
  --data-urlencode 'llprop=url|langname|autonym' \
  --data-urlencode 'llinlanguagecode=ja' \
  --data-urlencode 'lllimit=50' \
  --data-urlencode 'format=json' \
  --data-urlencode 'formatversion=2' \
  --data-urlencode 'origin=*'
```

応答例（HTTP 200、JSON構造）：

```json
{
  "continue": {
    "llcontinue": "3957|ext",
    "continue": "||"
  },
  "query": {
    "pages": [
      {
        "pageid": 3957,
        "ns": 0,
        "title": "宇宙",
        "langlinks": [
          {
            "lang": "ab",
            "url": "https://ab.wikipedia.org/wiki/%D0%90%D0%B4%D1%83%D0%BD%D0%B5%D0%B8",
            "langname": "アブハズ語",
            "autonym": "аԥсшәа",
            "title": "Адунеи"
          }
        ]
      }
    ]
  }
}
```

省略箇所：`$.query.pages[0].langlinks`：実応答50要素の先頭1要素のみ掲載。

`query.pages[].langlinks[]`に言語コード、他言語版のタイトル、および要求した追加属性が返る。続きは`llcontinue`を含む`continue`全体で取得する。言語間リンクは別wikiのページを指すため、`pageid`は移送されない。例えば`lang=fr`の結果はフランス語版の`https://fr.wikipedia.org/w/api.php`でタイトルを照会し、記事識別子を`wiki + pageid`の組として保持する。[API:Langlinks](https://www.mediawiki.org/wiki/API:Langlinks)

同一概念の言語版を束ねる用途では、`pageprops.wikibase_item`でQIDを取得し、第4.22節のWikidata sitelinkも照合する。QIDで識別する概念と、各言語版の`wiki + pageid`で識別する記事を別レコードとして扱うと、言語間対応と記事間リンクを混同しない。

### 4.17 テンプレート使用関係：`prop=templates`、`list=embeddedin`

記事が使用するテンプレートは次のように取得する。

```sh
curl --get --silent --show-error --compressed \
  --user-agent 'WikipediaApiExample/1.0 (https://example.org/contact)' \
  'https://ja.wikipedia.org/w/api.php' \
  --data-urlencode 'action=query' \
  --data-urlencode 'titles=宇宙' \
  --data-urlencode 'prop=templates' \
  --data-urlencode 'tlnamespace=10' \
  --data-urlencode 'tllimit=50' \
  --data-urlencode 'format=json' \
  --data-urlencode 'formatversion=2'
```

応答例（HTTP 200、JSON構造）：

```json
{
  "continue": {
    "tlcontinue": "3957|10|Nowrap",
    "continue": "||"
  },
  "query": {
    "pages": [
      {
        "pageid": 3957,
        "ns": 0,
        "title": "宇宙",
        "templates": [
          {
            "ns": 10,
            "title": "Template:Abbr"
          }
        ]
      }
    ]
  }
}
```

省略箇所：`$.query.pages[0].templates`：実応答50要素の先頭1要素のみ掲載。

逆に、指定テンプレートを埋め込む通常記事は次のように取得する。

```sh
curl --get --silent --show-error --compressed \
  --user-agent 'WikipediaApiExample/1.0 (https://example.org/contact)' \
  'https://ja.wikipedia.org/w/api.php' \
  --data-urlencode 'action=query' \
  --data-urlencode 'list=embeddedin' \
  --data-urlencode 'eititle=Template:基礎情報 人物' \
  --data-urlencode 'einamespace=0' \
  --data-urlencode 'eilimit=50' \
  --data-urlencode 'format=json' \
  --data-urlencode 'formatversion=2'
```

応答例（HTTP 200、JSON構造）：

```json
{
  "batchcomplete": true,
  "query": {
    "embeddedin": []
  }
}
```

省略箇所：なし。指定条件に一致する要素はなかった。

前者は`query.pages[].templates[]`、後者は`query.embeddedin[]`を返す。`tlcontinue`または`eicontinue`を処理する。テンプレートの埋込み関係は本文リンクとは別の関係であり、汎用テンプレートや保守テンプレートは非常に高次数になり得る。名前空間、テンプレートallowlist、次数上限を適用し、`transclusion`型の辺として保存する。[API:Templates](https://www.mediawiki.org/wiki/API:Templates) [API:Embeddedin](https://www.mediawiki.org/wiki/API:Embeddedin)

### 4.18 外部URL関係：`prop=extlinks`、`list=exturlusage`

```sh
curl --get --silent --show-error --compressed \
  --user-agent 'WikipediaApiExample/1.0 (https://example.org/contact)' \
  'https://ja.wikipedia.org/w/api.php' \
  --data-urlencode 'action=query' \
  --data-urlencode 'titles=宇宙' \
  --data-urlencode 'prop=extlinks' \
  --data-urlencode 'ellimit=50' \
  --data-urlencode 'format=json' \
  --data-urlencode 'formatversion=2'
```

応答例（HTTP 200、JSON構造）：

```json
{
  "continue": {
    "elcontinue": "45204239",
    "continue": "||"
  },
  "query": {
    "pages": [
      {
        "pageid": 3957,
        "ns": 0,
        "title": "宇宙",
        "extlinks": [
          {
            "url": "https://d-nb.info/gnd/4079154-3"
          }
        ]
      }
    ]
  }
}
```

省略箇所：`$.query.pages[0].extlinks`：実応答50要素の先頭1要素のみ掲載。

`query.pages[].extlinks[]`に、指定記事が含む外部URLが返る。逆向きに、あるURLを含むページを探すには`list=exturlusage`を使う。

```sh
curl --get --silent --show-error --compressed \
  --user-agent 'WikipediaApiExample/1.0 (https://example.org/contact)' \
  'https://ja.wikipedia.org/w/api.php' \
  --data-urlencode 'action=query' \
  --data-urlencode 'list=exturlusage' \
  --data-urlencode 'euquery=d-nb.info/gnd/4079154-3' \
  --data-urlencode 'eunamespace=0' \
  --data-urlencode 'euprop=ids|title|url' \
  --data-urlencode 'eulimit=50' \
  --data-urlencode 'format=json' \
  --data-urlencode 'formatversion=2'
```

応答例（HTTP 200、JSON構造）：

```json
{
  "batchcomplete": true,
  "query": {
    "exturlusage": [
      {
        "pageid": 3957,
        "ns": 0,
        "title": "宇宙",
        "url": "https://d-nb.info/gnd/4079154-3"
      }
    ]
  }
}
```

省略箇所：なし。指定条件に一致する要素は1件であった。

`euquery`はプロトコルを除いた検索文字列である。結果は`query.exturlusage[]`に入り、`eucontinue`で継続する。URLの文字列一致をそのまま同一資料と解釈せず、scheme、hostの大小文字、既定port、末尾slash、tracking query、fragment等をクライアント側で正規化する。APIは外部URLの安全性や到達可能性を保証しないため、取得したURLを自動巡回しない。[API:Extlinks](https://www.mediawiki.org/wiki/API:Extlinks) [API:Exturlusage](https://www.mediawiki.org/wiki/API:Exturlusage)

### 4.19 座標・地理近傍：`prop=coordinates`、`list=geosearch`

```sh
curl --get --silent --show-error --compressed \
  --user-agent 'WikipediaApiExample/1.0 (https://example.org/contact)' \
  'https://ja.wikipedia.org/w/api.php' \
  --data-urlencode 'action=query' \
  --data-urlencode 'titles=東京駅' \
  --data-urlencode 'prop=coordinates' \
  --data-urlencode 'coprimary=primary' \
  --data-urlencode 'coprop=type|name|dim|country|region' \
  --data-urlencode 'colimit=50' \
  --data-urlencode 'format=json' \
  --data-urlencode 'formatversion=2'
```

応答例（HTTP 200、JSON構造）：

```json
{
  "batchcomplete": true,
  "query": {
    "pages": [
      {
        "pageid": 1608983,
        "ns": 0,
        "title": "東京駅",
        "coordinates": [
          {
            "lat": 35.68111111,
            "lon": 139.76666667,
            "primary": true,
            "type": "railwaystation",
            "name": "JR 東京駅",
            "dim": "1000",
            "country": "JP",
            "region": "13"
          }
        ]
      }
    ]
  }
}
```

省略箇所：なし。

`query.pages[].coordinates[]`に緯度、経度、primary属性等が返る。記事を中心とした近傍検索は次のように行う。

```sh
curl --get --silent --show-error --compressed \
  --user-agent 'WikipediaApiExample/1.0 (https://example.org/contact)' \
  'https://ja.wikipedia.org/w/api.php' \
  --data-urlencode 'action=query' \
  --data-urlencode 'list=geosearch' \
  --data-urlencode 'gspage=東京駅' \
  --data-urlencode 'gsradius=1000' \
  --data-urlencode 'gsnamespace=0' \
  --data-urlencode 'gsprimary=primary' \
  --data-urlencode 'gslimit=50' \
  --data-urlencode 'format=json' \
  --data-urlencode 'formatversion=2'
```

応答例（HTTP 200、JSON構造）：

```json
{
  "batchcomplete": true,
  "query": {
    "geosearch": [
      {
        "pageid": 4692817,
        "ns": 0,
        "title": "東京ステーションギャラリー",
        "lat": 35.68095833333333,
        "lon": 139.76730555555557,
        "dist": 60.2,
        "primary": true
      }
    ]
  }
}
```

省略箇所：`$.query.geosearch`：実応答50要素の先頭1要素のみ掲載。

`query.geosearch[]`に`pageid`、`title`、`lat`、`lon`、中心からの`dist`等が返る。座標を直接与える場合は`gscoord=<緯度>|<経度>`を使う。`gsradius`の単位はメートルで、Wikimedia上の通常範囲は10～10,000メートルである。地理距離は意味的関連度とは別の`geo`型の辺として保存し、同一点の複数記事、primary以外の座標、地球以外のglobeを区別する。[Extension:GeoData](https://www.mediawiki.org/wiki/Extension:GeoData) [API:Geosearch](https://www.mediawiki.org/wiki/API:Geosearch)

### 4.20 記事別閲覧数：Wikimedia Analytics API

Wikimedia Analytics APIはAction APIとは別のサービスであり、記事別pageviewの時系列は次のURL形式で取得する。

```text
https://wikimedia.org/api/rest_v1/metrics/pageviews/per-article/{project}/{access}/{agent}/{article}/{granularity}/{start}/{end}
```

日本語版Wikipediaの「宇宙」について、利用者由来と判定された全accessの日次閲覧数を取得する例：

```sh
curl --silent --show-error --compressed \
  --user-agent 'WikipediaApiExample/1.0 (https://example.org/contact)' \
  'https://wikimedia.org/api/rest_v1/metrics/pageviews/per-article/ja.wikipedia.org/all-access/user/%E5%AE%87%E5%AE%99/daily/2026090100/2026090700'
```

応答例（HTTP 200、JSON構造）：

```json
{
  "items": [
    {
      "project": "ja.wikipedia",
      "article": "宇宙",
      "granularity": "daily",
      "timestamp": "2026090100",
      "access": "all-access",
      "agent": "user",
      "views": 366
    },
    {
      "project": "ja.wikipedia",
      "article": "宇宙",
      "granularity": "daily",
      "timestamp": "2026090200",
      "access": "all-access",
      "agent": "user",
      "views": 347
    }
  ]
}
```

省略箇所：`$.items`：実応答7要素の先頭2要素のみ掲載。

応答の`items[]`には`project`、`article`、`granularity`、`timestamp`、`access`、`agent`、`views`が入る。URLの`project`パラメーターは`ja.wikipedia.org`であり、応答内の`project`は`ja.wikipedia`のように返り得るため、内部のwiki識別子はリクエスト条件と対応付けて正規化する。`access`は例では`all-access`、`agent=user`はspider・automatedとして分類された通信を除いた区分、`granularity`は`daily`または`monthly`である。日時はUTCとして扱う。記事別endpointのデータ提供期間は2015年7月1日以降である。[Wikimedia Analytics API：Page view analytics](https://doc.wikimedia.org/generated-data-platform/aqs/analytics-api/reference/page-views.html) [Page viewsの定義](https://doc.wikimedia.org/generated-data-platform/aqs/analytics-api/concepts/page-views.html)

記事名は空白をunderscoreへそろえ、path segmentとしてURLエンコードする。Action APIでリダイレクトを正規化してから閲覧数を取得する。pageview集計ではリダイレクトへの閲覧が転送先記事の閲覧として加算されないため、別名を含む需要を測る場合は既知のリダイレクトを別々に取得して、目的を明示したうえで合算する。

時系列では0件の日が応答から省略され得る。また、HTTP 404だけでは値が0なのかデータがまだロードされていないのかを区別できない場合がある。欠けた日を機械的に0へ補う前に、対象期間とデータ投入状況を確認する。[Analytics API troubleshooting](https://doc.wikimedia.org/generated-data-platform/aqs/analytics-api/documentation/troubleshooting.html)

閲覧数は人気度・時事性の特徴であり、意味的関連度ではない。関連記事探索では`log1p(views)`、期間中央値、取得期間を別属性として保存し、リンクや本文類似のスコアを置き換えない。

### 4.21 増分更新通知：Wikimedia EventStreams

EventStreamsはServer-Sent Events（SSE）で連続イベントを配信する。取得済み記事の更新検知には`recentchange`または`revision-create`を使う。

```text
https://stream.wikimedia.org/v2/stream/recentchange
https://stream.wikimedia.org/v2/stream/revision-create
```

ブラウザで日本語版Wikipediaの変更だけを受け取る最小例：

```js
const stream = new EventSource(
  "https://stream.wikimedia.org/v2/stream/recentchange"
);

stream.onmessage = (event) => {
  const change = JSON.parse(event.data);
  if (change.meta?.domain === "canary") return;
  if (change.server_name !== "ja.wikipedia.org") return;
  console.log(change.title);
};
```

EventStreamsはwikiやpageidによるserver-side filterを提供しないため、`server_name`または`wiki`、対象pageid・titleを利用側で絞り込む。監視用のcanary eventも配信されるので、`meta.domain === "canary"`は通常の更新として処理しない。[EventStreams](https://wikitech.wikimedia.org/wiki/EventStreams)

SSEの`id`をcheckpointし、再接続時に`Last-Event-ID`として渡すと中断位置からの再開に使える。`since=<ISO 8601時刻>`による履歴開始も可能だが、履歴保持はstream設定に依存し、一般に7～31日程度で無期限ではない。複数datacenterではtimestampに基づく再開となり、exactly-onceの処理境界としては使えないため、イベントと版IDで冪等化する。接続はサーバー側で約15分ごとに終了し得るため、自動再接続を実装する。

イベントは「ページまたは版が変わった」という通知であり、本文リンクの追加・削除差分そのものではない。対象ノードの更新を検知したらAction APIまたはREST APIで最新版IDを確認し、保存済み版IDと異なる場合だけ再取得・再解析する。移動、削除、復元も考慮し、イベントIDと版IDの両方を監査ログへ残す。

### 4.22 Wikidata項目・statement・sitelink：`wbgetentities`

日本語版Wikipediaの記事から`pageprops.wikibase_item`でQIDを得た後、Wikidataのendpointへ問い合わせる。

```sh
curl --get --silent --show-error --compressed \
  --user-agent 'WikipediaApiExample/1.0 (https://example.org/contact)' \
  'https://www.wikidata.org/w/api.php' \
  --data-urlencode 'action=wbgetentities' \
  --data-urlencode 'ids=Q1' \
  --data-urlencode 'props=labels|claims|sitelinks/urls' \
  --data-urlencode 'languages=ja' \
  --data-urlencode 'languagefallback=1' \
  --data-urlencode 'sitefilter=jawiki|enwiki' \
  --data-urlencode 'format=json' \
  --data-urlencode 'formatversion=2' \
  --data-urlencode 'origin=*'
```

応答例（HTTP 200、JSON構造）：

```json
{
  "entities": {
    "Q1": {
      "type": "item",
      "id": "Q1",
      "labels": {
        "ja": {
          "language": "ja",
          "value": "宇宙"
        }
      },
      "claims": {
        "P31": [
          {
            "mainsnak": {
              "snaktype": "value",
              "property": "P31",
              "datavalue": {
                "value": {
                  "entity-type": "item",
                  "numeric-id": 36906466,
                  "id": "Q36906466"
                },
                "type": "wikibase-entityid"
              },
              "datatype": "wikibase-item"
            },
            "type": "statement",
            "id": "Q1$8983b0ea-4a9c-0902-c0db-785db33f767c",
            "rank": "normal"
          }
        ]
      },
      "sitelinks": {
        "enwiki": {
          "site": "enwiki",
          "title": "Universe",
          "badges": [
            "Q17437798"
          ],
          "url": "https://en.wikipedia.org/wiki/Universe"
        },
        "jawiki": {
          "site": "jawiki",
          "title": "宇宙",
          "badges": [],
          "url": "https://ja.wikipedia.org/wiki/%E5%AE%87%E5%AE%99"
        }
      }
    }
  },
  "success": 1
}
```

省略箇所：`$.entities.Q1.claims`：実応答の多数のpropertyから`P31`だけを掲載。`$.entities.Q1.claims.P31`：先頭statementだけを掲載。statementの`mainsnak.hash`を省略。

`entities.Q1.labels`にラベル、`entities.Q1.claims`にstatement、`entities.Q1.sitelinks`に各wikiの記事タイトル・badge・URLが返る。QIDの代わりに`sites=jawiki&titles=宇宙`を指定して、sitelinkから項目を逆引きすることもできる。[Wikibase API](https://www.mediawiki.org/wiki/Wikibase/API) [API:Presenting Wikidata knowledge](https://www.mediawiki.org/wiki/API:Presenting_Wikidata_knowledge)

`props=claims`は項目のclaim全体を返し、取得するプロパティだけをサーバー側で指定する機能ではない。利用側で`instance of`（P31）、`subclass of`（P279）、`part of`（P361）等のallowlistを適用する。各statementについて少なくとも`rank`、`mainsnak.snaktype`、`mainsnak.datavalue.type`、値、qualifier、referenceの有無を保存し、値を単純なQIDだと仮定しない。`novalue`・`somevalue`や不明なdata typeは明示的に扱う。

Wikidata QIDは概念層の識別子であり、Wikipediaの`pageid`とは役割が異なる。一つのQIDに各wikiのsitelinkを接続し、Wikipedia記事ノードは`wiki + pageid`で識別する。statementから得たQIDに表示名が必要な場合は、QIDをまとめて追加の`wbgetentities&props=labels`で解決し、キャッシュする。

### 4.23 制約付き概念探索：Wikidata Query Service

複数QIDについて許可した関係だけをまとめて取得する場合、Wikidata Query Service（WDQS）のSPARQL endpointを利用できる。

```sh
curl --get --silent --show-error --compressed \
  --user-agent 'WikipediaApiExample/1.0 (https://example.org/contact)' \
  --header 'Accept: application/sparql-results+json' \
  'https://query.wikidata.org/sparql' \
  --data-urlencode 'query=
PREFIX wd: <http://www.wikidata.org/entity/>
PREFIX wdt: <http://www.wikidata.org/prop/direct/>
SELECT ?source ?property ?target WHERE {
  VALUES ?source { wd:Q1 }
  VALUES ?property { wdt:P31 wdt:P279 wdt:P361 }
  ?source ?property ?target .
}'
```

応答例（HTTP 200、JSON構造）：

```json
{
  "head": {
    "vars": [
      "source",
      "property",
      "target"
    ]
  },
  "results": {
    "bindings": [
      {
        "source": {
          "type": "uri",
          "value": "http://www.wikidata.org/entity/Q1"
        },
        "property": {
          "type": "uri",
          "value": "http://www.wikidata.org/prop/direct/P31"
        },
        "target": {
          "type": "uri",
          "value": "http://www.wikidata.org/entity/Q36906466"
        }
      },
      {
        "source": {
          "type": "uri",
          "value": "http://www.wikidata.org/entity/Q1"
        },
        "property": {
          "type": "uri",
          "value": "http://www.wikidata.org/prop/direct/P31"
        },
        "target": {
          "type": "uri",
          "value": "http://www.wikidata.org/entity/Q67518978"
        }
      }
    ]
  }
}
```

省略箇所：なし。指定した3プロパティに一致するbindingは2件であった。

応答の`results.bindings[]`に変数ごとのURIまたはliteralが返る。この例の`wdt:`はrankを反映したtruthyな直接propertyであり、全statement、qualifier、referenceを返す表現ではない。それらが必要な検証では`p:`でstatement nodeを取得し、`ps:`、`pq:`、`prov:wasDerivedFrom`等を明示的にたどる。[WDQS User Manual](https://www.mediawiki.org/wiki/Wikidata_Query_Service/User_Manual) [Wikidata SPARQL tutorial：Qualifiers](https://www.wikidata.org/wiki/Wikidata:SPARQL_tutorial#Qualifiers)

検証では開始QIDを`VALUES`で限定し、property allowlist、`LIMIT`、小さなページ単位、クライアント側timeout、結果cacheを使う。無制約のproperty pathや巨大な集計を公開endpointへ送らない。WDQSはAction APIと異なる可用性・timeout特性を持つため、失敗時は第4.22節の`wbgetentities`で一項目ずつ取得できる設計にする。[Wikidata Query Service](https://www.wikidata.org/wiki/Wikidata:SPARQL_query_service) [WDQS endpoint比較](https://www.wikidata.org/wiki/Wikidata:SPARQL_query_service/Alternative_endpoints)

## 5. 非機能仕様・利用制限

### 5.1 呼び出しレート

グローバルなレート制限は、Action API・REST APIを含むWikimediaのサイトを横断して適用される。次表は2026年に導入された制限の公表値であり、実験・変更の対象と明記されているため、適用値は参照先の最新仕様に従う。

| クライアント区分 | リクエスト数上限 |
| --- | --- |
| IPアドレス以外の識別情報がないリクエスト | 10件/分 |
| 未認証ユーザーのWebブラウザ | 200件/分 |
| 適切なUser-Agentを持つ未認証bot | 200件/分 |
| 新規・編集実績の少ない認証ユーザー | 200件/分 |
| 編集実績のある認証ユーザー | 2,000件/分 |

これはモジュール別・言語版別の割当ではない。認証済みbotなどの免除区分もあるが、運用上の制限は別途適用される。ログインやクライアント名の指定だけで上位枠を保証するものではない。[Wikimedia APIs/Rate limits](https://www.mediawiki.org/wiki/Wikimedia_APIs/Rate_limits)

### 5.2 同時実行数・短時間の呼び出し

Robot policyのAction API向け条件は次のとおり。毎分の制限と併せて満たす必要がある。

| 条件 | 未認証 | 認証済み |
| --- | --- | --- |
| 同時リクエスト数 | 1件 | 最大3件 |
| 秒あたりの呼び出し | 5件未満 | 最大10件 |

応答に1秒を超える処理を要した場合は、次のリクエストまで5秒待機する。バッチ処理と圧縮を利用する。また、HTML取得はAction APIよりWebページまたはREST APIを優先する方針が示されている。第4.6節は解析機能の呼び出し例であり、継続的なHTML大量取得を推奨するものではない。[Robot policy：Action API rules](https://wikitech.wikimedia.org/wiki/Robot_policy#Action_API_rules)

Robot policyのREST API向け条件では、未認証時は同時実行を最大3件、全体を毎秒5件未満に保ち、認証時でも毎秒10件までとする。大量取得では最新版のcache可能なrouteを優先し、特定版HTMLを全履歴について巡回しない。[Robot policy：REST API rules](https://wikitech.wikimedia.org/wiki/Robot_policy#REST_API_rules)

### 5.3 制限・負荷超過時の応答

| 応答 | 意味・処理 |
| --- | --- |
| HTTP 429 | レート超過。`Retry-After`に従って待機する |
| HTTP 503 | バックエンド過負荷等。`Retry-After`があれば従う |
| 429/503で`Retry-After`なし | 少なくとも5秒待つか、指数バックオフを行う |
| HTTP 200かつ`error.code=maxlag` | レプリカ遅延による処理保留。成功データとして扱わない |

429/503の本文が必ずAction APIのJSON形式とは限らないため、HTTPステータスを先に検査する。[レート制限のエラー仕様](https://www.mediawiki.org/wiki/Wikimedia_APIs/Rate_limits#Errors)

非対話処理では`maxlag=5`が推奨される。単位は秒であり、レプリカの遅延許容値を表す。通信タイムアウトやリクエスト数の制限ではない。`maxlag`エラー時は最低5秒待ち、`Retry-After`がより長い場合はそちらに従う。[Manual:Maxlag parameter](https://www.mediawiki.org/wiki/Manual:Maxlag_parameter)

### 5.4 1回の取得件数・入力上限

以下は一般クライアント向けの値。これらの件数上限は呼び出しレートとは別である。

| 対象 | 上限・範囲 | 参照 |
| --- | --- | --- |
| `list=search`の`srlimit` | 1～500、既定10 | [Search](https://www.mediawiki.org/wiki/API:Search) |
| `prop=links`の`pllimit` | 1～500、既定10 | [Links](https://www.mediawiki.org/wiki/API:Links) |
| `list=backlinks`の`bllimit` | 1～500、既定10。`blredirect`併用時は別条件あり | [Backlinks](https://www.mediawiki.org/wiki/API:Backlinks) |
| `prop=categories`の`cllimit` | 1～500、既定10 | [Categories](https://www.mediawiki.org/wiki/API:Categories) |
| `list=categorymembers`の`cmlimit` | 1～500、既定10 | [Categorymembers](https://www.mediawiki.org/wiki/API:Categorymembers) |
| `generator=links`の`gpllimit` | 1～500、既定10 | [Links](https://www.mediawiki.org/wiki/API:Links) |
| `prop=revisions`の`rvlimit` | 1～500。`rvprop=content`等を含む場合は最大50 | [Revisions](https://www.mediawiki.org/wiki/API:Revisions) |
| `prop=langlinks`の`lllimit` | 1～500、既定10 | [Langlinks](https://www.mediawiki.org/wiki/API:Langlinks) |
| `prop=templates`の`tllimit` | 1～500、既定10 | [Templates](https://www.mediawiki.org/wiki/API:Templates) |
| `list=embeddedin`の`eilimit` | 1～500、既定10 | [Embeddedin](https://www.mediawiki.org/wiki/API:Embeddedin) |
| `prop=extlinks`の`ellimit` | 1～500、既定10 | [Extlinks](https://www.mediawiki.org/wiki/API:Extlinks) |
| `list=exturlusage`の`eulimit` | 1～500、既定10 | [Exturlusage](https://www.mediawiki.org/wiki/API:Exturlusage) |
| `list=geosearch`の`gslimit` | 一般クライアントは最大500、既定10 | [GeoData](https://www.mediawiki.org/wiki/Extension:GeoData#API) |
| `query`の`titles`・`pageids` | 各最大50値。`apihighlimits`権限では500値 | [Query](https://www.mediawiki.org/wiki/API:Query) |
| `extracts`の`exlimit` | 最大20。複数抜粋には`exintro=1`が必要 | [TextExtracts](https://www.mediawiki.org/wiki/Extension:TextExtracts#API) |
| `extracts`の`exchars` | 指定値1～1,200。実際の文字数は多少超過し得る | [TextExtracts](https://www.mediawiki.org/wiki/Extension:TextExtracts#API) |

リンクや検索結果は継続取得できるが、本文HTMLを同じ仕組みで件数分割するものではない。本文サイズは記事に依存する。参照先の仕様は、応答全体の一律のバイト上限を規定していない。

### 5.5 キャッシュ・可用性・変更

GET、複数対象の一括取得、クライアント側キャッシュが推奨される。Action APIのGETでは`maxage=<秒>`でブラウザ向け`max-age`、`smaxage=<秒>`で共有cache向け`s-maxage`を指定できるが、エラーはcacheされない。cache hitにはURL全体の一致が必要なため、複数タイトルは並べ替え・重複除去し、パラメーター順を含む正規化したquery keyを使う。[API:Caching data](https://www.mediawiki.org/wiki/API:Caching_data)

REST APIでは、第4.15節のとおり`ETag`／`If-None-Match`または`Last-Modified`／`If-Modified-Since`による条件付きGETを利用する。raw応答cacheと、リンク・カテゴリ・本文等を正規化した派生cacheを分け、派生結果には元のrevision IDとquery条件を付ける。キャッシュ有効期間の一律の保証はなく、HTTPキャッシュ指示と記事更新を考慮する。[API Etiquette](https://www.mediawiki.org/wiki/API:Etiquette)

公開APIについて、応答時間・稼働率の数値保証は参照資料に示されていない。接続タイムアウトやキャッシュ保持時間はクライアント側の設定である。APIの変更・廃止や利用制限があり得るため、提供機能の永続性は保証として扱わない。[API Usage Guidelines](https://foundation.wikimedia.org/wiki/Policy:Wikimedia_Foundation_API_Usage_Guidelines)

### 5.6 Action API以外のサービス

- Analytics APIも識別可能な`User-Agent`を必須とし、各リクエストの完了を待って次を送る。HTTP 429では`Retry-After`に従う。[Analytics API access policy](https://doc.wikimedia.org/generated-data-platform/aqs/analytics-api/documentation/access-policy.html)
- EventStreamsは短いpollを反復せず、一つのSSE接続を再接続しながら利用する。server-side filteringはないため受信後に絞り込むが、不要な複数streamを同時購読しない。
- Wikidata Query ServiceはAction APIと同じ件数上限ではない。短い制約付きqueryだけを使い、timeout・一時停止を通常の失敗として扱う。大量または全件の処理にはWikidata dumpを検討する。
- いずれのサービスでも、HTTP成功だけでなく応答形式、欠測、遅延、版または集計期間を検証し、異なるサービスの値を同一時点のsnapshotだと仮定しない。

## 6. コンテンツの再利用条件

Wikipediaの文章を表示、保存、再配布する場合は、該当ページのライセンスと帰属表示の要件に従う必要がある。Wikimediaの利用規約は、再利用時に原記事または著者表示に相当するページへのリンクなどによる帰属表示を求めている。文章の変更・追加を再配布する場合は、ライセンス表示と改変の明示も必要である。[利用規約第7節](https://foundation.wikimedia.org/wiki/Policy:Terms_of_Use#7._Licensing_of_Content)

画像・音声などの非テキストメディアには個別のライセンスが適用され得る。記事本文に適用される条件だけから、画像の再利用条件を判断してはならない。

Analytics APIが提供する集計データはCC0 1.0とされている。Wikidataの構造化データにもWikipedia本文とは別のライセンス条件がある。複数サービスのデータを結合する場合は、出典URL、取得日時、対象wiki、版IDまたは集計期間、各データセットのライセンスを別々に記録する。[Analytics API access policy](https://doc.wikimedia.org/generated-data-platform/aqs/analytics-api/documentation/access-policy.html) [Wikidata:Data access](https://www.wikidata.org/wiki/Wikidata:Data_access)

## 補足：記事探索・構造化・レンダリングでの利用時の留意事項

この節はWikipedia APIそのものの仕様ではなく、記事データを利用する処理で必要となる制御とデータ管理を補足する。特定のアプリケーションを実装することは前提としない。ここでは、記事検索から始まり、利用者または呼出元による記事選択、関連記事の取得、取得した記事を起点とする再帰探索へ進む同期的な処理系列を想定する。探索とは別の契機から非同期に停止指示が届く場合、および取得済み記事を構造化してレンダリングする場合も扱う。

### 想定する処理系列

| 段階 | 主な処理 | 利用するAPI・保存する状態 |
| --- | --- | --- |
| 記事検索 | 検索語から候補記事を列挙する | `list=search`、検索語、順位、取得日時 |
| 記事選択 | 選択されたタイトルを正規化し、リダイレクトと曖昧さを確認する | `prop=info|pageprops`、`pageid`、正規タイトル、版ID |
| 関連記事探索 | 出リンク、被リンク、カテゴリ、言語間リンク、構造化関係等から次の候補を取得する | Links、Backlinks、Categories、Wikidata等と関係の取得根拠 |
| 再帰探索ループ | 未取得候補をキューから取り出し、深さ・件数・時間等の予算内で同じ処理を反復する | 訪問済み集合、frontier、深さ、`continue`、取得版、打切り理由 |
| 構造化・レンダリング | 取得済みの本文、属性、関係を正規化し、用途に応じた表現へ変換する | raw応答、正規化済みレコード、出典URL、版ID、レンダリング対象snapshot |

検索から再帰探索までの実行系列は同期的に進められるが、停止指示は別の制御経路から非同期に到着し得る。停止要求を安全に処理するため、探索runごとに一意なIDと状態を持ち、少なくとも`running`、`stop-requested`、`stopped`、`completed`、`failed`を区別する。停止要求後は新しいAPI呼び出しとfrontier追加を開始せず、実行中のHTTP要求を中断できる場合は中断する。中断できない応答が後から届いた場合は、run IDと状態を照合して採用可否を決める。

停止は完了や失敗と同義ではない。停止時には受領済みのraw応答、正規化済み記事、訪問済みID、未処理frontier、処理中だったquery条件と継続情報を整合する単位で保存する。ただし`continue`値は元のquery条件にだけ有効な不透明値であり、停止から長時間後に無条件で再利用せず、第3.3節のsnapshot上の制約も記録する。

構造化とレンダリングは探索制御から分離する。レンダリング処理へ渡す入力は、対象run、対象wiki、記事の版ID、取得日時を固定したsnapshotとし、画面描画を契機に暗黙の再帰探索を開始しない。探索中の途中結果を表示する場合も、未確定・停止済み・完了済みの状態を区別し、後着応答で既に停止した表示内容を意図せず更新しない。

### 構造化された出来事情報の利用

記事本文、表、年表テンプレートは記事ごとに記法が異なるため、出来事の年月日や関係者を本文から一律の規則で抽出する対象にはしない。記事の技術的メタデータに含まれる`pageprops.wikibase_item`を、対応するWikidata項目IDとして利用する。

1. `action=query&prop=pageprops`で記事の`pageprops.wikibase_item`を取得する。
2. 項目IDがある場合は、Wikidataの`https://www.wikidata.org/w/api.php`へ`action=wbgetentities&ids=<項目ID>&props=labels|claims&languages=ja&format=json`を送る。
3. `entities[項目ID].claims`から、あらかじめ用途を定めたプロパティだけを機械的に読み取る。項目や値の参照はQIDのまま保持する。値に含まれるQIDの表示名は、QIDをまとめて指定した追加の`wbgetentities`リクエストで`labels`を取得して解決する。

| 解釈対象 | Wikidataプロパティ | 値の扱い |
| --- | --- | --- |
| 単一時点の日時 | `point in time`（P585） | 時刻・精度・暦モデルを保持する |
| 期間の開始・終了 | `start time`（P580）、`end time`（P582） | 両方がある場合に期間として扱う |
| 関係者 | `participant`（P710） | 人物・集団・組織のQIDを複数値として保持する |
| 場所 | `location`（P276） | 場所のQIDとして保持する |
| 行政区域 | `located in the administrative territorial entity`（P131） | 地理的な補助情報として保持する |

例えば本能寺の変に対応するWikidata項目は`Q169598`であり、P585、P710、P276、P131の各ステートメントを持つ。このようにプロパティIDとデータ型に基づく読み取りは決定的である。一方で、個々の項目にどのステートメントがあるか、複数値が網羅的か、日付の精度が年・月・日まであるかは保証されない。[Wikibase API](https://www.mediawiki.org/wiki/Wikibase/API) [Wikidataのデータモデル](https://www.wikidata.org/wiki/Help:Data_model)

取得側は、対象プロパティがない場合を`unknown`として扱い、本文・カテゴリ・名称から補完推定しない。値が複数ある場合は任意の1件へ縮約せず、すべてを保持する。出来事の時系列を構成する場合も、P585またはP580/P582を持つ値だけを時刻順に並べ、日付のない関係は時系列順であると解釈しない。

### ClickstreamはAPIではなく月次データセット

Wikipedia Clickstreamは、記事間の実際の遷移回数を補助的な関係として利用できる月次公開データであり、オンラインAPIではない。提供対象の言語版と年月について、`https://dumps.wikimedia.org/other/clickstream/`から圧縮TSVを取得する。[Wikipedia Clickstream](https://meta.wikimedia.org/wiki/Research:Wikipedia_clickstream)

現在の基本列は`prev`、`curr`、`type`、`n`である。`type=link`は、参照元と遷移先が記事で、参照元記事から遷移先へのリンクが存在した組を表す。`n`はその月の観測回数である。公開データでは10回以下の`(referrer, resource)`組が除外され、redirectは通常解決されている。このため、存在しない行を遷移0回と解釈してはならず、低頻度遷移の裾は打ち切られている。

Clickstreamは初期探索やリアルタイム更新には使わず、Action APIで作った`wiki + 正規化title/pageid`の記事レコードへ月次の`click`型関係として後付けする。dumpの年月、言語版、`type`、元のタイトル、正規化後のpageidを保存し、Pageviewsと組み合わせる場合も集計期間を一致させる。

### カテゴリ・記事識別子の正規化・再帰探索

- カテゴリを利用すると「記事 → 所属カテゴリ → 同じカテゴリの記事」という経路を作れる。記事間リンク、カテゴリ所属、カテゴリ間の親子関係は、それぞれ異なる関係として扱う。
- カテゴリをたどる処理では、複数経路からの重複や循環に備えて訪問済みカテゴリを記録する。階層全体は自動取得されないため、追加呼び出しにも第5節のレート制限が適用される。これらは分類による探索を実装するための補足である。

- `prop=links`が返すのはタイトル中心のリンク候補である。記事レコードを安定して統合するには、候補タイトルを`titles=...&prop=info&redirects=1`で照会し、解決後の`pageid`を識別子として利用する。
- タイトルの正規化・リダイレクト・不存在は、応答の`normalized`、`redirects`、`missing`、`invalid`で区別する。応答配列の順序だけで元タイトルと解決先を対応付けない。
- 曖昧さ回避ページは`pageprops.disambiguation`で検知できる。APIは意味を一つに決定しないため、利用側が自動選択せず候補として提示する設計が必要である。
- `prop=links`の返却順は関連度ではない。意味的な推薦や本文での出現順を必要とする場合、`action=parse`のHTMLを追加解析することになる。ただし、テンプレート等を除いた「本文だけ」の完全な判定をAPIが保証するわけではない。
- HTMLを表示・解析する場合、APIが返すHTMLをアプリケーションのDOMへ無加工で挿入しない。サニタイズとリンク先検証が必要である。
- リンク展開を自動で再帰取得するとリクエスト数が急増する。ユーザーが選択した記事を中心に取得し、継続取得とキャッシュを利用し、負荷応答時には再試行を遅延する必要がある。
