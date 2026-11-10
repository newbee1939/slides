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
  ul ul li::marker { color: var(--sky-deep); }
  li { margin-top: 0.4em; }
  blockquote {
    background: var(--sky-soft);
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

# Cloudflare MCP Server Portal × Tunnel で社内 MCP 基盤を作った話

2026-11-11 Tamachi.sre#7

---

<!-- _class: profile -->

## 自己紹介

![w:1000](h6o.jpeg)

- hide
- 開発エンジニア → SRE
- 趣味: サッカー、ビリヤニ

---

## 背景

- ビジネス側のメンバーも**社内のデータを Claude から使いたい**
- 社内 API を直接使うのは難しい（URL・認証方法がバラバラ、API キー配布は危ない）
- → MCP を利用する方針に

---

## アーキテクチャの制約

- Claude アプリからの通信は、社内ではなく Anthropic の IP 範囲（`160.79.104.0/21`, `153.61.208.0/20`）から来る
- MCP サーバーへのアクセスを社内 IP で制限するのは難しい
- そこで、次の方針にした

<br>

> IP 制限ではなく、入口だけをインターネットに公開し、リクエストのたびに認証・認可を行う。MCP サーバー本体はインターネットから届かない場所に置く

---

<!-- _class: figure -->

## Cloudflare MCP Server Portal

複数の MCP サーバーを 1 つの URL にまとめ、Cloudflare Access（誰がアクセスできるかをポリシーで判定する機能）で守って提供する Cloudflare の機能

![](fig-portal.svg)

---

<!-- _class: figure -->

## Cloudflare Tunnel

サーバー側から Cloudflare へ外向きの通信経路（Tunnel）を張り、**グローバル IP やポート開放なし**でサービスを公開できる機能

![](fig-tunnel.svg)

---

<!-- _class: figure full -->

## アーキテクチャ

![](architecture.svg)

---

<style scoped>section > p { font-size: 56px; font-weight: 700; color: var(--sky-deep); }</style>

## アーキテクチャのポイント

なるべく「閉じる」

---

<!-- _class: figure -->

## 閉じる ①: Cloud Run の ingress を `internal` に

![](fig-close-ingress.svg)

---

<!-- _class: figure -->

## 閉じる ②: Private hostname でつなぐ

Portal から Cloudflare Gateway（Cloudflare を通る通信を検査・制御する機能）を経由して、Tunnel の Private hostname につなぐ

![](fig-close-private.svg)

---

<!-- _class: figure -->

## 発生した問題

WARP 端末（通信を Gateway に送るアプリを入れた社内 PC）からの通信が、Portal を迂回して MCP サーバーに届いてしまった

![](fig-wall.svg)

---

<style scoped>section > p { font-size: 44px; font-weight: 700; color: var(--sky-deep); }</style>

## 原因

Private hostname で保証されるのは「Gateway 経由なら届く」ことだけで、「Portal からしか届かない」わけではない

---

<!-- _class: figure -->

## 対処

Gateway の HTTP ポリシーで、Traffic Source（通信元） が Portal 以外なら MCP サーバー宛てをブロックする条件を追加

![](fig-fix.svg)

---

## まとめ

- Cloudflare MCP Server Portal を使えば、MCP 基盤を簡単に作れる
- Portal と Cloudflare Tunnel を組み合わせると、セキュアな構成にできる
- ただし、WARP など Gateway 経由の通信も直接届いてしまうので、MCP サーバー宛ての通信は Portal 経由だけに制限する

---

<!-- _class: end -->
