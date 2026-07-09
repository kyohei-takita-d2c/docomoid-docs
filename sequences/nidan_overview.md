# nidan 概要シーケンス図

対象: [d2c-zeus/nidan-hera-js の src/nidan](https://github.com/d2c-zeus/nidan-hera-js/tree/master/src/nidan) 配下の TypeScript 全体。呼び出し契機ごとの概要シーケンスと関連ドキュメントの入口。

## ビルド構成(tsconfig / gulp より)

- **index.js(nidan 本体)** — `tsconfig.nidan.json` により `src/nidan/src/main.ts` + `src/nidan/src/lib/*.ts` を単一ファイルに結合(`outFile`)。`namespace d2c.nidan` の単一アプリ。
- **index-dASync.js(DASync)** — `tsconfig.dasync.json` により `src/nidan/src/lib-dasync/dasync.ts` のみを別バンドル化。本体 index.js と同じページに並載され、本体の `d2c.nidan.getIds` を呼び出して動作する(index-dasync.html 参照)。

nidan はメディアサイト(掲載サイト)に設置されるブラウザ用 ID 同期 SDK で、`task.ts` の `Task<T>` 基底クラスによるタスクチェーンで NidanID / DadUID(DaisyID)の同期とログ送信を行う。全タスクは `context.ts` の `NidanContext`(完了フラグ・コールバックキュー)を共有する。

nidan の呼び出し契機は 2 種類あり、それぞれ概要シーケンス(ブラウザ / CDN / JS / API の粒度)を用意している。

1. **掲載サイト契機**(本ページ下記) — 掲載サイト(メディア)に nidan JS 単体(index.js)が設置され、ページ表示を契機に NidanID を同期する。Allox が `getId` で NidanID を利用する。
2. **Hera タグ契機**([hera_call.md](hera_call.md)) — 広告主ページに貼られた広告効果測定タグ Hera が、ユーザー ID 特定のために同梱 nidan の `getNidanAndDadUID`(= `getIds`)を呼び出す。

関連:
- Prebid 連携時の ID 取得シーケンスと遅延分析(現行 pubProvidedId 方式 / User ID サブモジュール方式の比較)は [prebid_id_integration.md](prebid_id_integration.md) を参照。
- ID 取得と ID 発行を分離した「あるべき姿」のシーケンスは [nidan_to_be.md](nidan_to_be.md) を参照。

## ① 掲載サイト契機の概要シーケンス

内部のタスクチェーンを省略し、コンポーネント間のやりとりだけに絞った流れ。

```mermaid
sequenceDiagram
    autonumber
    box ブラウザ(掲載サイトのページ)
        participant Page as 掲載サイトのページ<br/>(Allox 等の利用側 JS を含む)
        participant JS as nidan JS<br/>(index.js)
    end
    participant CDN as CDN<br/>(nidan JS 配信)
    participant API as Nidan / Daisy API<br/>(nidan.addlv.smt.docomo.ne.jp)

    Note over Page,CDN: 掲載サイトのページ表示
    Page->>CDN: nidan JS を取得<br/>(script タグ。事前に d2c.nidan.domain を設定)
    CDN-->>Page: index.js
    Page->>JS: 同一ページ内でスクリプト実行(初期化)

    JS->>JS: localStorage の NidanID を確認
    alt NidanID がローカルにある
        JS->>JS: NidanID 確定(通信なし)<br/>アクセスログを送信
    else 無い
        JS->>API: GET /id/nidan/pre(JSONP)
        API-->>JS: NidanID(callback: d2c.nidan.ns)
        JS->>JS: NidanID を localStorage に保存
        JS->>API: GET /id/nidan/receiver(JSONP)<br/>※ 発行用 Cookie の削除。応答は使わない
    end

    Page->>JS: d2c.nidan.getId(callback)<br/>(Allox が NidanID を要求)
    Note over JS: 同期済みなら即 callback。<br/>同期中ならキューに積んで完了時に通知
    JS-->>Page: callback(nidanId)

    opt DASync 併載ページの場合(index-dASync.js)
        Page->>JS: d2c.nidan.getIds(callback)
        JS->>API: GET /id/daisy/sync(JSONP)
        API-->>JS: DadUID(callback: d2c.nidan.ds)
        JS-->>Page: callback(nidanId, dadUid)<br/>→ daSync API へ Cookie 同期
    end
```

## ② Hera タグ契機の概要シーケンス

広告主ページの Hera 計測タグが nidan を呼び出す流れは **[hera_call.md](hera_call.md)** を参照。

## 詳細シーケンス(タスクチェーンの粒度)

内部のタスクチェーン(`Task<T>`)の動きまで含めた詳細版は **[nidan_detail.md](nidan_detail.md)** を参照。
