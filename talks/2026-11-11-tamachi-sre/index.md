---
marp: true
theme: default
style: |
  @import url('https://fonts.googleapis.com/css2?family=BIZ+UDPGothic:wght@400;700&display=swap');
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
    font-family: "BIZ UDPGothic", "Hiragino Sans", sans-serif;
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
    color: var(--ink);
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
    background: var(--sky-pale);
    padding: 48px 80px;
    align-content: center !important;
    text-align: center;
  }
  section.lead::before { content: none; }
  section.lead h1 { color: var(--ink); font-size: 52px; font-weight: 700; line-height: 1.5; }
  section.lead p { color: var(--ink); font-size: 30px; }
  section.profile {
    display: grid;
    grid-template-columns: 1fr 420px;
    column-gap: 48px;
    align-items: start;
  }
  section.profile ul { grid-column: 1; grid-row: 1; margin: 0; }
  section.profile p { grid-column: 2; grid-row: 1; margin: 0; }
  section.profile img { border-radius: 50%; border: 8px solid var(--sky-soft); }
  section.figure { padding-top: 130px; display: flex !important; flex-direction: column; }
  section.figure p { margin: 4px 0; }
  section.figure p:has(> img) { flex: 1 1 0; min-height: 0; margin-top: 40px; text-align: center; }
  section.figure img { width: 100%; height: 100%; object-fit: contain; }
  section.full { padding: 120px 40px 24px; }
  section.full p:has(> img) { margin-top: 0; }
  section.end { background: #222222; }
---

<!-- _class: lead -->

# Cloudflare MCP Server Portal × Cloudflare Tunnel × Cloud Run で社内 MCP 基盤を構築した話

2026-11-11 Tamachi.sre#7

---

<!-- _class: profile -->

## 自己紹介

![w:1000](h6o.jpeg)

- hide
- 開発エンジニア（PHP / TypeScript） -> SRE
- 趣味
    - サッカー（長年）
    - ビリヤニ（最近）

---

## 今日のゴール

- 社内 MCP 基盤を構築するときの選択肢を 1 つ知る
- その構成のポイントや注意点を知る

---

## 背景

- ビジネス側のメンバーも、**社内のデータを Claude から利用したい**という要望があった
  - 例: AI に「この商材で過去に似た施策をやったことがあるか調べて」と頼むだけで、社内の施策データベースを検索できる
- 社内 API はすでにあるが、ビジネス側のメンバーが直接使うのは現実的ではない
  - API ごとに URL や認証方式が違う
  - 利用者一人ひとりに API キーを配るのはセキュリティ的に危ない
- API の機能を MCP で提供することにした

---

## アーキテクチャの制約

- Claude からの通信は、社内ではなく Anthropic の IP 範囲（`160.79.104.0/21`, `153.61.208.0/20`）から来る
- MCP サーバーへのアクセスを社内 IP アドレスで制限するのは難しい
- IP 制限ではなく、**入口だけをインターネットに公開し、リクエストのたびに認証・認可を行う**。MCP サーバー本体はインターネットから届かない場所に置く

---

<!-- _class: figure full -->

## アーキテクチャ

![](architecture.svg)

---

<!-- _class: figure -->

## Cloudflare MCP Server Portal

- 複数の MCP サーバーを 1 つの URL にまとめて提供する Cloudflare の機能

![](fig-portal.svg)

---

<!-- _class: figure -->

## Cloudflare Tunnel

- サーバー側から Cloudflare へ暗号化された通信経路（Tunnel）を張り、**グローバル IP やポート開放なし**でサービスを届けられる仕組み

![](fig-tunnel.svg)

---

<!-- _class: figure full -->

## アーキテクチャ（再掲）

![](architecture.svg)

---

<!-- _class: lead -->

# なるべく「閉じる」

アーキテクチャのポイント

---

<!-- _class: figure -->

## 閉じる ①: Cloud Run の ingress を `internal` に

![](fig-close-ingress.svg)

---

<!-- _class: figure -->

## 閉じる ②: Cloudflare Tunnel で入口を作らない

![](fig-close-tunnel.svg)

---

<!-- _class: figure -->

## 閉じる ③: Private hostname でつなぐ

- Portal は Cloudflare Gateway を経由して Tunnel の Private hostname に届く（2026 年 9 月に追加された機能。[Private MCP server support for MCP server portals](https://developers.cloudflare.com/changelog/post/2026-09-22-private-mcp-servers/)）

![](fig-close-private.svg)

---

<!-- _class: figure -->

## ぶつかった壁

- WARP 端末（通信を Gateway に送るアプリを入れた社内 PC）から Portal を迂回して MCP サーバーに届いてしまった

![](fig-wall.svg)

---

<!-- _class: figure -->

## 原因と対処

- 原因: Private hostname は「Gateway 経由なら届く」であって「Portal からしか届かない」ではなかった
- 対処: Gateway の HTTP ポリシーで、Traffic Source（通信の入口）が Portal 以外なら MCP サーバー宛てを Block

![](fig-fix.svg)

---

## まとめ

- Cloudflare MCP Server Portal を使えば、MCP 基盤を簡単に作れる
- Portal と Cloudflare Tunnel を組み合わせると、セキュアな構成にできる
- ただし MCP サーバー宛ての通信を Portal 経由だけに制限しないと、WARP など Gateway を経由する通信から MCP サーバーに直接届いてしまうので注意

---

<!-- _class: end -->
