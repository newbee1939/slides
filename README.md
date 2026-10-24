# slides

登壇資料（[Marp](https://marp.app/)）。main に入ると https://newbee1939.github.io/slides/<ディレクトリ名>/ に公開される。

| 日付 | タイトル | イベント | Speaker Deck |
| --- | --- | --- | --- |

## 使い方

```sh
npm install
cp -r talks/2026-01-01-sample talks/YYYY-MM-DD-slug  # 新しい登壇
npm run build                                       # dist/ に HTML
npm run pdf -- talks/YYYY-MM-DD-slug/index.md -o YYYY-MM-DD-slug.pdf  # Speaker Deck 用（要 Chrome）
```
