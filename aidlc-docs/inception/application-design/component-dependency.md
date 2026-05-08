# Component Dependency

## アーキテクチャ概要

```
┌─────────────────────────────────────────────────────────────────┐
│                         LINE Platform                            │
│  ┌──────────┐  ┌──────────────┐  ┌───────────────────────────┐ │
│  │   LIFF   │  │ Messaging API│  │      LINE Webhook         │ │
│  └────┬─────┘  └──────┬───────┘  └─────────────┬─────────────┘ │
└───────┼────────────────┼────────────────────────┼───────────────┘
        │                │                        │
        ▼                │                        ▼
┌──────────────┐         │              ┌──────────────────┐
│  CloudFront  │         │              │   API Gateway    │
│  + S3 (SPA)  │         │              │  (Webhook URL)   │
│              │         │              └────────┬─────────┘
│ React + Vite │         │                      │
└──────┬───────┘         │                      ▼
       │                 │              ┌──────────────────┐
       ▼                 │              │  line-webhook    │
┌──────────────┐         │              │    (L-11)        │
│  API Gateway │         │              └────┬──────────┬──┘
│  (REST API)  │         │                   │          │
└──────┬───────┘         │                   ▼          ▼
       │                 │          ┌───────────┐  ┌───────┐
       ▼                 │          │ Bedrock   │  │  S3   │
┌──────────────────┐     │          │(AI応答)   │  │(画像) │
│ API Lambdas      │     │          └───────────┘  └───────┘
│ L-01〜L-07, L-12 │     │
└──────┬───────────┘     │
       │                 │
       ▼                 ▼
┌──────────────────────────────────────┐
│            DynamoDB Tables           │
│  Users │ UserNamekemono │ ...        │
└──────────────────────────────────────┘

       ┌───────────────────────────────┐
       │       EventBridge Rules       │
       │  21:00  │ 8:00 │ 10/13/16:00 │
       └────┬────┴───┬──┴──────┬──────┘
            ▼        ▼         ▼
       ┌────────┐┌────────┐┌──────────┐
       │ L-08   ││ L-09   ││  L-10    │
       │ night  ││morning ││ otukai   │
       └────┬───┘└───┬────┘└────┬─────┘
            │        │          │
            ▼        ▼          ▼
       ┌──────────────────────────────┐
       │     LINE Messaging API       │
       │        (Push通知)            │
       └──────────────────────────────┘
```

---

## 依存関係マトリクス

### Lambda → サービス

| Lambda | GameCore | Shop | Affinity | Notification | Otukai | User |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| L-01 user-api | ○ | | | | | ○ |
| L-02 swipe-api | ○ | | | | | |
| L-03 forest-api | ○ | | | | | |
| L-04 shop-api | | ○ | | | | |
| L-05 affinity-api | | ○ | ○ | | | |
| L-06 encyclopedia-api | | | | | | |
| L-07 diary-api | | | | | | |
| L-08 night-notification | | | | ○ | | |
| L-09 morning-notification | | | | ○ | | |
| L-10 otukai-scheduler | | | | | ○ | |
| L-11 line-webhook | | | | | ○ | ○ |
| L-12 dev-api | ○ | | | ○ | ○ | ○ |

### Lambda → 共通モジュール

| Lambda | db-client | line-client | master-data | game-config | s3-client | auth |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| L-01 user-api | ○ | | | ○ | | ○ |
| L-02 swipe-api | ○ | | ○ | ○ | | ○ |
| L-03 forest-api | ○ | | ○ | ○ | | ○ |
| L-04 shop-api | ○ | | ○ | ○ | | ○ |
| L-05 affinity-api | ○ | | ○ | ○ | | ○ |
| L-06 encyclopedia-api | ○ | | ○ | | | ○ |
| L-07 diary-api | ○ | | | | | ○ |
| L-08 night-notification | ○ | ○ | ○ | | | |
| L-09 morning-notification | ○ | ○ | | | | |
| L-10 otukai-scheduler | ○ | ○ | ○ | | | |
| L-11 line-webhook | ○ | ○ | ○ | | ○ | |
| L-12 dev-api | ○ | ○ | ○ | | | (共有シークレット検証) |

### Lambda → 外部サービス

| Lambda | DynamoDB | LINE Messaging | Bedrock | S3 (画像) | 天気API | 祝日API |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| L-01〜L-07 | ○ | | | | | |
| L-08 night-notification | ○ | ○ | | | | |
| L-09 morning-notification | ○ | ○ | | | | |
| L-10 otukai-scheduler | ○ | ○ | ○ | | ○ | ○ |
| L-11 line-webhook | ○ | ○ | ○ | ○ | | |
| L-12 dev-api | ○ | ○ | ○ | | | |

---

## DynamoDBテーブル設計

### テーブル一覧

| # | テーブル名 | PK | SK | 概要 |
|---|---|---|---|---|
| T-01 | Users | userId (LINE userId) | — | ユーザー情報、NMK残高、最終ログイン、オンボーディング状態、**notificationEnabled**、**currentOtukaiId**（進行中おつかい参照）、通知時刻設定、地域設定、displayName |
| T-02 | UserNamekemono | userId | namekemonoId | ユーザーが所持するナマケモノ（親密度、出現日、最終収穫日時） |
| T-03 | UserProgress | userId | progressType#key | スワイプ履歴、出現条件進行状況、途中離脱回数、所持品、**pendingAppearances** |
| T-04 | Letters | userId | letterId (timestamp) | 受け取った手紙（ナマケモノID、テキスト、日付） |
| T-05 | OtukaiHistory | userId | otukaiId (timestamp) | おつかい履歴（内容、会話ログ、達成状態（status: in_progress/completed）、日記テキスト、画像S3 URI配列） |

### UserProgressのSK構成例

| SK | 内容 |
|---|---|
| `swipe#count#total` | 累積スワイプ回数 |
| `swipe#count#category#ねむねむ` | カテゴリ別スワイプ回数（「やった」のみ） |
| `swipe#abandoned#count` | 途中離脱回数 |
| `inventory#items` | 所持品（Map: {itemId: count}） |
| `pending#{namekemonoId}` | 保留中の出現予定（appearAt: ISO8601）|

### 主要アクセスパターン

| # | アクセスパターン | テーブル | クエリ |
|---|---|---|---|
| AP-01 | ユーザー情報取得 | Users | GetItem(userId) |
| AP-02 | ユーザーのナマケモノ一覧 | UserNamekemono | Query(userId) |
| AP-03 | 特定ナマケモノの親密度 | UserNamekemono | GetItem(userId, namekemonoId) |
| AP-04 | スワイプ進行状況 | UserProgress | Query(userId, begins_with("swipe#")) |
| AP-05 | 所持品取得 | UserProgress | GetItem(userId, "inventory#items") |
| AP-06 | 手紙一覧 | Letters | Query(userId, ScanIndexForward=false) |
| AP-07 | おつかい履歴 | OtukaiHistory | Query(userId, ScanIndexForward=false) |
| AP-08 | 通知対象ユーザー取得 | Users | Scan（notificationEnabled=trueフィルタ、〜5ユーザーのため許容） |
| AP-09 | おつかい対象者判定 | UserNamekemono + OtukaiHistory | Query + フィルタ |
| AP-10 | pendingAppearances取得 | UserProgress | Query(userId, begins_with("pending#")) |
| AP-11 | 朝通知対象ユーザー取得 | Users (Scan) + UserProgress | Scan → 各ユーザーのpendingを確認 |
| AP-12 | 進行中おつかい取得 | Users + OtukaiHistory | Users.GetItem(userId) → currentOtukaiId → OtukaiHistory.GetItem(userId, currentOtukaiId) |

> **Note**: 〜5ユーザーの規模のため、Scanは許容。GSIは不要。

---

## データフロー

### スワイプ報告フロー（翌朝出現方式）

```
[フロント] SwipeReportPage
    │
    ├─ startSwipe ──→ [L-02] swipe-api
    │                    │
    │                    ├─ master-data: ダメカード30種から10枚選出
    │                    └─ 10枚をレスポンス（stateless）
    │
    └─ submitSwipeResult ──→ [L-02] swipe-api
                                 │
                                 ├─ cardIdの妥当性検証（master-dataと照合）
                                 │
                                 ├─ GameCoreService.calculateSwipeReward
                                 │    └─ game-config: NMK計算パラメータ
                                 │
                                 ├─ GameCoreService.checkAppearanceConditions
                                 │    ├─ DB(UserProgress): swipe#count#* 更新
                                 │    └─ master-data: 出現条件マスタ
                                 │
                                 ├─ DB(Users): NMK残高更新
                                 │
                                 └─ [条件達成時]
                                    DB(UserProgress): pending#{namekemonoId}
                                    に appearAt=翌朝JST8:00 を登録
                                    → レスポンス: 「森に足跡が…」
```

### 森画面フロー（pendingAppearances実体化）

```
[フロント] ForestPage
    │
    └─ getForestState ──→ [L-03] forest-api
                              │
                              ├─ GameCoreService.resolvePendingAppearances
                              │    ├─ DB(UserProgress): pending#* を取得
                              │    ├─ appearAt <= now のものを抽出
                              │    ├─ DB(UserNamekemono): 実体化して追加
                              │    └─ DB(UserProgress): pendingを削除
                              │
                              ├─ DB(UserNamekemono): 全ナマケモノ取得
                              │
                              └─ レスポンス: ナマケモノ一覧 + 新着ナマケモノ配列
```

### おつかいフロー（画像対応）

```
[EventBridge] 10:00/13:00/16:00
    │
    └─→ [L-10] otukai-scheduler
              │
              ├─ OtukaiService.getEligibleUsers
              │    ├─ DB(UserNamekemono): 親密度MAX判定
              │    ├─ DB(OtukaiHistory): 前回配信からの経過日数
              │    └─ DB(Users): currentOtukaiId が空のユーザーのみ
              │
              ├─ 天気API / 祝日API: ガードレール情報取得
              │
              ├─ OtukaiService.generateOtukai
              │    ├─ master-data: キャラ設定・口調参照
              │    └─ Bedrock: おつかい内容生成
              │
              ├─ DB(OtukaiHistory): おつかい記録（status=in_progress）
              ├─ DB(Users): currentOtukaiId = 新otukaiId セット
              │
              └─ LINE Messaging API: Push送信

[LINE Webhook] ユーザー返信
    │
    └─→ [L-11] line-webhook
              │
              ├─ handleWebhook: 署名検証
              │
              ├─ DB(Users): currentOtukaiId 取得
              ├─ DB(OtukaiHistory): GetItem(userId, currentOtukaiId) で進行中特定
              │
              ├─ OtukaiService.processReply
              │    ├─ [テキスト] master-data参照 + Bedrockへ
              │    ├─ [画像] 
              │    │   ├─ line-client.fetchMessageContent(messageId)
              │    │   ├─ s3-client.putObject
              │    │   │    (otukai/{userId}/{otukaiId}/{timestamp}.jpg)
              │    │   └─ master-data参照 + Bedrockにマルチモーダルで渡す
              │    └─ Bedrock: キャラクタートーンで応答生成
              │
              ├─ LINE Messaging API: Reply送信
              │
              └─ [会話終了時]
                   ├─ OtukaiService.judgeCompletion
                   │    └─ Bedrock: 達成判定
                   ├─ DB(Users): currentOtukaiId クリア
                   └─ [達成時のみ] OtukaiService.recordDiary
                        ├─ Bedrock: 会話ログをキャラクタートーンで要約
                        └─ DB(OtukaiHistory): status=completed、日記+画像URI配列記録
                        ※未達成時は日記を残さない（statusのみ更新）
```

### LINE友だち追加/ブロックフロー

```
[LINE Webhook] Follow/Unfollow
    │
    └─→ [L-11] line-webhook
              │
              ├─ handleWebhook: 署名検証
              │
              ├─ [Follow]
              │    ├─ UserService.getOrCreateUser(userId, displayName=null)
              │    │    → displayNameは空欄で登録（LIFF初回起動時に更新）
              │    └─ UserService.setNotificationEnabled(true)
              │
              └─ [Unfollow]
                   └─ UserService.setNotificationEnabled(false)
                      → DB(Users): notificationEnabled = false

[LIFF初回起動] displayName更新
    │
    └─→ [L-01] user-api / getUser
              │
              ├─ フロントから liff.getProfile() の displayName を受信
              ├─ UserService.getOrCreateUser(userId, displayName)
              │    → 既存レコードのdisplayNameを更新
              └─ レスポンス返却
```
