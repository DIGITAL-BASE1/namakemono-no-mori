# コンポーネント依存関係 — ナマケモノの森 MVP v1

---

## アーキテクチャ概要

```
┌─────────────────────────────────────────────────────────┐
│  LINE Platform                                          │
│  ┌──────────────┐  ┌──────────────────┐                │
│  │ LINE App     │  │ Messaging API    │                │
│  │ (LIFF Browser)│  │ (Push通知)       │                │
│  └──────┬───────┘  └────────▲─────────┘                │
│         │                    │                          │
└─────────┼────────────────────┼──────────────────────────┘
          │ HTTPS              │ Push
          ▼                    │
┌─────────────────────────────────────────────────────────┐
│  AWS                                                    │
│                                                         │
│  ┌──────────────────┐                                   │
│  │ CloudFront       │                                   │
│  │ (HTTPS配信)      │                                   │
│  └────────┬─────────┘                                   │
│           │                                             │
│  ┌────────▼─────────┐     ┌──────────────────┐         │
│  │ S3               │     │ API Gateway      │         │
│  │ (React SPA)      │     │ (REST API)       │         │
│  └──────────────────┘     └────────┬─────────┘         │
│                                     │                   │
│                            ┌────────▼─────────┐        │
│                            │ Lambda Functions  │        │
│                            │ ・getUserState    │        │
│                            │ ・submitSwipeReport│       │
│                            │ ・harvestCoins    │        │
│                            │ ・purchaseItem    │        │
│                            │ ・giveGift        │        │
│                            │ ・getEncyclopedia │        │
│                            │ ・lineWebhook     │        │
│                            │ ・sendNight/Morning│       │
│                            │ ・checkNewArrivals│        │
│                            └────────┬─────────┘        │
│                                     │                   │
│                            ┌────────▼─────────┐        │
│                            │ DynamoDB         │        │
│                            │ (ユーザーデータ)  │        │
│                            └──────────────────┘        │
│                                                         │
│  ┌──────────────────┐                                   │
│  │ EventBridge      │──→ sendNightNotification          │
│  │ (スケジューラー)  │──→ checkNewArrivals               │
│  └──────────────────┘                                   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 依存関係マトリクス

### フロントエンド → バックエンド

| フロントエンド | 呼び出すAPI機能 | タイミング |
|---|---|---|
| OnboardingDialog | ユーザー登録, スワイプ報告送信, アイテム購入, 贈り物 | 初回起動時 |
| ForestPage | ユーザー状態取得, コイン収穫 | ログイン時 |
| SwipePage | スワイプ報告送信 | スワイプ完了/途中離脱時 |
| ShopPage | ユーザー状態取得（コイン残高）, アイテム購入 | ショップ表示時、購入時 |
| GiftPage | 贈り物 | 贈り物実行時 |
| EncyclopediaPage | 図鑑データ取得 | 図鑑表示時 |
| LetterPage | ユーザー状態取得（手紙データ） | 手紙表示時 |

### バックエンド → 外部サービス

| Lambda | 依存先 | 用途 |
|---|---|---|
| sendNightNotification | LINE Messaging API | Push通知送信 |
| sendMorningNotification | LINE Messaging API | Push通知送信 |
| lineWebhook | LINE Platform | Webhook検証 |
| 全Lambda | DynamoDB | データ読み書き |

### バックエンド内部依存

| Lambda | 使用する共通モジュール |
|---|---|
| submitSwipeReport | conditionEngine, coinCalculator, masterData |
| harvestCoins | coinCalculator, masterData |
| giveGift | affinityManager, masterData |
| checkNewArrivals | conditionEngine, masterData |
| sendNightNotification | masterData（セリフデータ） |
| getEncyclopedia | conditionEngine, masterData |

---

## データフロー

### スワイプ報告 → ナマケモノ出現の流れ

```
1. ユーザー初回起動（LINE ID登録）
   OnboardingDialog → liff.getProfile() → registerUser API
   → LINEユーザーIDをDynamoDBに保存（Push通知の前提条件）

2. ユーザーがスワイプ報告
   SwipePage → submitSwipeReport API

3. サーバーで処理
   → コイン計算（スワイプ分 + 途中離脱ボーナス）
   → 報告履歴をDynamoDBに保存
   → 出現条件エンジンで全未発見ナマケモノの条件を評価
   → 条件達成したナマケモノがあれば「pending」ステータスに

4. レスポンス返却
   → コイン獲得数
   → ほのめかしメッセージ（条件達成: 「足跡が…」/ 進行中: 「誰かが見ている…」/ なし: 「寝ている…」）

5. 翌朝バッチ（EventBridge → checkNewArrivals）
   → pending状態のナマケモノを「arrived」に更新
   → 該当ユーザーに朝の通知を送信（LINE Messaging API、登録済みLINE IDを使用）

6. ユーザーがログイン
   → getUserState で pendingArrivals を取得
   → ForestPage で到着演出を表示
```

---

## データ保存の概要

### 保存が必要なデータ

| データ種別 | 内容 | 備考 |
|---|---|---|
| ユーザー基本データ | コイン残高、設定、所持アイテム、登録日時 | LINEユーザーIDをキーに |
| ナマケモノ状態 | 発見状態、親密度、条件進捗 | ユーザーごと×ナマケモノごと |
| スワイプ履歴 | 日次の報告結果、途中離脱情報 | 出現条件判定に使用 |
| 手紙データ | 受け取った手紙、既読状態 | 手紙コレクション用 |

### マスターデータ（JSONバンドル）

MVP v1ではLambdaにJSONファイルとしてバンドル。DB不要。

| ファイル | 内容 |
|---|---|
| ナマケモノ定義 | 5体の名前、カテゴリ、出現条件、セリフ5種、好き嫌い |
| ダメカード定義 | 30枚のカテゴリ、テキスト |
| ショップアイテム定義 | アイテム名、価格、効果 |
| 手紙定義 | ナマケモノごとの手紙テキスト |

※ 具体的なDynamoDBテーブル設計（PK/SK設計、属性定義）はCONSTRUCTION phaseで詳細化

---

## 通信パターン

| パターン | 使用箇所 |
|---|---|
| **同期 REST API** | フロント → API Gateway → Lambda → DynamoDB → レスポンス |
| **非同期バッチ** | EventBridge → Lambda（通知送信、到着チェック） |
| **Webhook** | LINE Platform → API Gateway → lineWebhook Lambda |
