# Application Design 統合ドキュメント — ナマケモノの森 MVP v1

---

## 技術スタック

| レイヤー | 技術 |
|---|---|
| フロント | React + Vite + LIFF SDK |
| ホスティング | S3 + CloudFront |
| 認証 | LINEログイン（LIFF SDK自動処理） |
| 通知 | LINE Messaging API（Bot） |
| バックエンド | Lambda + API Gateway |
| DB | DynamoDB |
| インフラ管理 | CDK |
| スケジューラー | EventBridge |
| 開発フロー | ビルド → S3デプロイ → CloudFront経由でiPhone確認 |

---

## アーキテクチャ概要

```
LINE App (LIFF Browser)
    │
    │ HTTPS
    ▼
CloudFront → S3 (React SPA)
    │
    │ API呼び出し
    ▼
API Gateway → Lambda Functions → DynamoDB
    │
    │ Push通知
    ▼
LINE Messaging API → LINE App (通知)

EventBridge (スケジューラー) → Lambda (通知送信 / 到着チェック)
```

---

## 画面構成（SPA）

| 画面 | 種別 | 主な機能 |
|---|---|---|
| 森（メイン） | Page | ナマケモノ表示、コイン収穫、タップ操作 |
| スワイプ報告 | Page | ダメカード10枚スワイプ、途中離脱 |
| ショップ | Page | アイテム一覧、購入 |
| 贈り物 | Page | ナマケモノ選択 → アイテム贈呈 |
| 図鑑 | Page | シルエット/実体、ヒント、進捗 |
| 手紙 | Page | 手紙コレクション閲覧 |
| オンボーディング | Dialog（森の上にオーバーレイ） | 絵本 → スワイプ体験 → 召喚 → ショップ → 贈り物 |

**ルーティング**: React Router（クライアントサイド）。LIFF内ブラウザでのページ遷移問題を回避。

---

## API 機能一覧

※ 具体的なエンドポイントパス、リクエスト/レスポンススキーマはCONSTRUCTION phaseで詳細化

| カテゴリ | 機能 | 目的 |
|---|---|---|
| ユーザー管理 | ユーザー登録 | 初回LIFF起動時にLINE IDを取得・保存（Push通知の前提） |
| ユーザー管理 | ユーザー状態取得 | コイン、ナマケモノ、親密度、図鑑進捗の一括取得 |
| ゲームコア | スワイプ報告送信 | カード結果受付、コイン付与、出現条件進行 |
| ゲームコア | コイン収穫 | 日次収穫（複数日蓄積対応） |
| ゲームコア | 出現条件チェック（バッチ） | 条件達成ナマケモノの到着処理 |
| ショップ | アイテム購入 | コイン消費 → アイテム付与 |
| 親密度 | 贈り物 | アイテム贈呈 → 親密度更新 → 手紙解放判定 |
| 図鑑 | 図鑑データ取得 | 発見状況、ヒント解放状態 |
| 通知 | 夜のスワイプ通知 | ナマケモノ固有セリフでPush送信 |
| 通知 | 朝の到着通知 | 新しい仲間が来た朝にPush送信 |
| 通知 | 開発用手動トリガー | 動画撮影用。本番では無効化 |
| LINE連携 | Webhook受信 | LINEプラットフォームからのイベント受信 |

---

## データ保存

### DynamoDB に保存するデータ

| データ種別 | 内容 |
|---|---|
| ユーザー基本データ | コイン残高、設定、所持アイテム、LINE ID |
| ナマケモノ状態 | 発見状態、親密度、条件進捗（ユーザーごと） |
| スワイプ履歴 | 日次の報告結果（出現条件判定に使用） |
| 手紙データ | 受け取った手紙、既読状態 |

### マスターデータ（JSONバンドル）

MVP v1ではLambdaにJSONファイルとしてバンドル。DB不要。変更頻度が低く5体分のデータ量なので十分。

| データ | 内容 |
|---|---|
| ナマケモノ定義 | 5体の名前、カテゴリ、出現条件、セリフ5種、好き嫌い |
| ダメカード定義 | 30枚のカテゴリ、テキスト |
| ショップアイテム定義 | アイテム名、価格、効果 |
| 手紙定義 | ナマケモノごとの手紙テキスト |

※ 具体的なテーブル設計（PK/SK、属性定義）はCONSTRUCTION phaseで詳細化

---

## マスターデータ（JSONバンドル）

MVP v1ではLambdaにJSONファイルとしてバンドル。DB不要。

| ファイル | 内容 |
|---|---|
| `namekemonos.json` | 5体の定義（名前、カテゴリ、条件、セリフ5種、好き嫌い） |
| `cards.json` | 30枚のダメカード定義（カテゴリ、テキスト） |
| `items.json` | ショップアイテム定義（名前、価格、効果） |
| `letters.json` | 手紙テキスト定義 |

---

## 通知フロー

### 前提：LINE ID登録
- ユーザーが初めてLIFFアプリを開いた時に `liff.getProfile()` でLINEユーザーIDを取得
- バックエンドに送信してDynamoDBに保存
- **この登録が完了していないユーザーにはPush通知を送れない**
- オンボーディング完了 = LINE ID登録完了 として扱う

### 夜のスワイプ通知
```
EventBridge (毎晩トリガー) → sendNightNotification Lambda
  → DynamoDBから登録済みユーザー一覧取得
  → 各ユーザーの森のナマケモノからランダム1匹選出
  → masterDataからセリフ選択
  → LINE Messaging API Push送信（LINEユーザーIDを使用）
    メッセージ: "{ナマケモノ名}「{セリフ}」"
    アクション: LIFFリンク（スワイプ画面に遷移）
```

### 朝の到着通知
```
EventBridge (毎朝トリガー) → checkNewArrivals Lambda
  → 条件達成済みユーザーを判定
  → ナマケモノを "pending" → "arrived" に更新
  → sendMorningNotification Lambda
    → LINE Messaging API Push送信（LINEユーザーIDを使用）
      メッセージ: "森に新しい仲間が来ているよ！見に行こう🦥"
      アクション: LIFFリンク（森画面に遷移）
```

### 開発用手動トリガー
```
開発用API呼び出し
  → 指定ユーザーに即座に通知送信
  → 動画撮影時に使用。本番では無効化
```

---

## セキュリティ

| 項目 | 方針 |
|---|---|
| API認証 | LIFF SDKから取得したアクセストークンをAuthorizationヘッダーに付与。Lambda側でLINE APIにトークン検証リクエストを送り、userIdの正当性を確認 |
| CORS | CloudFrontドメインのみ許可 |
| Webhook検証 | LINE署名検証（X-Line-Signature） |
| 開発用API | 環境変数フラグで本番では無効化 |
| マスターデータ検証 | カード選出はフロントでランダム化するが、報酬・条件判定はサーバー側でIDベースで検証。フロントから送られたcardId/itemIdの妥当性をサーバーで確認 |

## LINE連携の成立条件

### Push通知を送るための前提
1. **LINEユーザーIDの取得**: 初回LIFF起動時に`liff.getProfile()`でuserIdを取得し、サーバーに送信・保存
2. **アクセストークン検証**: サーバー側でLIFFアクセストークンを検証し、取得したuserIdが信頼できることを確認
3. **LINE公式アカウントのフォロー**: Push通知はMessaging APIのチャネルに紐づくBotから送信。ユーザーがこのBotを友だち追加している必要がある
4. **チャネル/Provider設定**: LINE DevelopersコンソールでMessaging APIチャネルとLINE Loginチャネル（LIFF用）を同一Provider内に作成

### Push失敗時の扱い（MVP v1）
- ブロックされた場合: エラーを無視（MVP v1ではリトライ不要）
- 友だち追加されていない場合: エラーを無視
- MVP v1はデモ用なので、失敗時のリカバリーは実装しない

### 技術スパイク（最優先実装）
Units Generation後、最初のCode Generationで以下の縦串を通す：
1. LIFF起動 → `liff.getProfile()` でuserId取得
2. Lambda側でLIFFアクセストークン検証
3. DynamoDBにuserId保存
4. LINE Messaging APIでPush送信（テストメッセージ）

この縦串が通れば、残りの機能は比較的作りやすい。

---

## 開発環境構成

| 環境 | 用途 | URL |
|---|---|---|
| dev | 開発・テスト | `https://dev.d1xxxxx.cloudfront.net` |
| prod | 動画撮影・プレゼン | `https://prod.d2xxxxx.cloudfront.net` |

- LIFF IDのエンドポイントURLはdev/prodで切り替え
- CDKのスタック分離で環境を管理
- デプロイ: `cdk deploy --context env=dev` / `cdk deploy --context env=prod`

---

## 詳細設計への引き継ぎ事項

以下はすべてCONSTRUCTION phase（Functional Design + Code Generation）で詳細化：
- APIエンドポイント設計（パス、HTTPメソッド、リクエスト/レスポンススキーマ）
- DynamoDBテーブル設計（PK/SK、属性定義、インデックス）
- 出現条件の具体的な判定ロジック（conditionEngine）
- ナマケコインの計算式（coinCalculator）
- 親密度の増減量テーブル（affinityManager）
- 途中離脱ボーナスの計算式
- 初回オンボーディングの状態管理フロー
- フロントエンドのコンポーネント分割（UIコンポーネント粒度）
- 具体的なメソッドシグネチャ・型定義
