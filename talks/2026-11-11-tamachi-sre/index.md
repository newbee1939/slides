---
marp: true
theme: default
style: |
  :root {
    --sky: #8AC2DD;
    --sky-deep: #3D7FA0;
    --sky-soft: #D6EAF4;
    --sky-pale: #F1F8FC;
    --ink: #33475B;
  }
  section {
    background: var(--sky-pale);
    color: var(--ink);
    font-family: "Hiragino Maru Gothic ProN", "Zen Maru Gothic", sans-serif;
    font-weight: 500;
    font-size: 28px;
    line-height: 1.6;
    padding: 150px 80px 48px;
    align-content: start !important;
  }
  h2 {
    position: absolute;
    top: 52px;
    left: 80px;
    right: 80px;
    margin: 0;
    padding-left: 24px;
    color: var(--sky-deep);
    font-size: 42px;
    font-weight: 700;
  }
  h2::before {
    content: "";
    position: absolute;
    left: 0;
    top: 10%;
    height: 80%;
    width: 10px;
    border-radius: 5px;
    background: var(--sky);
  }
  strong { color: var(--sky-deep); }
  a { color: var(--sky-deep); }
  code { background: var(--sky-soft); color: var(--sky-deep); border-radius: 8px; padding: 2px 8px; }
  ul ul { font-size: 0.9em; }
  li::marker { color: var(--sky); }
  blockquote {
    background: var(--sky-pale);
    border: none;
    border-radius: 20px;
    color: var(--ink);
    padding: 16px 28px;
    margin: 16px 0;
  }
  blockquote::before, blockquote::after { content: none; }
  section.lead {
    background:
      radial-gradient(circle at 12% 18%, rgba(255,255,255,0.45) 0 140px, transparent 141px),
      radial-gradient(circle at 90% 85%, rgba(255,255,255,0.35) 0 200px, transparent 201px),
      linear-gradient(135deg, var(--sky) 0%, var(--sky-soft) 100%);
    padding: 48px 80px;
    align-content: center !important;
    text-align: center;
  }
  section.lead::before { content: none; }
  section.lead h1 { color: #FFFFFF; font-size: 52px; font-weight: 700; line-height: 1.5; text-shadow: 0 2px 8px rgba(61,127,160,0.35); }
  section.lead p { color: var(--ink); font-size: 30px; }
  section.profile {
    display: grid;
    grid-template-columns: 1fr 300px;
    column-gap: 48px;
    align-items: start;
  }
  section.profile ul { grid-column: 1; grid-row: 1; margin: 0; }
  section.profile p { grid-column: 2; grid-row: 1; margin: 0; }
  section.profile img { border-radius: 50%; border: 8px solid var(--sky-soft); }
  section.figure { padding-top: 130px; }
  section.figure p { text-align: center; margin: 4px 0; font-size: 24px; }
  section.figure img { border-radius: 16px; }
---

<!-- _class: lead -->

# Cloudflare MCP Server Portal × Tunnel × Cloud Run で社内 MCP 基盤を構築した話

2026-11-11 Tamachi.sre#7

---

<!-- _class: profile -->

## 自己紹介

![w:280](h6o.jpeg)

- hide
    - @newbee1939
- 開発エンジニア -> SRE
- 趣味
    - サッカー（長年）
    - ビリヤニ（最近）

---

## 今日のゴール

- 社内 MCP 基盤を構築するときの選択肢を 1 つ知る
- その構成のポイントや注意点を知る

---

## 背景

- ビジネス側のメンバー（非エンジニア）も、**社内のデータを Claude から利用したい**という要望があった
  - 例: AIに「この商材で過去に似た施策をやったことがあるか調べて」と頼むだけで、社内の施策データベースを検索できる
- 社内 API はすでにあるが、ビジネス側のメンバーが直接使うのは現実的ではない
  - API ごとに URL や認証方式が違う
  - 利用者一人ひとりに API キーを配るのはセキュリティ的に危ない
- そこで、API の機能を MCP で提供することにした

---

## アーキテクチャの制約

- Claude アプリからの通信は、社内ではなく Anthropic の IP 範囲（`160.79.104.0/21`, `153.61.208.0/20`）から来る
- MCP へのアクセスを社内 IP アドレスで制限するのは難しい
- IP 制限ではなく、次の方針を取った

> 入口（MCP Server Portal）だけをインターネットに公開し、リクエストのたびに認証・認可を行う。MCP サーバー本体はインターネットから届かない場所に置く（Zero Trust の考え方）

---

<!-- _class: figure -->

## アーキテクチャ

![w:1500](architecture.svg)

---

ポイントを話す前に、前提となる知識を説明します

---

## Cloudflare MCP Server Portal とは

- 複数の MCP サーバーを 1 つの URL にまとめ、Cloudflare Access（誰を通すかを判定するアクセス制御機能）の認証・認可をかけて提供する Cloudflare の機能

---

## Cloudflare Tunnel とは

- サーバー側から Cloudflare へ暗号化された通信経路（Tunnel）を張り、**グローバル IP やポート開放なし**でサービスを届けられる仕組み
- 主なつなぎ方は 2 つ
  - Published application: インターネットに公開するホスト名を作り、Tunnel につなぐ
  - Private hostname: 公開せず、Cloudflare Gateway を通る通信だけが届く（**今回はこちら**）

---

<!-- _class: figure -->

## アーキテクチャ（再掲）

![w:1500](architecture.svg)

---

## アーキテクチャのポイント

- なるべく「閉じる」
- 認証・認可は Cloudflare に寄せる

---

## なるべく「閉じる」

Cloud Run の ingress は `internal`

---

## なるべく「閉じる」

Cloudflare Tunnel で、MCP サーバーにインターネット側の入口を作らない

---

## なるべく「閉じる」

- Private hostname で、Cloudflare Gateway を通る通信だけが届くようにする
  - Portal は Gateway を経由して Tunnel の Private hostname に届く（2026 年 9 月に追加された機能。[Private MCP server support for MCP server portals](https://developers.cloudflare.com/changelog/post/2026-09-22-private-mcp-servers/)）

---

## 認証・認可は Cloudflare に寄せる

- ログインと認可は Cloudflare Access と IdP（社員のログインを担うサービス）に任せる
  - MCP サーバーには認証・認可を実装しない
- 「論理 MCP」: 同じ MCP サーバーを Portal に 2 件登録し、許可するグループと有効にするツールを分ける
  - MCP サーバーには手を入れずに、人によって使えるツールを変えられる

```
mcp-a : read_xxx だけ有効             / 許可: 閲覧グループ
mcp-b : read_xxx と write_xxx を有効  / 許可: 編集グループ
```

---

## ぶつかった壁と解決策

- WARP 端末（通信を Gateway に送るアプリを入れた社内 PC）から Portal を迂回して MCP サーバーに届いてしまった
  - 原因: Private hostname は「Gateway 経由なら届く」であって「Portal からしか届かない」ではなかった
  - 対処: Gateway の Traffic Source（通信の入口）が Portal 以外なら、MCP サーバー宛てを Block

---

## まとめ

- Cloudflare MCP Server Portal を使えば、MCP 基盤を簡単に作れる
- Portal と Cloudflare Tunnel を組み合わせると、セキュアな構成にできる
- ただし MCP サーバー宛ての通信を Portal 経由だけに制限しないと、WARP など Gateway を経由する通信から MCP サーバーに直接届いてしまうので注意
