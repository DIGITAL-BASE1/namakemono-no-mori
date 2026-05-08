# Component Methods

## 共通前提

### 認証
- `L-01〜L-07` の全APIは `Authorization: Bearer <LIFF ID Token>` ヘッダー必須
- Lambda内で `M-06 auth.verifyIdToken()` によってLINE userIdを抽出し、以降の処理で使用
- `L-12 dev-api` は開発用のため共有シークレット（ヘッダー）で認証
- `L-11 line-webhook` は LINE署名（X-Line-Signature）で検証

### エラー表現
- 全APIは失敗時 `{ code: string, message: string }` 形式を返す
- 具体的なエラーコード一覧は Functional Design で定義

### タイムゾーン
- 「1日1回」「翌朝」等の日付判定は JST 0:00 基準
- DynamoDBへの保存はUTC ISO8601形式（表示・判定時にJSTへ変換）

### 冪等性
- `purchaseItem`, `harvestCoins`, `submitSwipeResult`, `giftItem` は二重送信対策が必要
- 具体的な冪等キー設計は Functional Design で定義

> **Note**: 詳細なビジネスルール（NMK計算式、出現条件の閾値等）はFunctional Design（CONSTRUCTION phase）で定義します。

---

## バックエンド API メソッド

### L-01: user-api

| メソッド | 概要 | Input | Output |
|---|---|---|---|
| `getUser` | ユーザー情報取得 + 初回起動判定 + **放置ボーナス自動付与** + 最終ログイン日時更新 + **displayName更新**（初回時） | (authから取得したuserId) + displayName（LIFF liff.getProfile()から） | ユーザー情報、NMK残高、初回フラグ、idleBonus(days, nmk)、日次収穫可否 |
| `completeOnboarding` | オンボーディング完了記録 | — | 更新結果 |

**備考**: `registerUser`は削除。新規ユーザー登録は以下の2経路のみ：
- LIFF起動時の`getUser`内部で`getOrCreateUser(userId, displayName)`（displayNameあり）
- LINE友だち追加時の`handleFollow`で`getOrCreateUser(userId, displayName=null)`（displayName空欄、後でLIFF起動時に更新）

**備考**: `claimIdleBonus`は削除。`getUser`呼び出し時点で放置ボーナスは**自動付与**（NMK加算完了）。返却値の`idleBonus`はフロント演出用の情報であり、付与の可否には影響しない。

---

### L-02: swipe-api

| メソッド | 概要 | Input | Output |
|---|---|---|---|
| `startSwipe` | スワイプセッション開始。ダメカードを選出して返す（stateless）。通常10枚、初回（isFirstTime=true）は5枚 | isFirstTime: boolean | cards[{cardId, name, category}]（通常10枚 / 初回5枚。初回は1枚目=ねむねむ強制、2枚目=もぐもぐ強制） |
| `submitSwipeResult` | スワイプ結果送信（完了 or 途中離脱）。全枚数の結果を一括送信（通常10枚 / 初回5枚） | results[{cardId, swiped: boolean or null}], abandoned: boolean | 獲得NMK、出現条件達成フラグ（翌朝出現）、新残高 |

**備考**: sessionIdは廃止。stateless運用。クライアントがカード配列を保持し、submit時にcardIdの妥当性をサーバーで検証（masterDataに存在するか）。

**備考**: `submitSwipeResult`は**即時出現のナマケモノを返さない**（オンボーディング時のネムリンを除く）。条件達成時は`UserProgress`の`pendingAppearances`に保留登録し、翌朝`getForestState`で実体化する。

---

### L-03: forest-api

| メソッド | 概要 | Input | Output |
|---|---|---|---|
| `getForestState` | 森の状態取得 + **pendingAppearancesの実体化**（appearAt <= now のものをUserNamekemonoに追加） | — | ナマケモノ一覧（位置・状態）、森の成長段階、新着ナマケモノ配列（今回実体化したもの）、収穫可否 |
| `harvestCoins` | 日次収穫実行 | — | 収穫NMK（匹ごとの内訳）、親密度ボーナス、新残高 |
| `getWhisper` | ナマケモノのつぶやき取得（5種類からランダム） | namekemonoId | つぶやきテキスト |

---

### L-04: shop-api

| メソッド | 概要 | Input | Output |
|---|---|---|---|
| `getShopItems` | ショップアイテム一覧取得 | — | アイテム一覧、所持NMK |
| `purchaseItem` | アイテム購入 | itemId | 残NMK、所持品更新結果 |
| `getInventory` | 所持品一覧取得（shop-apiに集約） | — | アイテム一覧（所持数） |

---

### L-05: affinity-api

| メソッド | 概要 | Input | Output |
|---|---|---|---|
| `giftItem` | アイテム贈呈（所持品減算はShopService経由） | namekemonoId, itemId | 親密度変動、現在の親密度レベル、手紙解放フラグ、おつかい解放フラグ、新手紙（解放時） |
| `getLetters` | 手紙コレクション取得 | — | 手紙一覧（ナマケモノ名、テキスト、日付） |

---

### L-06: encyclopedia-api

| メソッド | 概要 | Input | Output |
|---|---|---|---|
| `getEncyclopedia` | 図鑑データ取得 | — | 全16枠の状態（発見済み/シルエット）、ヒント（レベル1 or 2）、出現条件達成率 |

---

### L-07: diary-api

| メソッド | 概要 | Input | Output |
|---|---|---|---|
| `getDiaryEntries` | 森の日記取得 | limit, offset | 日記エントリ一覧（日付、ナマケモノ名、テキスト） |

---

### L-08: night-notification

| メソッド | 概要 | Input | Output |
|---|---|---|---|
| `sendNightNotifications` | 夜のスワイプ促し通知を全対象ユーザー（notificationEnabled=true）に送信 | EventBridgeイベント | 送信結果（成功/失敗数） |

---

### L-09: morning-notification

| メソッド | 概要 | Input | Output |
|---|---|---|---|
| `sendMorningNotifications` | 当朝実体化されるpendingAppearancesを持つユーザーに通知送信 | EventBridgeイベント | 送信結果（成功/失敗数） |

---

### L-10: otukai-scheduler

| メソッド | 概要 | Input | Output |
|---|---|---|---|
| `processOtukaiSchedule` | おつかい対象者判定 → Bedrockでおつかい生成 → LINE Push | EventBridgeイベント（時刻情報） | 配信結果 |

---

### L-11: line-webhook

| メソッド | 概要 | Input | Output |
|---|---|---|---|
| `handleWebhook` | LINE Webhookイベント受信。署名検証後にイベント種別でルーティング | LINE Webhook Event | 200 OK（即返し） |
| `handleFollow` | 友だち追加イベント処理 | Follow Event | ユーザー登録 or 再有効化（notificationEnabled=true） |
| `handleUnfollow` | ブロック（unfollow）イベント処理 | Unfollow Event | `Users.notificationEnabled=false` |
| `handleOtukaiReply` | おつかい報告処理。テキスト/画像を処理してBedrockで応答生成 | メッセージイベント | LINE Reply送信 |

**画像メッセージ処理フロー**:
1. `M-02 line-client.fetchMessageContent(messageId)` で画像バイト取得
2. `M-05 s3-client.putObject` でS3保存（キー: `otukai/{userId}/{otukaiId}/{timestamp}.jpg`）
3. 会話ログにS3 URIを付与
4. Bedrock呼び出し時にマルチモーダルモデルで画像+テキストを渡す

---

### L-12: dev-api（開発用）

| メソッド | 概要 | Input | Output |
|---|---|---|---|
| `triggerNightNotification` | 夜通知を任意タイミングで発火 | userId | 送信結果 |
| `triggerMorningNotification` | 朝通知を任意タイミングで発火 | userId | 送信結果 |
| `triggerOtukai` | おつかい配信を任意タイミングで発火 | userId, namekemonoId | 配信結果 |
| `forceAppearNamekemono` | 特定のナマケモノを即時出現させる | userId, namekemonoId | 出現結果 |
| `resetUserData` | ユーザーデータを初期状態にリセット | userId | リセット結果 |

認証: 共有シークレット（`X-Dev-Secret`ヘッダー）で保護。

---

## 共通モジュール メソッド

### M-02: line-client

| メソッド | 概要 |
|---|---|
| `pushMessage(userId, messages)` | LINE Push送信 |
| `replyMessage(replyToken, messages)` | LINE Reply送信 |
| `fetchMessageContent(messageId)` | 画像バイト取得（`GET /v2/bot/message/{messageId}/content`） |
| `verifySignature(body, signature)` | Webhook署名検証 |

### M-05: s3-client

| メソッド | 概要 |
|---|---|
| `putObject(key, body, contentType)` | S3にオブジェクトをアップロード |
| `getObject(key)` | S3からオブジェクト取得 |
| `getPresignedUrl(key, expiresIn)` | Presigned URL発行（必要時） |

### M-06: auth

| メソッド | 概要 |
|---|---|
| `verifyIdToken(idToken)` | LIFF ID tokenを検証し、LINE userIdを返す |

---

## フロントエンド 主要メソッド

### API通信層

| メソッド | 概要 |
|---|---|
| `api.getUser()` | ユーザー情報取得（初回判定・放置ボーナス自動付与含む） |
| `api.startSwipe(isFirstTime)` | スワイプセッション開始（10枚のカード取得） |
| `api.submitSwipe(results, abandoned)` | スワイプ結果送信 |
| `api.getForest()` | 森の状態取得（新着ナマケモノ実体化含む） |
| `api.harvestCoins()` | 日次収穫 |
| `api.getShopItems()` | ショップ一覧 |
| `api.purchaseItem(itemId)` | アイテム購入 |
| `api.getInventory()` | 所持品取得 |
| `api.giftItem(namekemonoId, itemId)` | アイテム贈呈 |
| `api.getEncyclopedia()` | 図鑑取得 |
| `api.getLetters()` | 手紙コレクション取得 |
| `api.getDiary(limit, offset)` | 森の日記取得 |
| `api.getWhisper(namekemonoId)` | つぶやき取得 |
| `api.completeOnboarding()` | オンボーディング完了 |

### 状態管理（useContext）

| Context | 管理する状態 |
|---|---|
| `UserContext` | ユーザー情報、NMK残高、初回フラグ |
| `ForestContext` | 森の状態、ナマケモノ一覧、収穫可否 |
| `InventoryContext` | 所持品一覧 |
