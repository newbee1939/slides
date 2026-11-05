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

## 画像（SVG など）を貼る

`index.md` と同じディレクトリに置き、相対パスで参照する。

```markdown
![w:900](architecture.svg)
```

- `npm run build` は `.md` 以外のファイルも `dist/` にコピーする（Marp CLI 自体は画像をコピーしないため）
- PDF の変換は `--allow-local-files` を付けないとローカルの画像を読めないので、`npm run pdf` に付けてある

## プレゼンのTips

- いかに認知の負荷（聞き手が理解するのに使う頭の容量）を下げるか
- 徹底的に相手の視点に立つ
