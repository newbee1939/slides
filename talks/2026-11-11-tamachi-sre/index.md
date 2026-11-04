---
marp: true
paginate: true
---

# Cloudflare MCP Server Portal × Tunnel × Cloud Run で社内 MCP 基盤を構築した話

h6o / Tamachi.sre#7 2026-11-11

---

## 自己紹介

h6o / SREとして働いている

## 今日のゴール

- 社内MCP基盤を構築する際の一つの選択肢を知る
- 注意点を知る

## 背景

- ビジネス側のメンバー（非エンジニア）にも、社内のデータを Claude から使ってほしいという依頼があった
- 例えば「この商材で過去に似た施策をやったことがあるか調べて」と Claude に頼むだけで、社内の施策データベースを検索できる
- 社内には API がすでにありますが、ビジネス側のメンバーが API を直接使うのは現実的ではありません。API ごとに URL や認証方式が違いますし、利用者一人ひとりに API キーを配るのはセキュリティ的にも危ないです
- API の機能を MCP（Model Context Protocol）で提供することに

## アーキテクチャ制約

- IPで制限するのは難しい
- 例えばClaudeの場合、通信は社内ではなく
    -  160.79.104.0/21 と 153.61.208.0/20 からくる
- IP制限ではなく、

> 入口（Portal）だけをインターネットに公開し、リクエストのたびに認証・認可を行う。MCP サーバー本体はインターネットから届かない場所に置く

Zero Trust的な考え方

## アーキテクチャ

<!-- 画像貼る？ -->

アーキテクチャのポイントを話す前に、いくつか事前知識を説明。

## Cloudflare MCP Portalとは

- 社内 MCP の唯一の入口。Claude 向けの OAuth の窓口になり、各サーバーの Access ポリシーで見せるサーバーを、Portal のツール設定で見せるツールを絞って中継する

## Cloudflare Tunnelとは

- Cloud Run の内側から Cloudflare に向けて張る通り道。Cloud Run 側に外から入る口を開けずに済む

## アーキテクチャのポイント

- なるべく「閉じる」
- 口は最小限にする

##　アーキテクチャのポイント詳細

- Cloud RunのIngressはinternal
- 認証・認可はCloudflareに任せる
    - MCP Serverの仕様をシンプルに保てる
- Cloudflare Tunnelにより、内向きの通信を完全に遮断
- Private hostname routeを使うことで内部通信のみ許可
    - Portal は Gateway を経由して、Tunnel の private ホスト名に届きます（2026 年 9 月に追加された機能です。変更履歴）。

## ぶつかった壁・どう解消したか

- WARP 端末から Portal を迂回して届いてしまった
    - GatewayのTraffic SourceをPortalからに制限
        - 特定のMCP Serverに対するアクセスをMCP Portalからのアクセスのみに制限できる
- 同一MCPでアクセス可能な人を絞りたい場合
    - 論理MCP
        - MCPごとに特定のユーザーと特定のツールを許可

## まとめ

- Cloudflare MCP Portalを使えばMCP基盤を簡単に作れる
- MCP PortalとCloudflare Tunnelを組み合わせることでセキュアな構成に
- GatewayへのアクセスをPortalのみに制限しないとWARPなどGatewayを経由する通信であればMCP Serverに直接アクセスできてしまうので注意
