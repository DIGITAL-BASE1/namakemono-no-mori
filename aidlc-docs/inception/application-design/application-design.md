# Application Design — ナマケモノの森

## 設計概要

本ドキュメントは「ナマケモノの森」MVP v1のアプリケーション設計を統合的にまとめたものです。

---

## 技術スタック

| レイヤー | 技術 | 備考 |
|---|---|---|
| フロントエンド | React + Vite + TypeScript + LIFF SDK | SPA、S3+CloudFrontで配信 |
| バックエンド | AWS Lambda (Python) | 機能単位で分割（12関数） |
| データベース | Amazon DynamoDB | エンティティ別テーブル（5テーブル） |
| AI | Amazon Bedrock | おつかい生成・応答・達成判定（マルチモーダル対応モデル） |
| インフラ | AWS CDK (TypeScript) | IaC |
| 通知 | LINE Messaging API | Push通知 |
| 認証 | LIFF | LINE Front-end Framework |
| スケジュール | Amazon EventBridge | 通知・おつかい配信 |
| 画像保存 | Amazon S3 | おつかい報告画像 |
| 外部API | 天気API、祝日API | おつかいガードレール |

---

## 共通設計方針

### 認証
- 全API（L-01〜L-07）は `Authorization: Bearer <LIFF ID Token>` ヘッダー必須
- Lambda内で `M-06 auth.verifyIdToken()` によってLINE userIdを抽出
- L-11 line-webhook は LINE署名（X-Line-Signature）で検証
- L-12 dev-api は `X-Dev-Secret` ヘッダーで共有シークレット検証

### エラー表現
- 全APIは失敗時 `{ code: string, message: string }` 形式を返す
- 具体的なエラーコード一覧は Functional Design で定義

### タイムゾーン
- 「1日1回」「翌朝」等の日付判定は **JST 0:00 基準**
- DynamoDBへの保存はUTC ISO8601形式（表示・判定時にJSTへ変換）
- 翌朝出現の `appearAt` は **JST 8:00** に固定

### 冪等性
- `purchaseItem`, `harvestCoins`, `submitSwipeResult`, `giftItem` は二重送信対策が必要
- 具体的な冪等キー設計は Functional Design で定義

---

## 画面構成（7画面 + タブナビゲーション）

| 画面 | 概要 | タブ |
|---|---|---|
| ForestPage | メイン画面。森・ナマケモノ・収穫・贈り物 | 森 |
| SwipeReportPage | スワイプ報告 | スワイプ |
| ShopPage | アイテム購入 | ショップ |
| EncyclopediaPage | 図鑑 | 図鑑 |
| LetterCollectionPage | 手紙コレクション | その他 |
| DiaryPage | 森の日記 | その他 |
| OnboardingDialog | 初回絵本（オーバーレイ） | — |

**ナビゲーション**: 下部タブバー（森、スワイプ、ショップ、図鑑、その他）
**所持品**: 専用画面なし。ダイアログ（InventoryDialog）で表示。

---

## バックエンド構成（12 Lambda）

### API Lambda（8関数）— API Gateway経由

| Lambda | 機能 |
|---|---|
| user-api | ユーザー取得、初回判定、放置ボーナス自動付与 |
| swipe-api | スワイプ報告、NMK計算、出現条件 |
| forest-api | 森の状態、pending実体化、日次収穫、つぶやき |
| shop-api | ショップ、購入、所持品取得 |
| affinity-api | 贈呈、親密度、手紙取得 |
| encyclopedia-api | 図鑑データ |
| diary-api | 森の日記 |
| dev-api | 開発用（通知/おつかい手動発火、強制出現、データリセット） |

### スケジュール Lambda（3関数）— EventBridge経由

| Lambda | トリガー | 機能 |
|---|---|---|
| night-notification | 21:00 JST | 夜のスワイプ促し通知 |
| morning-notification | 8:00 JST | 朝の新着ナマケモノ通知（pendingAppearances対象） |
| otukai-scheduler | 10:00/13:00/16:00 JST | おつかい配信 |

### Webhook Lambda（1関数）— LINE Webhook経由

| Lambda | 機能 |
|---|---|
| line-webhook | follow/unfollow処理、おつかい報告受信（テキスト/画像）、画像S3保存、Bedrock呼び出し、LINE Reply。タイムアウト30秒 |

---

## DynamoDBテーブル設計（5テーブル）

| テーブル | PK | SK | 内容 |
|---|---|---|---|
| Users | userId | — | ユーザー情報、NMK残高、設定、notificationEnabled、**currentOtukaiId**（進行中おつかい参照）、displayName |
| UserNamekemono | userId | namekemonoId | 所持ナマケモノ、親密度、収穫日時 |
| UserProgress | userId | progressType#key | スワイプ履歴、出現条件進行、所持品、pendingAppearances |
| Letters | userId | letterId | 手紙コレクション |
| OtukaiHistory | userId | otukaiId | おつかい履歴、会話ログ、日記、画像S3 URI、status（in_progress/completed） |

> 〜5ユーザー規模のため、GSI不要。Scan許容。

---

## 翌朝出現ロジック（pendingAppearances方式）

即時出現は**オンボーディング時のネムリンのみ**（特別扱い）。他のナマケモノは全て翌朝出現。

### フロー

```
1. submitSwipeResult
   - GameCoreService.checkAppearanceConditions で条件達成検出
   - UserProgress に pending#{namekemonoId} として保留登録
     （appearAt: 翌朝JST 8:00）
   - レスポンス: 「森に足跡が…」(即時出現はしない)

2. morning-notification（8:00 JST EventBridge）
   - 今朝実体化されるpendingを持つユーザーに通知送信
   - 「森に新しい仲間が来ているよ！🦥」

3. getForestState（ユーザーがアプリを開いた時）
   - GameCoreService.resolvePendingAppearances
     - appearAt <= now のpendingを UserNamekemono に実体化
     - pendingを削除
   - 新着ナマケモノ配列をレスポンスに含める
   - フロントで出現演出再生
```

---

## マスターデータ管理

**方式**: JSONファイルとしてLambdaにバンドル

| ファイル | 内容 |
|---|---|
| `cards.json` | ダメカード30種（ID、名前、カテゴリ） |
| `namekemono.json` | ナマケモノ定義（ID、名前、カテゴリ、出現条件、つぶやき5種、ヒント） |
| `items.json` | ショップアイテム（ID、名前、カテゴリ、価格、親密度効果） |
| `game-config.json` | ゲームバランスパラメータ（NMK獲得量、コスト、閾値等） |
| `letters.json` | 手紙テンプレート（ナマケモノID、親密度レベル、テキスト） |

---

## EventBridgeスケジュール（5ルール）

| ルール | Cron | Lambda | 実行タイミング |
|---|---|---|---|
| NightNotification | cron(0 12 * * ? *) | night-notification | UTC毎日12:00実行 = JST21:00 |
| MorningNotification | cron(0 23 * * ? *) | morning-notification | UTC毎日23:00実行 = 翌日JST8:00 |
| OtukaiSchedule10 | cron(0 1 * * ? *) | otukai-scheduler | UTC毎日01:00実行 = JST10:00 |
| OtukaiSchedule13 | cron(0 4 * * ? *) | otukai-scheduler | UTC毎日04:00実行 = JST13:00 |
| OtukaiSchedule16 | cron(0 7 * * ? *) | otukai-scheduler | UTC毎日07:00実行 = JST16:00 |

> **Note**: EventBridge cronは常にUTC基準。表の「実行タイミング」列はUTCとJSTの対応を明示。
> 日次収穫はEventBridgeではなく、アプリ初回起動時にオンデマンド計算。
> MVP v2で通知時刻の個人設定対応時、NightNotificationを毎時実行に変更予定（v1では過剰な先回り設計を避け、固定1ルールで対応）。

---

## 状態管理（フロントエンド）

**方式**: React標準（useState + useContext）

| Context | 管理する状態 |
|---|---|
| UserContext | ユーザー情報、NMK残高、初回フラグ |
| ForestContext | 森の状態、ナマケモノ一覧、収穫可否 |
| InventoryContext | 所持品一覧 |

---

## LINE連携の前提条件

| 項目 | 内容 |
|---|---|
| 友だち追加 | LIFFアプリ利用の前提。友だち追加でWebhookのfollowイベント発火 → ユーザー登録（displayName空欄） + notificationEnabled=true |
| LINE userId取得 | LIFF SDK `liff.getProfile()` で取得。全APIのユーザー識別子として使用 |
| displayName取得 | LIFF初回起動時に `liff.getProfile()` でフロントが取得し、getUser経由でサーバーに渡しDBに保存。handleFollow時点では空欄（line-client依存を回避） |
| Push通知の前提 | 友だち追加済み + LINE userId取得済み + notificationEnabled=true |
| Webhook署名検証 | LINE Webhook受信時にX-Line-Signatureヘッダーで署名検証必須 |
| Push失敗時 | ブロック等でPush失敗した場合はログ記録のみ（リトライなし） |
| Unfollow処理 | handleUnfollowで `Users.notificationEnabled = false` にセット |
| 画像メッセージ | `GET /v2/bot/message/{messageId}/content` で取得し、S3に保存後にBedrockへマルチモーダルで渡す |
| おつかい返信の特定 | `Users.currentOtukaiId` で進行中おつかいを追跡。generateOtukai時にセット、会話終了時にクリア |

---

## 贈り物UXフロー

```
ショップ画面 → アイテム選択 → 購入確認 → 購入完了
                                              │
                                              ▼
森画面 → ナマケモノタップ → InventoryDialog表示
         → アイテム選択 → 贈る → 親密度UP演出
```

所持品は専用画面を持たず、ナマケモノをタップした際にInventoryDialogとして表示。

---

## 開発用API（L-12 dev-api）

| メソッド | 用途 |
|---|---|
| triggerNightNotification | 夜通知を任意タイミングで発火 |
| triggerMorningNotification | 朝通知を任意タイミングで発火 |
| triggerOtukai | おつかい配信を任意タイミングで発火 |
| forceAppearNamekemono | 特定のナマケモノを即時出現 |
| resetUserData | ユーザーデータを初期状態にリセット |

認証: `X-Dev-Secret` ヘッダーによる共有シークレット検証。動画撮影時のタイミング制御に使用。

---

## 画像アセット管理

**配置**: フロントエンドにバンドル（`frontend/src/assets/`）→ CloudFrontでキャッシュ配信

| カテゴリ | 内容 | 枚数目安 |
|---|---|---|
| ナマケモノ | 各キャラの立ち絵（1枚/キャラ）、シルエット | 5体 × 2 = 10枚 |
| アイテム | ショップアイテムのアイコン | 10〜15枚 |
| 森の背景 | 成長段階ごとの背景（4段階） | 4枚 |
| UI | タブアイコン、ボタン、コイン等 | 10〜15枚 |

**アニメーション方式**: 静止画 + CSSアニメーション

| 表現 | CSS手法 |
|---|---|
| ナマケモノの呼吸感 | `transform: translateY()` のゆっくりループ |
| もぐもぐ揺れ | `transform: rotate()` の微振動 |
| zzz表示 | opacity fade in/out |
| 葉っぱの揺れ | `transform: rotate()` ランダム遅延 |
| 光の粒 | opacity + scale のfade |

> ナマケモノは怠けているので「ほぼ動かない」のが世界観に合致。1キャラ1枚の静止画で十分な表現力を確保。

---

## API仕様書（OpenAPI）

- **配置**: `backend/openapi.yaml`
- **所有**: U2 Backend Unitが作成・管理（Day 1に初版を作成）
- **役割**: FE-BE間のSingle Source of Truth（SSOT）。フロントエンド・バックエンド並列開発のための契約として機能
- **詳細化**: 詳細なスキーマ定義（リクエスト/レスポンスの型、エラーコード、冪等キー等）はFunctional Design（CONSTRUCTION phase）で詳細化

---

## 詳細設計への引き継ぎ事項

以下はFunctional Design（CONSTRUCTION phase）で詳細化する：

- NMK計算の具体的な数式・パラメータ
- 出現条件の具体的な閾値設計
- 親密度ポイントの具体的な計算ロジック
- おつかいAIのプロンプト設計
- ダメカードの選出アルゴリズム（ランダム性の制御）
- 手紙テンプレートの具体的な内容
- エラーコード一覧
- 冪等キー設計（purchaseItem / harvestCoins / submitSwipeResult / giftItem）
