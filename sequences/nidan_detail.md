# nidan 詳細シーケンス(タスクチェーンの粒度)

対象: [d2c-zeus/nidan-hera-js の src/nidan](https://github.com/d2c-zeus/nidan-hera-js/tree/master/src/nidan) 配下の TypeScript 全体。
内部のタスクチェーン(`Task<T>`)の動きまで含めた詳細版。概要版は [nidan_overview.md](nidan_overview.md) を参照。

```mermaid
sequenceDiagram
    autonumber
    actor Media as 設置ページ<br/>(掲載サイト: Allox /<br/>広告主ページ: Hera 計測タグ)
    participant DASyncJS as dasync.ts<br/>(別バンドル index-dASync.js)
    participant Manager as d2c.nidan / NidanIdManager<br/>(main.ts, nidan_id_manager.ts)
    participant ActLog as ActivityLogTask<br/>(activity_log.ts)
    participant NSync as NidanSyncTask<br/>(nidan-sync.ts)
    participant AccLog as AccessLogTask<br/>(access_log.ts)
    participant DSync as DadUIDSyncTask<br/>(daisy-sync.ts)
    participant WaitT as WaitTask<br/>(wait-task.ts)
    participant NotifyT as DadUIDNotifyTask<br/>(daduid-notify-task.ts)
    participant Browser as ブラウザ<br/>(DOM / localStorage / Cookie)
    participant Nidan as Nidanサーバー<br/>(/id/nidan/pre, /receiver)
    participant Daisy as Daisyサーバー<br/>(/id/daisy/sync)
    participant LogAPI as ログAPI<br/>(/activity, /access)
    participant DAAPI as daSync API<br/>(/dasync)

    Note over Media,DAAPI: ① スクリプトロード・初期化(main.ts)
    Media->>Manager: index.js 読み込み<br/>(事前に d2c.nidan.domain / tasks を設定)
    Manager->>Manager: setTimeout(scriptLoadedNotify)<br/>→ getInstance()(シングルトン生成)
    Manager->>Browser: localStorage の保存済み NidanID を検証<br/>(80桁英数字でなければ削除)
    Manager->>Manager: レシーバー関数 ns / pd / ds を初期化<br/>domain 未設定なら throw Error
    Note over Manager,AccLog: タスクチェーン構築:<br/>ActivityLogTask → NidanSyncTask → AccessLogTask

    Note over Media,DAAPI: ② アクティビティログ準備
    Manager->>ActLog: start(context)
    ActLog->>ActLog: activityLogId 発行(ランダム20文字)<br/>logging 関数を生成
    ActLog->>Browser: addEventListener('visibilitychange')<br/>※ 解除関数を removers に登録(Hera から削除可)
    ActLog->>NSync: complete() → start(context)

    Note over Media,DAAPI: ③ NidanID 同期 〜 アクセスログ
    NSync->>Browser: localStorage から NidanID / nidanCookieId 取得<br/>(クエリパラメータ __did__ が1つだけあれば優先し保存)
    alt ローカルで NidanID を解決できた
        NSync->>NSync: completeNidanSync = true
        NSync->>Media: 待機中コールバック<br/>(notifyCallbacksForNidanID)へ通知
        NSync->>AccLog: complete() → start(context)
        AccLog->>LogAPI: sendBeacon(/access)<br/>(activityLogId, origin, domain, referrer, nidanId)
    else ローカルに無い(サーバー同期)
        NSync->>NSync: アクセスログ送信を抑止<br/>(compleatSendAccessLog = true)<br/>レシーバー d2c.nidan.ns を定義
        NSync->>Browser: script タグ追加(JSONP)
        Browser->>Nidan: GET /id/nidan/pre<br/>?callback=d2c.nidan.ns&origin&domain&referrer&nv...
        Nidan-->>NSync: d2c.nidan.ns(id) 呼び出し
        NSync->>Browser: NidanID / nidanCookieId を localStorage に保存
        NSync->>NSync: completeNidanSync = true
        NSync->>Media: 待機中コールバックへ通知
        NSync->>AccLog: complete() → start(context)<br/>※ アクセスログ送信はスキップ
        opt NidanID を取得できた場合
            NSync->>Browser: script タグ追加(JSONP)
            Browser->>Nidan: GET /id/nidan/receiver?callback=d2c.nidan.pd<br/>(nidanPublication Cookie の削除。pd は何もしない)
        end
    end

    Note over Media,DAAPI: ④ getId — NidanID のみ取得(主に Allox)
    Media->>Manager: getId(callback)
    alt NidanSync 完了済み
        Manager-->>Media: callback(nidanId) 即時実行
    else 同期中
        Manager->>Manager: notifyCallbacksForNidanID に追加<br/>(③ の完了時に通知)
    end

    Note over Media,DAAPI: ⑤ getIds — NidanID + DadUID 取得(主に Hera)
    Media->>Manager: getIds(callback)
    alt NidanSync・DaisySync とも完了済み
        Manager-->>Media: callback(nidanId, daisyId) 即時実行
    else 同期中
        Manager->>Manager: notifyCallbacksForNidanAndDadUID に追加
        opt DaisySync 未要求(requestedDaisySync フラグで初回のみ)
            Note over Manager,NotifyT: タスクチェーン構築:<br/>ActivityLogTask → DadUIDSyncTask → WaitTask → DadUIDNotifyTask
            Manager->>Manager: getNidanID(→ WaitTask.start)<br/>※ NidanSync 完了との合流用の予約コールバック
            Manager->>ActLog: start(context)<br/>※ 実施済みのため complete のみ
            ActLog->>DSync: complete() → start(context)
            DSync->>Browser: script タグ追加(JSONP)
            Browser->>Daisy: GET /id/daisy/sync<br/>?callback=d2c.nidan.ds&origin&domain&referrer(&ncid)
            Daisy-->>DSync: d2c.nidan.ds(dadUid) 呼び出し
            DSync->>DSync: daisyId 設定<br/>completeDaisySync = true
            DSync->>WaitT: complete() → start(context)
            Note over WaitT: DaisySync 完了経路と NidanSync 完了経路の<br/>2 経路から呼ばれ、両フラグが揃ったときだけ<br/>次へ進む(合流バリア)
            WaitT->>NotifyT: complete() → start(context)
            NotifyT->>Media: 待機中コールバックへ<br/>(nidanId, daisyId) を通知
        end
    end

    Note over Media,DAAPI: ⑥ DASync(index-dASync.js を併載したページのみ)
    opt dasync.ts(別バンドル)が読み込まれている場合
        DASyncJS->>Manager: setTimeout(addDASyncRequest)<br/>→ getIds(callback)(⑤ のフローで解決)
        Manager-->>DASyncJS: callback(nidanid, daduid)
        DASyncJS->>Browser: Cookie 取得(dasynced, daxtr)
        alt 同期済み(dasynced === daduid)or idsite / uid 無し
            DASyncJS->>Browser: 1x1 透明 GIF の img タグを追加して終了
        else 同期が必要
            DASyncJS->>DAAPI: fetch GET /dasync<br/>?idsite&url&uid(=daduid)&dauid(=daxtr)&companyid=13
            DAAPI-->>DASyncJS: レスポンス本文 = 画像 URL(200 / 302)
            opt 302 の場合
                DASyncJS->>Browser: Cookie dasynced=uid を設定(3日間)
            end
            DASyncJS->>Browser: img タグ追加(src = 返却 URL)<br/>※ 同期先へのリクエストを発火
        end
    end

    Note over Media,DAAPI: ⑦ ページ離脱時のアクティビティログ
    Media->>Browser: タブを閉じる / 非表示化(visibilitychange)
    Browser->>ActLog: handler → logging('pageExit')
    ActLog->>LogAPI: sendBeacon(/activity)<br/>(滞在時間・スクロール位置・画面寸法・ncid 等)
```

## 補足(重要な設計ポイント)

- **タスクチェーン**: `Task<T>`(task.ts)が Chain of Responsibility の基底クラス。`complete()` が次タスクの `start(context)` を呼ぶ。初期化時は「ActivityLog → NidanSync → AccessLog」、getIds 初回時は「ActivityLog → DadUIDSync → Wait → DadUIDNotify」の 2 チェーンが動く。
- **非同期待ち合わせ**: `getId` / `getIds` は「同期済みなら即時コールバック、未同期ならキューに積んで完了時に一括通知」。DadUID 側は `requestedDaisySync` フラグで初回のみ同期を起動し、`WaitTask` が NidanSync / DaisySync 両方の完了を待ち合わせる。
- **サーバー通信**: ID 同期はすべて JSONP(script タグ挿入)方式。コールバック名は通信量削減のため `ns`(NidanSync)/ `pd`(PublicationCookieDelete)/ `ds`(DadUIDSync)に短縮。
- **ログ送信**: `navigator.sendBeacon` を使用。`activityLogId` はページ内で 1 つに固定され、アクセスログとアクティビティログを紐付ける。NidanID をローカルで解決できなかった場合、その回のアクセスログは送信されない。
- **設定**: `environment.ts` が接続先 URL・ストレージキー等を保持し、`window._d2c_nidan_option` で一部上書き可能。ログ API ドメインはビルド時に `gulp-preprocess` で環境ごとに埋め込まれる(`env/*.js`)。
- **未使用コード**: `lib/uach.ts`(UACHTask)は全体がコメントアウトされており現在は未使用。

※ 大元リポジトリの `docs/nidan-sequence-diagram.md` には getId / getIds で分割された 2 枚の図があるが、本書はそれらと DASync・ログ送信を含む全体を 1 本に統合したもの。
