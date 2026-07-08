# Prebid 連携時の ID 取得遅延シーケンス

対象: `origin_apps/prebid/*/prd/*_D2CID_PubProvidedID.js`(連携タグ)+ `origin_apps/nidan-hera-js/src/nidan`(nidan JS)

Prebid の `pubProvidedId` へ NidanID / DadUID を渡すまでの遅延を、
**localStorage に NidanID がある場合(再訪問)/ ない場合(初回訪問)** の 2 ケースで図示する。

## 前提条件(見積もりの根拠)

| 項目 | 想定値 |
|---|---|
| ネットワーク RTT | 約 50ms(モバイル 4G 〜 一般的な回線) |
| 新規オリジンへの接続確立(DNS + TCP + TLS) | 約 120〜150ms |
| 小さな JS/JSONP 応答の転送 | 約 1 RTT(50ms)+ サーバー処理時間 |
| Prebid の `requestBids` 発火 | ページ読み込み後 数百 ms 以内が一般的(`auctionDelay` 未設定 = 0ms でオークションは ID を待たない) |

※ 数値はあくまで目安。回線・端末・サーバー負荷により変動する。

## ケース A: localStorage に NidanID が**ある**場合(再訪問) — 合計 約 350〜700ms

NidanID はローカルで即時解決するが、**DadUID(daisyId)は永続化されないため毎回 JSONP 往復が必須**。
これがこのケースの遅延のほぼすべてを占める。

```mermaid
sequenceDiagram
    autonumber
    box ブラウザ(掲載サイトのページ)
        participant Tag as 連携タグ<br/>(D2CID_PubProvidedID.js)
        participant JS as nidan JS<br/>(index.js)
        participant Pbjs as Prebid.js<br/>(pbjs)
    end
    participant CDN as CDN<br/>(cdn.nidan.d2c.ne.jp)
    participant DAPI as Daisyサーバー<br/>(nidan.addlv.smt.docomo.ne.jp)

    Note over Tag,Pbjs: ページ表示。Prebid はこの間も進行し<br/>requestBids は数百 ms 以内に発火しうる

    rect rgb(255, 240, 220)
        Note over Tag,CDN: 遅延❶ CDN から index.js 取得<br/>ブラウザキャッシュ有効: 〜20ms /<br/>キャッシュなし: 約 150〜250ms(接続確立+転送)
        Tag->>CDN: script タグ追加(async)
        CDN-->>Tag: index.js
    end

    rect rgb(255, 240, 220)
        Note over JS: 遅延❷ setTimeout 経由の初期化 + localStorage 読み込み<br/>約 10ms
        Tag->>JS: スクリプト実行 → 初期化
        JS->>JS: localStorage に有効な NidanID あり<br/>→ NidanID 即時確定(ネットワーク往復ゼロ)
    end

    JS-->>Tag: getId コールバック発火(nidanId)
    Note over Tag: ここまで累計 約 30〜280ms

    rect rgb(255, 220, 220)
        Note over Tag,DAPI: 遅延❸ DaisySync JSONP(毎回発生)<br/>新規オリジンへの接続確立 約 150ms<br/>+ サーバー処理 + 転送 約 150〜300ms<br/>= 約 300〜450ms<br/>※ daisyId は localStorage に保存されないため<br/>再訪問でも省略できない
        Tag->>JS: getIds(callback)
        JS->>DAPI: script タグ追加(JSONP)<br/>GET /id/daisy/sync
        DAPI-->>JS: DadUID(callback: d2c.nidan.ds)
        JS->>JS: WaitTask 合流<br/>(NidanID は確定済みのため即通過)
    end

    JS-->>Tag: callback(nidanId, dadUid)

    rect rgb(255, 240, 220)
        Note over Tag,Pbjs: 遅延❹ Prebid への反映<br/>mergeConfig + refreshUserIds 約 10ms
        Tag->>Pbjs: pubProvidedId に eids 設定
    end

    Note over Tag,Pbjs: 合計 約 350〜700ms<br/>(CDN キャッシュ有効なら 350ms 前後)<br/>requestBids に間に合うかは五分五分
```

| # | 遅延要因 | 目安 |
|---|---|---|
| ❶ | CDN からの index.js 取得 | キャッシュ有効 〜20ms / なし 150〜250ms |
| ❷ | 初期化(setTimeout + localStorage 読み込み) | 〜10ms |
| ❸ | **DaisySync JSONP(毎回必須・最大要因)** | **300〜450ms** |
| ❹ | Prebid への反映 | 〜10ms |
| | **合計** | **約 350〜700ms** |

## ケース B: localStorage に NidanID が**ない**場合(初回訪問) — 合計 約 700〜1,200ms

NidanID のサーバー発行(JSONP)が加わり、しかも連携タグが
「getId 完了 → getIds 呼び出し」と直列化しているため、**❸ と ❹ が並列にならず足し算になる**。

```mermaid
sequenceDiagram
    autonumber
    box ブラウザ(掲載サイトのページ)
        participant Tag as 連携タグ<br/>(D2CID_PubProvidedID.js)
        participant JS as nidan JS<br/>(index.js)
        participant Pbjs as Prebid.js<br/>(pbjs)
    end
    participant CDN as CDN<br/>(cdn.nidan.d2c.ne.jp)
    participant NAPI as Nidanサーバー<br/>(js.api.nidan.d2c.ne.jp)
    participant DAPI as Daisyサーバー<br/>(nidan.addlv.smt.docomo.ne.jp)

    Note over Tag,Pbjs: ページ表示(初回訪問)

    rect rgb(255, 240, 220)
        Note over Tag,CDN: 遅延❶ CDN から index.js 取得<br/>初回はキャッシュなし: 約 150〜250ms
        Tag->>CDN: script タグ追加(async)
        CDN-->>Tag: index.js
    end

    rect rgb(255, 240, 220)
        Note over JS: 遅延❷ 初期化 約 10ms<br/>localStorage に NidanID なし
        Tag->>JS: スクリプト実行 → 初期化
    end

    rect rgb(255, 220, 220)
        Note over JS,NAPI: 遅延❸ NidanID 発行 JSONP<br/>新規オリジンへの接続確立 約 150ms<br/>+ ID 発行処理 + 転送 約 100〜250ms<br/>= 約 250〜400ms
        JS->>NAPI: script タグ追加(JSONP)<br/>GET /id/nidan/pre
        NAPI-->>JS: NidanID(callback: d2c.nidan.ns)
        JS->>JS: NidanID を localStorage に保存
    end

    JS-->>Tag: getId コールバック発火(nidanId)
    Note over Tag: ここまで累計 約 400〜700ms<br/>※ この後さらに Cookie 削除の JSONP<br/>(/id/nidan/receiver)も走るが通知はブロックしない

    rect rgb(255, 220, 220)
        Note over Tag,DAPI: 遅延❹ DaisySync JSONP<br/>約 300〜450ms<br/>※ 連携タグが getId 完了後に getIds を呼ぶため<br/>❸ と直列になり所要時間が「合計」になる<br/>(設計上も getIds が呼ばれるまで DaisySync は始まらない)
        Tag->>JS: getIds(callback)
        JS->>DAPI: script タグ追加(JSONP)<br/>GET /id/daisy/sync
        DAPI-->>JS: DadUID(callback: d2c.nidan.ds)
        JS->>JS: WaitTask 合流
    end

    JS-->>Tag: callback(nidanId, dadUid)

    rect rgb(255, 240, 220)
        Note over Tag,Pbjs: 遅延❺ Prebid への反映 約 10ms
        Tag->>Pbjs: pubProvidedId に eids 設定
    end

    Note over Tag,Pbjs: 合計 約 700〜1,200ms<br/>requestBids(数百 ms 以内)には ほぼ間に合わない
```

| # | 遅延要因 | 目安 |
|---|---|---|
| ❶ | CDN からの index.js 取得(初回=キャッシュなし) | 150〜250ms |
| ❷ | 初期化 | 〜10ms |
| ❸ | **NidanID 発行 JSONP** | **250〜400ms** |
| ❹ | **DaisySync JSONP(❸ と直列)** | **300〜450ms** |
| ❺ | Prebid への反映 | 〜10ms |
| | **合計** | **約 700〜1,200ms** |

## まとめ

| | ケース A(NidanID あり) | ケース B(NidanID なし) |
|---|---|---|
| ネットワーク往復(直列) | 1〜2 回(CDN + Daisy) | 3 回(CDN + Nidan + Daisy) |
| 合計目安 | **約 350〜700ms** | **約 700〜1,200ms** |
| requestBids(数百 ms)に間に合うか | 五分五分 | ほぼ間に合わない |

- 両ケースに共通する最大の恒常要因は **❸/❹ DaisySync JSONP**。daisyId が localStorage に永続化されないため、再訪問でも毎ページビューで docomo ドメインへの往復(約 300〜450ms)が必ず発生する。
- ケース B ではさらに NidanID 発行が前段に直列で入る。nidan の設計上 DaisySync は `getIds` 呼び出しまで開始されず、連携タグは `getId` 完了を待って `getIds` を呼ぶため、2 つの JSONP は並列化されない。
- `pubProvidedId` は `auctionDelay` 未設定だとオークションを待たせないため、この合計時間が `requestBids` 発火より遅ければ初回オークションに ID が乗らない。
