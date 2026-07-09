# Hera からの呼び出しシーケンス

対象: [d2c-zeus/nidan-hera-js の src/hera](https://github.com/d2c-zeus/nidan-hera-js/tree/master/src/hera) 配下。

Hera は**広告主ページに貼られる広告効果測定用の計測タグ**。
弊社がユニークに付与しているユーザー ID(NidanID / DadUID)を特定するために、
同梱された NidanJS の関数(`NidanIdManager.getNidanAndDadUID` = `getIds` 相当)を利用し、
特定した ID を計測イベント(コンバージョン等)に付与して送信する。

構成要素:
- **base スニペット**(`src/hera/src/base/base.prod.js`) — 広告主ページに直接貼られる短い JS。イベントキュー `etq` と登録関数 `et()` を用意し、CDN から Hera 本体を非同期ロードする。
- **Hera 本体 JS**(`src/hera/src/*.ts` → index.js) — `tsconfig.hera.json` のビルドで **nidan のコードを同梱**した単一スクリプト。

```mermaid
sequenceDiagram
    autonumber
    box ブラウザ(広告主ページ)
        participant Page as 広告主ページ<br/>(base スニペット)
        participant Hera as Hera 本体 JS<br/>(計測タグ)
        participant JS as nidan JS<br/>(Hera に同梱)
    end
    participant CDN as CDN<br/>(cdn.hera.d2c.ne.jp)
    participant IDAPI as Nidan / Daisy API<br/>(ID 同期)
    participant HeraAPI as Hera API<br/>(/v1/clicks, /v1/events)

    Note over Page,CDN: ① ページ表示・タグ読み込み
    Page->>Page: base スニペット実行<br/>イベントキュー etq と et() を用意<br/>d2c.nidan.domain を設定
    Page->>Page: et('イベント名', 'トラッカーID') で<br/>計測イベントをキューに積む(本体ロード前でも可)
    Page->>CDN: GET /1.0/index.js(async)
    CDN-->>Page: Hera 本体 JS(nidan JS 同梱)
    Page->>Hera: スクリプト実行<br/>(HeraEventManager 初期化)

    Note over Page,HeraAPI: ② 初期化タスクチェーン(ID 特定 → クリック処理)
    Note over Hera: NidanTask → NidanAndUACHSync → ActivityLog<br/>→ ClickIDSync → NotifyClick → StartSendingEvent

    Hera->>JS: NidanTask:<br/>getNidanAndDadUID(callback)<br/>※ ユーザー ID の特定を依頼
    JS->>IDAPI: NidanID / DadUID を同期(JSONP)<br/>※ 詳細は nidan_overview.md の概要シーケンス参照
    IDAPI-->>JS: NidanID, DadUID
    JS-->>Hera: callback(nidanId, dadUid)<br/>→ HeraContext に保存
    Hera->>Hera: ActivityLog 準備(activityLogId 発行)
    Hera->>Hera: ClickIDSync:<br/>URL の dadcid クエリからクリック ID を取得

    opt 新しいクリック ID がある場合(前回送信分と不一致)
        Hera->>HeraAPI: GET /v1/clicks/{clickId}(JSONP)<br/>?b=browserId&n=nidanId&d=dadUid&r=現在URL
        HeraAPI-->>Hera: browserId(callback: d2c.hera.cr)<br/>→ localStorage に保存
    end

    Note over Page,HeraAPI: ③ 計測イベント送信
    Hera->>Hera: グローバルの et() を本実装に差し替え<br/>キュー etq のイベントを消化開始
    loop キューされたイベントごと
        Hera->>HeraAPI: img タグ追加(ビーコン)<br/>GET /v1/events/{trackerId}/{eventName}/tag.gif<br/>?a=activityLogId&n=nidanId&d=dadUid&r=URL 等
        opt サードパーティ許可イベント(pv, dadoptpoint, dadpluslcv)
            Hera->>HeraAPI: XHR GET(サードパーティ用ビーコン)<br/>応答の JS を script タグとして実行
        end
    end
```

## 補足

- **ID 特定が最優先**: タスクチェーンの先頭が NidanTask であり、NidanID / DadUID が確定するまで計測イベントは送信されない(キューに滞留する)。ID を必ずイベントに紐付けるための設計。
- **cp フラグ**: `d2c.hera.cp` が false の場合は NidanTask の代わりに GetDaisyIdTask が使われ、DadUID のみで動作する(NidanID はイベントに付与されない)。
- **イベント URL のパラメータ**: `a` = activityLogId、`n` = NidanID、`d` = DadUID、`r` = ページ URL、`b` = browserId、`nt` = Navigation Timing 種別。キャッシュバスター `cb` も付与される。
- **クリック計測**: 広告クリックで遷移してきた場合、ランディング URL の `dadcid` パラメータをクリック ID として `/v1/clicks/` に通知し、返却された browserId を localStorage に保持する(クリック→コンバージョンの紐付け用)。同じクリック ID は再送しない。
- **アナリティクスモード**: URL に特定のクエリ(ビルド時埋め込みのキー/値)が付いている場合、ClickIDSync はアナリティクス用 API からクリック ID を JSONP で取得する経路に切り替わる。
