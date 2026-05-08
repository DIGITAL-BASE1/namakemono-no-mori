# Services

## サービスレイヤー定義

バックエンドLambda内で使用するサービス層の定義。各Lambdaはサービスを組み合わせてビジネスロジックを実行する。

---

### S-01: GameCoreService

| 項目 | 内容 |
|---|---|
| **責務** | ゲームのコアロジック（NMK計算、出現条件判定、日次収穫計算、放置ボーナス計算） |
| **利用Lambda** | L-01 (user-api), L-02 (swipe-api), L-03 (forest-api), L-12 (dev-api) |
| **依存** | M-01 (db-client), M-03 (master-data), M-04 (game-config) |

**主要メソッド:**
- `calculateSwipeReward(results, abandoned)` — スワイプ結果からNMK獲得量を計算
- `checkAppearanceConditions(userId, swipeResults)` — ナマケモノ出現条件の判定。達成時は`pendingAppearances`に保留登録
- `resolvePendingAppearances(userId, now)` — `appearAt <= now`のpendingを実体化しUserNamekemonoへ追加
- `calculateHarvest(userId)` — 日次収穫量の計算（親密度ボーナス含む）
- `calculateAndApplyIdleBonus(userId)` — 放置ボーナスの計算 + NMK加算（getUser内で自動実行）

---

### S-02: ShopService

| 項目 | 内容 |
|---|---|
| **責務** | ショップ取引 + **所持品管理（読み書きを集約）** |
| **利用Lambda** | L-04 (shop-api), L-05 (affinity-api：giftItemの内部で在庫減算) |
| **依存** | M-01 (db-client), M-03 (master-data), M-04 (game-config) |

**主要メソッド:**
- `getAvailableItems(userBalance)` — 購入可能アイテム一覧
- `purchaseItem(userId, itemId)` — 購入処理（残高チェック → 減算 → 所持品追加）
- `getInventory(userId)` — 所持品取得
- `consumeItem(userId, itemId)` — 所持品から1つ消費（giftItem時にAffinityServiceから呼ばれる）

---

### S-03: AffinityService

| 項目 | 内容 |
|---|---|
| **責務** | 親密度管理（贈呈処理、レベル判定、解放要素チェック、手紙管理） |
| **利用Lambda** | L-05 (affinity-api) |
| **依存** | S-02 (ShopService), M-01 (db-client), M-03 (master-data), M-04 (game-config) |

**主要メソッド:**
- `giftItem(userId, namekemonoId, itemId)` — 贈呈処理（ShopService.consumeItemを呼んで在庫減算 → 親密度加算 → 解放判定）
- `checkUnlocks(userId, namekemonoId, newLevel)` — 手紙/おつかい解放判定
- `getAffinityLevel(affinityPoints)` — ポイントから★レベルへの変換
- `getLetters(userId)` — 手紙コレクション取得

---

### S-04: NotificationService

| 項目 | 内容 |
|---|---|
| **責務** | LINE Push通知の送信（夜通知、朝通知） |
| **利用Lambda** | L-08 (night-notification), L-09 (morning-notification), L-12 (dev-api) |
| **依存** | M-01 (db-client), M-02 (line-client), M-03 (master-data) |

**主要メソッド:**
- `sendNightNotification(userId, namekemonoId)` — 夜のスワイプ促し通知（キャラ固有セリフ）
- `sendMorningNotification(userId, namekemonoName)` — 朝の新着通知
- `getEligibleUsersForNight()` — 夜通知対象ユーザー取得（notificationEnabled=true）
- `getEligibleUsersForMorning()` — 朝通知対象ユーザー取得（今朝実体化されるpendingAppearancesあり）

---

### S-05: OtukaiService

| 項目 | 内容 |
|---|---|
| **責務** | おつかい機能全般（生成、配信、応答、達成判定、画像処理） |
| **利用Lambda** | L-10 (otukai-scheduler), L-11 (line-webhook), L-12 (dev-api) |
| **依存** | M-01 (db-client), M-02 (line-client), M-03 (master-data), M-05 (s3-client), Amazon Bedrock |

**主要メソッド:**
- `generateOtukai(userId, namekemonoId, context)` — Bedrockでおつかい生成（天気・曜日・履歴考慮、master-dataからキャラ設定・口調参照）。生成後に `Users.currentOtukaiId = 新otukaiId` をセット
- `processReply(userId, message)` — おつかい報告の処理
  - `Users.currentOtukaiId` から進行中おつかいを特定
  - テキストメッセージ: そのままBedrockに渡す
  - 画像メッセージ: `line-client.fetchMessageContent` → `s3-client.putObject` → S3 URIを会話ログに付与 → Bedrockにマルチモーダルで渡す
- `judgeCompletion(userId, otukaiId, conversation)` — 達成判定。会話終了時に `Users.currentOtukaiId` をクリア
- `getEligibleUsers(currentHour)` — おつかい配信対象者の判定（親密度MAX + 前回配信から2〜3日経過 + `currentOtukaiId` が空（進行中なし））
- `recordDiary(userId, otukaiId, conversation)` — 森の日記への記録。**達成時のみ実行**。内部でBedrockを呼び、会話ログをナマケモノのキャラクタートーンで一文に要約してから `OtukaiHistory` に書き込む。未達成時はスキップ

---

### S-06: UserService

| 項目 | 内容 |
|---|---|
| **責務** | ユーザー管理（登録、取得、状態更新、通知設定） |
| **利用Lambda** | L-01 (user-api), L-11 (line-webhook), L-12 (dev-api) |
| **依存** | M-01 (db-client) |

**主要メソッド:**
- `getOrCreateUser(lineUserId, displayName)` — ユーザー取得 or 新規登録。displayNameが指定されている場合はDB更新（LIFF初回起動時経路）
- `updateLastLogin(userId)` — 最終ログイン日時更新
- `isFirstTime(userId)` — 初回起動判定
- `completeOnboarding(userId)` — オンボーディング完了フラグ設定
- `setNotificationEnabled(userId, enabled)` — 通知有効/無効切替（handleFollow/handleUnfollowから呼ばれる）

**displayName運用方針（line-client不要化）:**
- handleFollow（友だち追加）時は userId のみ記録。displayNameは空欄または暫定値
- LIFF初回起動時に `liff.getProfile()` でフロント側がdisplayNameを取得し、getUser経由でサーバーへ渡す → `getOrCreateUser` 内でDB更新
- この方針により、S-06 UserService はLINE Messaging API（line-client）に依存せず、db-clientのみで完結

---

## オーケストレーションフロー

### フロー1: 初回起動

```
LIFF起動 → getUser (L-01)
  → UserService.getOrCreateUser
  → UserService.isFirstTime = true
  → GameCoreService.calculateAndApplyIdleBonus（初回は0）
  → フロント: OnboardingDialog表示
  → 絵本3ページ完了
  → startSwipe (L-02, isFirstTime=true)
    → 固定カード配置（1枚目=ねむねむ強制、2枚目=もぐもぐ強制）
  → submitSwipeResult (L-02)
    → GameCoreService.calculateSwipeReward
    → GameCoreService.checkAppearanceConditions
      → 【例外】オンボーディング時のネムリンは即時出現（UserNamekemonoに追加）
      → モグモグは pendingAppearances に登録（翌朝出現）
  → completeOnboarding (L-01)
  → getForestState (L-03)
    → GameCoreService.resolvePendingAppearances（通常処理）
    → ネムリン表示
  → ショップ体験 → 贈り物体験
```

### フロー2: 通常日次ループ

```
LIFF起動 → getUser (L-01)
  → UserService.updateLastLogin
  → GameCoreService.calculateAndApplyIdleBonus（放置日数 × NMK/日を自動付与）
    → idleBonus情報を返却（フロント演出用）
  → getForestState (L-03)
    → GameCoreService.resolvePendingAppearances
      → appearAt <= now のpendingを UserNamekemono に追加
      → 新着フラグを立てて返却
    → 収穫可否チェック
  → harvestCoins (L-03)
    → GameCoreService.calculateHarvest
  → [夜] スワイプ報告
    → submitSwipeResult (L-02)
    → GameCoreService.checkAppearanceConditions
      → 条件達成時は pendingAppearances に登録（翌朝出現）
```

### フロー3: おつかい（LINE上）

```
EventBridge (10:00/13:00/16:00) → otukai-scheduler (L-10)
  → OtukaiService.getEligibleUsers
    → 親密度MAX + 前回配信から2〜3日経過 + currentOtukaiId=空のユーザー
  → OtukaiService.generateOtukai（Bedrock呼び出し）
    → master-dataからキャラ設定・口調を参照
  → DB(OtukaiHistory): 新規おつかい記録（status=in_progress）
  → DB(Users): currentOtukaiId = 新otukaiId にセット
  → LINE Push送信

ユーザー返信（テキスト） → LINE Webhook → line-webhook (L-11)
  → handleWebhook → 署名検証 → handleOtukaiReply
  → DB(Users).currentOtukaiId 取得 → DB(OtukaiHistory).GetItem で進行中おつかい特定
  → OtukaiService.processReply
    → Bedrockで応答生成（キャラクタートーン、master-data参照）
  → LINE Reply送信

ユーザー返信（画像） → LINE Webhook → line-webhook (L-11)
  → handleWebhook → 署名検証 → handleOtukaiReply
  → DB(Users).currentOtukaiId 取得 → DB(OtukaiHistory).GetItem で進行中おつかい特定
  → OtukaiService.processReply
    → line-client.fetchMessageContent(messageId)
    → s3-client.putObject（キー: otukai/{userId}/{otukaiId}/{timestamp}.jpg）
    → Bedrockにマルチモーダルで画像+会話履歴を渡す
  → LINE Reply送信

[会話終了時]
  → OtukaiService.judgeCompletion
    → DB(Users): currentOtukaiId をクリア
  → [達成時のみ] OtukaiService.recordDiary
    → Bedrockで会話ログを要約（キャラクタートーン）
    → DB(OtukaiHistory): status=completed、日記テキスト記録
  → [未達成時] 日記は残さない（OtukaiHistoryのstatusのみ更新）
```

### フロー4: 通知

```
EventBridge (21:00) → night-notification (L-08)
  → NotificationService.getEligibleUsersForNight
  → NotificationService.sendNightNotification（キャラ固有セリフ）

EventBridge (8:00) → morning-notification (L-09)
  → NotificationService.getEligibleUsersForMorning
    → 今朝実体化されるpendingAppearances（appearAt <= 今朝8:00）を持つユーザー
  → NotificationService.sendMorningNotification
```

### フロー5: LINE友だち追加/ブロック

```
友だち追加 → LINE Webhook → line-webhook (L-11)
  → handleWebhook → handleFollow
  → UserService.getOrCreateUser(userId, displayName=null)
    → displayNameは空欄で登録（LIFF初回起動時に更新される）
  → UserService.setNotificationEnabled(true)

ブロック → LINE Webhook → line-webhook (L-11)
  → handleWebhook → handleUnfollow
  → UserService.setNotificationEnabled(false)

LIFF初回起動時（User経由）:
  フロント: liff.getProfile() でdisplayName取得
  → getUser呼び出し時にdisplayNameを送信
  → UserService.getOrCreateUser(userId, displayName)
    → 既存レコードのdisplayNameを更新
```

### フロー6: 開発用API

```
デモ撮影時:
  POST /dev/trigger-night-notification (L-12, X-Dev-Secret検証)
  → NotificationService.sendNightNotification（即時実行）
  
  POST /dev/force-appear (L-12)
  → UserNamekemonoに直接追加（pendingスキップ）
  
  POST /dev/reset (L-12)
  → Users, UserNamekemono, UserProgress, Letters, OtukaiHistory を全削除
```
