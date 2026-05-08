# Unit of Work — ナマケモノの森

## ユニット構成概要

4ユニット構成（FE / BE / Infra / Assets）。全ユニット並列開発。

| ユニット | 担当 | 主責務 |
|---|---|---|
| **U1: Frontend** | メンバーA | LIFFアプリ全画面、UI/UXコンポーネント、状態管理、API通信層 |
| **U2: Backend** | メンバーB | Lambda関数（12本）、サービス層、共通モジュール、DynamoDBアクセス、LINE連携、Bedrock連携 |
| **U3: Infrastructure** | メンバーB（BE兼務） | AWS CDK、S3+CloudFront、API Gateway、Lambda、DynamoDB、EventBridge、IAM |
| **U4: Assets** | メンバーC（海外出張中） | ナマケモノ立ち絵、シルエット、アイテムアイコン、森の背景、UI素材 |

---

## U1: Frontend Unit

### 責務
- LIFFアプリ全体の画面実装（7画面）
- UIコンポーネント（10コンポーネント）
- 状態管理（Context API）
- API通信層（axios等）
- CSSアニメーション（呼吸感、揺れ、zzz）
- オンボーディング絵本UI
- スワイプインタラクション

### 成果物
- `frontend/` ディレクトリ以下
  - `src/pages/` — 画面コンポーネント
  - `src/components/` — 再利用UIコンポーネント
  - `src/contexts/` — UserContext, ForestContext, InventoryContext
  - `src/api/` — API通信層
  - `src/hooks/` — カスタムフック
  - `src/styles/` — CSS、アニメーション定義
  - `src/assets/` — Uniq4からの画像を配置
- Vite設定ファイル、TypeScript設定
- LIFF初期化コード

### 担当者: メンバーA

### 依存
- **U2 Backend** のAPI仕様（OpenAPI）
- **U4 Assets** の画像素材（プレースホルダーでも開発可能）
- **U3 Infrastructure** のCloudFront配信URL（最終デプロイ時）

### スケジュール目安
- Day 1-2: プロジェクトセットアップ、LIFF初期化、ルーティング
- Day 3-5: 森画面、スワイプ画面、ショップ画面
- Day 6-8: 図鑑、手紙、森の日記、オンボーディング
- Day 9-11: アニメーション、画像差し替え、統合テスト
- Day 12-14: デモシナリオ磨き込み、動画用調整

---

## U2: Backend Unit

### 責務
- Lambda関数 12本の実装
  - API Lambda: user-api, swipe-api, forest-api, shop-api, affinity-api, encyclopedia-api, diary-api, dev-api
  - Schedule Lambda: night-notification, morning-notification, otukai-scheduler
  - Webhook Lambda: line-webhook
- サービス層（6サービス: GameCore, Shop, Affinity, Notification, Otukai, User）
- 共通モジュール（db-client, line-client, master-data, game-config, s3-client, auth）
- DynamoDBテーブルへのCRUD
- LINE Messaging API連携（Push/Reply/画像取得/署名検証）
- Bedrock連携（おつかい生成・応答・達成判定）
- マスターデータJSON（cards, namekemono, items, game-config, letters）
- ゲームロジック（NMK計算、出現条件、親密度、pendingAppearances等）

### 成果物
- `backend/` ディレクトリ以下
  - `backend/functions/` — Lambda関数実装
  - `backend/services/` — サービス層
  - `backend/modules/` — 共通モジュール
  - `backend/master_data/` — JSONファイル
  - `backend/tests/` — ユニットテスト
  - `requirements.txt`, `pyproject.toml` 等
- OpenAPI仕様書 `backend/openapi.yaml`（並列開発の要）

### 担当者: メンバーB

### 依存
- **U3 Infrastructure** のDynamoDBテーブル名、S3バケット名等の環境変数
- **U4 Assets** には依存しない

### スケジュール目安
- Day 1: OpenAPI仕様書作成（Frontendと共有）
- Day 2-4: 共通モジュール、マスターデータ、user-api/swipe-api/forest-api
- Day 5-7: shop-api/affinity-api/encyclopedia-api/diary-api
- Day 8-10: 通知Lambda、おつかいLambda（Bedrock連携）、Webhook Lambda
- Day 11-12: dev-api、テスト、統合確認
- Day 13-14: 動画撮影用調整

---

## U3: Infrastructure Unit

### 責務
- AWS CDK（TypeScript）によるインフラ定義
- S3バケット + CloudFrontディストリビューション（SPA配信）
- API Gateway REST API（Lambda統合）
- DynamoDBテーブル 5つ
- EventBridgeルール 5本
- Lambda関数のデプロイ
- IAMロール（Lambda実行ロール、Bedrock/S3/DynamoDBアクセス権限）
- CloudWatch Logs
- Secrets Manager（LINE Channel Access Token等）

### 成果物
- `infra/` ディレクトリ以下
  - `infra/lib/` — CDKスタック定義
    - `frontend-stack.ts` — S3+CloudFront
    - `api-stack.ts` — API Gateway + Lambda
    - `database-stack.ts` — DynamoDB
    - `schedule-stack.ts` — EventBridge
  - `infra/bin/` — CDKアプリエントリポイント
  - `cdk.json`, `package.json`, `tsconfig.json`
- デプロイ手順書

### 担当者: メンバーB（BEと兼務）

### 依存
- **U2 Backend** のLambdaコードパッケージ
- **U1 Frontend** のビルド成果物（S3デプロイ用）

### スケジュール目安
- Day 1-2: CDKスタック骨格、DynamoDB、S3+CloudFront
- Day 3-5: API Gateway + Lambda統合、初回デプロイ
- Day 6-8: EventBridge、Bedrock IAM、Secrets Manager
- Day 9-11: デプロイスクリプト整備、CI/CD（最小限）
- Day 12-14: 動画撮影用環境の安定化

---

## U4: Assets Unit

### 責務
- ナマケモノ立ち絵（5体、実体 + シルエット）
- アイテムアイコン（10〜15枚）
- 森の背景（4段階: 小さな/にぎやかな/豊かな/伝説の森）
- UI素材（タブアイコン、ボタン、コイン、通知バッジ等）
- オンボーディング絵本イラスト（3ページ）
- トンマナ維持、キャラ統一感

### 成果物
- `assets/` ディレクトリ以下
  - `assets/namekemono/` — キャラ画像
  - `assets/items/` — アイテムアイコン
  - `assets/forest/` — 森背景
  - `assets/ui/` — UI素材
  - `assets/onboarding/` — 絵本イラスト
- 最終成果物を `frontend/src/assets/` にコミット

### 担当者: メンバーC（海外出張中）

### 依存
- なし（独立して作業可能）

### スケジュール目安（出張制約を考慮）
- Day 1-3: ネムリン、モグモグ（初期Day1-2で使うキャラ優先）
- Day 4-7: アトデーノ、スクロン、メンドクサウルス
- Day 8-10: アイテムアイコン、森背景
- Day 11-12: UI素材、オンボーディング絵本
- Day 13-14: 調整、微修正

**リスク緩和**: Frontendはプレースホルダー画像で開発可能。素材が届き次第差し替え。

---

## リポジトリ構成（モノレポ）

```
namekemono-no-mori/
├── frontend/              # U1: React + Vite + TypeScript
│   ├── src/
│   │   ├── pages/
│   │   ├── components/
│   │   ├── contexts/
│   │   ├── api/
│   │   ├── hooks/
│   │   ├── styles/
│   │   └── assets/        # U4から配置
│   ├── public/
│   ├── package.json
│   ├── vite.config.ts
│   └── tsconfig.json
│
├── backend/               # U2: AWS Lambda (Python)
│   ├── functions/         # Lambda関数
│   ├── services/          # サービス層
│   ├── modules/           # 共通モジュール
│   ├── master_data/       # JSONマスター
│   ├── tests/
│   ├── openapi.yaml       # API仕様書（FE/BE共有）
│   ├── requirements.txt
│   └── pyproject.toml
│
├── infra/                 # U3: AWS CDK (TypeScript)
│   ├── lib/               # CDKスタック
│   ├── bin/               # CDKエントリ
│   ├── cdk.json
│   ├── package.json
│   └── tsconfig.json
│
├── assets/                # U4: イラスト素材（原本）
│   ├── namekemono/
│   ├── items/
│   ├── forest/
│   ├── ui/
│   └── onboarding/
│
├── shared/                # 共有型定義・定数
│   └── types/             # TypeScript型定義（FE/BE共有用）
│
├── aidlc-docs/            # AI-DLC設計ドキュメント
├── docs/                  # その他ドキュメント
├── .github/
├── README.md
└── AGENTS.md
```

---

## コード構成戦略

### モノレポ採用
- 4ユニット全てを1リポジトリで管理
- Git履歴が一元化され、ユニット横断の変更（例: API仕様変更 + FE/BE対応）が追跡しやすい

### 並列開発を支える仕組み
1. **OpenAPI先行**: Day 1 に `backend/openapi.yaml` を作成。FE/BEが同じ仕様書を参照
2. **shared/types**: TypeScript型定義を共有（必要に応じて）
3. **プレースホルダー**: FEはAssets待ちをせず、プレースホルダー画像で開発
4. **環境変数**: インフラ情報（API URL、S3バケット名等）は環境変数で注入

### ブランチ戦略（提案）
- `main` — 常にデプロイ可能
- `feature/u1-*` / `feature/u2-*` / `feature/u3-*` / `feature/u4-*` — ユニット別ブランチ
- PRレビューは担当者優先、他ユニット関連部分のみクロスレビュー

### CI/CD（最小限）
- ハッカソン用途のため過度なCI/CDは不要
- 最低限: `main` へのマージ時に `cdk deploy` を手動実行
- リンター・ユニットテストはローカルで実行
