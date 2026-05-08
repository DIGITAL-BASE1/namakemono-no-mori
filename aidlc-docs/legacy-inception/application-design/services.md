# サービス定義 — ナマケモノの森 MVP v1

---

## サービス一覧

### 1. ゲームコアサービス（Game Core Service）

**責務**: ゲームの中核ロジックを統括

| 機能 | 説明 |
|---|---|
| スワイプ報告処理 | カード結果の受付、コイン計算、出現条件進行 |
| 出現条件判定 | マスターデータの条件とユーザー履歴を照合 |
| コイン収穫 | 日次収穫量の計算と付与 |
| 放置ボーナス | 未ログイン日数に応じたボーナス計算 |

**依存先**: DynamoDB、マスターデータ

---

### 2. ショップサービス（Shop Service）

**責務**: アイテム購入と在庫管理

| 機能 | 説明 |
|---|---|
| アイテム一覧取得 | マスターデータからアイテム情報を返す |
| 購入処理 | コイン残高チェック → 消費 → アイテム付与 |

**依存先**: DynamoDB、マスターデータ

---

### 3. 親密度サービス（Affinity Service）

**責務**: ナマケモノとの親密度管理

| 機能 | 説明 |
|---|---|
| 贈り物処理 | アイテムとナマケモノの好み照合 → 親密度増減 |
| 手紙解放判定 | 親密度が★★★に達したか判定 |
| リアクション選択 | 贈り物に対するナマケモノのリアクション決定 |

**依存先**: DynamoDB、マスターデータ

---

### 4. 通知サービス（Notification Service）

**責務**: LINE Messaging APIを通じた通知送信

| 機能 | 説明 |
|---|---|
| 夜のスワイプ通知 | ナマケモノ選出 → セリフ選択 → Push送信 |
| 朝の到着通知 | 新到着ユーザー判定 → Push送信 |
| 手動トリガー | 開発用：任意タイミングで通知発火 |

**依存先**: LINE Messaging API、DynamoDB、EventBridge

---

### 5. ユーザーサービス（User Service）

**責務**: ユーザーデータの管理

| 機能 | 説明 |
|---|---|
| ユーザー初期化 | LINEフォロー時にユーザーデータ作成 |
| 状態取得 | コイン、ナマケモノ、親密度、図鑑進捗の一括取得 |
| 図鑑データ取得 | 発見状況、ヒント解放状態の取得 |

**依存先**: DynamoDB、LINE Login

---

### 6. マスターデータサービス（Master Data Service）

**責務**: ゲーム定義データの提供

| データ種別 | 内容 |
|---|---|
| ナマケモノ定義 | 5体の名前、カテゴリ、出現条件、セリフ（5種）、好き嫌い |
| ダメカード定義 | 30枚のカテゴリ、テキスト |
| ショップアイテム定義 | アイテム名、価格、効果、対応ナマケモノ |
| 手紙定義 | ナマケモノごとの手紙テキスト |

**実装方針**: MVP v1ではJSONファイルとしてLambdaにバンドル（DBに入れない）。変更頻度が低く、5体分のデータ量なので十分。

---

## サービス間のオーケストレーション

```
【スワイプ報告フロー】
  SwipePage → API Gateway → submitSwipeReport Lambda
    → ゲームコアサービス.processSwipeReport()
      → coinCalculator.calculate()
      → conditionEngine.evaluate()
      → DynamoDB更新
    → レスポンス返却

【コイン収穫フロー】
  ForestPage → API Gateway → harvestCoins Lambda
    → ゲームコアサービス.harvest()
      → coinCalculator.calculateDailyHarvest()
      → DynamoDB更新
    → レスポンス返却

【贈り物フロー】
  GiftPage → API Gateway → giveGift Lambda
    → 親密度サービス.processGift()
      → affinityManager.calculate()
      → 手紙解放判定
      → DynamoDB更新
    → レスポンス返却

【夜の通知フロー】
  EventBridge（毎晩トリガー）→ sendNightNotification Lambda
    → 通知サービス.sendNightMessages()
      → ユーザー一覧取得
      → 各ユーザーのナマケモノからランダム選出
      → LINE Messaging API Push送信

【朝の到着フロー】
  EventBridge（毎朝トリガー）→ checkNewArrivals Lambda
    → ゲームコアサービス.processArrivals()
      → 条件達成ユーザー判定
      → ナマケモノ到着ステータス更新
    → 通知サービス.sendMorningMessages()
      → LINE Messaging API Push送信
```
