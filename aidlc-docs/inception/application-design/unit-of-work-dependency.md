# Unit of Work Dependency

## ユニット間依存関係マトリクス

| ユニット | 依存するユニット | 依存の内容 |
|---|---|---|
| U1 Frontend | U2 Backend | OpenAPI仕様書（API I/O定義） |
| U1 Frontend | U3 Infrastructure | CloudFront配信URL、API Gateway URL（環境変数経由） |
| U1 Frontend | U4 Assets | 画像素材（プレースホルダー開発可能。最終差し替え） |
| U2 Backend | U3 Infrastructure | DynamoDBテーブル名、S3バケット名、IAMロール（環境変数経由） |
| U2 Backend | U1 Frontend | OpenAPI仕様合意のみ（コード依存なし） |
| U3 Infrastructure | U2 Backend | Lambdaコードパッケージ（zip） |
| U3 Infrastructure | U1 Frontend | ビルド成果物（dist/）のS3デプロイ |
| U4 Assets | — | 依存なし（独立作業） |

---

## 依存関係図

```
             ┌──────────────┐
             │  U4 Assets   │ (独立)
             └──────┬───────┘
                    │ 画像素材
                    ▼
┌──────────────┐  API仕様  ┌──────────────┐
│ U1 Frontend  │ ◄───────► │  U2 Backend  │
└──────┬───────┘           └──────┬───────┘
       │                          │
       │ ビルド成果物              │ Lambdaコード
       ▼                          ▼
     ┌──────────────────────────────┐
     │      U3 Infrastructure       │
     │  (CDK / AWSリソース管理)     │
     └──────────────────────────────┘
              │     │     │
              ▼     ▼     ▼
          S3+CF  APIGW  DynamoDB...
```

---

## 並列開発のためのインターフェース

### OpenAPI仕様書（FE-BE間の契約）

**配置**: `backend/openapi.yaml`

**所有者**: U2 Backend が作成・管理
**利用者**: U1 Frontend が参照

**更新ルール**:
- API仕様変更時はBE担当がOpenAPIを更新
- FE担当に速やかに共有（Slack等）
- breaking changeは事前相談

### 環境変数（Infrastructure → FE/BE）

U3 Infrastructure がデプロイ時に出力する値:

| 変数 | 受け取り側 | 用途 |
|---|---|---|
| `VITE_API_BASE_URL` | U1 Frontend | API Gateway URL |
| `VITE_LIFF_ID` | U1 Frontend | LIFF ID |
| `DYNAMODB_TABLE_USERS` | U2 Backend | Usersテーブル名 |
| `DYNAMODB_TABLE_USER_NAMEKEMONO` | U2 Backend | UserNamekemonoテーブル名 |
| `DYNAMODB_TABLE_USER_PROGRESS` | U2 Backend | UserProgressテーブル名 |
| `DYNAMODB_TABLE_LETTERS` | U2 Backend | Lettersテーブル名 |
| `DYNAMODB_TABLE_OTUKAI_HISTORY` | U2 Backend | OtukaiHistoryテーブル名 |
| `S3_BUCKET_OTUKAI_IMAGES` | U2 Backend | おつかい画像保存バケット |
| `LINE_CHANNEL_ACCESS_TOKEN` | U2 Backend | Secrets Manager経由 |
| `LINE_CHANNEL_SECRET` | U2 Backend | Secrets Manager経由 |
| `BEDROCK_MODEL_ID` | U2 Backend | 使用するBedrockモデル |

### 画像アセット（Assets → FE）

**配置ルール**:
- U4 が作成 → `assets/` ディレクトリに配置
- 最終成果物を `frontend/src/assets/` にコピー
- 命名規則: `namekemono/{id}.png`, `items/{id}.png` 等

**プレースホルダー運用**:
- U1は `assets/placeholder/` を参照して開発
- U4から素材が届いたら差し替え

---

## クリティカルパス

```
Day 1: BE が OpenAPI仕様書を作成 → FE/BE並列開発開始
Day 1-2: Infra の DynamoDB + S3+CloudFront 立ち上げ → BE/FE が開発環境に接続
Day 3-5: FE/BE 並列開発（コア機能）
Day 6-8: Infra Bedrock連携、EventBridge → BE がおつかい機能実装
Day 9-11: Assets 差し替え、統合テスト
Day 12-14: 動画撮影、微調整
```

---

## リスクと緩和策

| リスク | 影響ユニット | 緩和策 |
|---|---|---|
| API仕様の食い違い | U1, U2 | OpenAPIを単一の真実の源（SSOT）とする。変更時は必ず仕様書を先に更新 |
| 画像素材の遅延 | U1, U4 | プレースホルダーで開発可能にし、最終日に差し替え |
| CDK学習コスト | U3 | 基本パターンのサンプルコードを早期に確立。AWS公式サンプル参照 |
| LIFF/LINE API未経験 | U1, U2 | Day 1-2で縦串スパイク（友だち追加→LIFF起動→API呼び出しの動作確認） |
| Bedrock統合の複雑さ | U2 | Day 6より前にBedrock単体の呼び出し実験を済ませる |
| メンバーB（1人）の作業負荷 | U2, U3 | Infraは最小限に絞る。可能な限り自動化（CDK）で手作業を減らす |
| メンバーC不在による素材不足 | U1, U4 | 最低限の5体 + 主要UIアイコンを優先的に作成 |
