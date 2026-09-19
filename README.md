# Wikiwalker

Wikipedia 宇宙の中を「記事星」を辿って散策するグラフィカルブラウザー。

記事を 3D 空間に星として配置し、意味的に近い記事や本文からの出リンクを辿りながら知識のつながりを眺められます。大量の出リンクは共有カテゴリごとの「星団」に集約して表示します。

![スクリーンショット](README.screenshot.png)

動作確認用のサンプルページを [こちら](https://ura14h.github.io/Wikiwalker/) に用意しています。お試しください。

## 使い方

`index.html` をブラウザーで開くだけです。ビルドもローカルサーバーも不要です（外部ライブラリと Wikipedia API の取得にインターネット接続が必要）。

- 検索欄に記事名を入力して候補から選ぶと、その記事を起点に探索が始まります
- 記事星や星団をクリックすると選択・展開できます
- ドラッグで回転、ホイールでズーム、Prev / Next で履歴を移動します
- 言語ボタンから Wikipedia の言語版を切り替えられます

## 構成

| パス | 内容 |
| --- | --- |
| `index.html` | アプリ本体（単一 HTML、バニラ JavaScript） |
| `docs/` | コンセプト、ストーリーボード、設計仕様書、参考資料 |
| `proto/` | 開発過程のプロトタイプと AI エージェント間のチャットログ |

外部ライブラリは [three.js](https://threejs.org/)、[3d-force-graph](https://github.com/vasturiano/3d-force-graph)、[three-spritetext](https://github.com/vasturiano/three-spritetext) を esm.sh から読み込んでいます。

## 表示するデータについて

記事の内容は [Wikipedia](https://www.wikipedia.org/) の API から取得しています。記事本文は [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/) で提供されており、Wikipedia はウィキメディア財団の登録商標です。本アプリはウィキメディア財団とは無関係です。

## ライセンス

MIT License。詳細は [LICENSE](LICENSE) を参照してください。
