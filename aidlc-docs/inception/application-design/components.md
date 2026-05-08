# Components

## フロントエンド（React + Vite + LIFF）

### 画面コンポーネント

| # | コンポーネント | 責務 |
|---|---|---|
| P-01 | ForestPage | メイン画面。森の表示、ナマケモノ配置、つぶやき表示、日次収穫、放置ボーナス受け取り、贈り物（InventoryDialog経由） |
| P-02 | SwipeReportPage | スワイプ報告画面。ダメカード表示、左右スワイプ操作、途中離脱、結果表示 |
| P-03 | ShopPage | ショップ画面。アイテム一覧表示、購入操作 |
| P-04 | EncyclopediaPage | 図鑑画面。ナマケモノ一覧（実体/シルエット）、ヒント表示、詳細表示 |
| P-05 | LetterCollectionPage | 手紙コレクション画面。受け取った手紙の一覧・閲覧 |
| P-06 | DiaryPage | 森の日記画面。おつかい記録の時系列表示 |
| P-07 | OnboardingDialog | 初回起動時の絵本オンボーディング（3ページ）。ForestPage上にオーバーレイ表示 |

### UIコンポーネント（再利用可能）

| # | コンポーネント | 責務 |
|---|---|---|
| U-01 | TabBar | 下部タブナビゲーション（森、スワイプ、ショップ、図鑑、その他） |
| U-02 | SwipeCard | スワイプ可能なダメカード。左右スワイプ操作のUI |
| U-03 | NamekemonoSprite | 森画面上のナマケモノ表示。タップでつぶやき表示 |
| U-04 | CoinDisplay | ナマケコイン残高表示 |
| U-05 | InventoryDialog | 所持品ダイアログ。アイテム一覧と贈る相手の選択 |
| U-06 | SummonAnimation | ナマケモノ出現演出 |
| U-07 | AffinityMeter | 親密度ゲージ（★1〜★5） |
| U-08 | LetterCard | 手紙1通の表示カード |
| U-09 | HarvestAnimation | 日次収穫演出（コイン獲得） |
| U-10 | NotificationBadge | 新着通知バッジ（新しいナマケモノ、手紙等） |

---

## バックエンド（AWS Lambda / Python）

### API Lambda（機能単位）

| # | Lambda | 責務 | トリガー |
|---|---|---|---|
| L-01 | user-api | ユーザー取得（初回起動判定・放置ボーナス自動付与含む）、オンボーディング完了 | API Gateway |
| L-02 | swipe-api | スワイプ報告受付・NMK計算・出現条件進行・pendingAppearances登録・途中離脱処理 | API Gateway |
| L-03 | forest-api | 森の状態取得（pendingAppearances実体化）・ナマケモノ一覧・つぶやき取得・日次収穫 | API Gateway |
| L-04 | shop-api | アイテム一覧取得・購入処理・所持品取得 | API Gateway |
| L-05 | affinity-api | アイテム贈呈・親密度更新・手紙解放判定・手紙取得 | API Gateway |
| L-06 | encyclopedia-api | 図鑑データ取得・ヒント解放状態 | API Gateway |
| L-07 | diary-api | 森の日記取得 | API Gateway |
| L-12 | dev-api | 開発用API（通知・おつかい手動トリガー、データリセット、強制出現） | API Gateway（認証は共有シークレット or IP制限） |

### スケジュール Lambda

| # | Lambda | 責務 | トリガー |
|---|---|---|---|
| L-08 | night-notification | 夜のスワイプ促し通知送信 | EventBridge（21:00 JST） |
| L-09 | morning-notification | 朝の新着ナマケモノ通知送信（pendingAppearances対象） | EventBridge（8:00 JST） |
| L-10 | otukai-scheduler | おつかい対象者判定・おつかい生成・配信 | EventBridge（10:00/13:00/16:00 JST） |

### Webhook Lambda

| # | Lambda | 責務 | トリガー |
|---|---|---|---|
| L-11 | line-webhook | LINE Webhook受信。follow/unfollow処理、おつかい報告（テキスト/画像）のルーティング、画像S3保存、Bedrock呼び出し、LINE Reply | API Gateway（LINE Webhook URL） |

### 共通モジュール（Lambda Layer or 共有コード）

| # | モジュール | 責務 |
|---|---|---|
| M-01 | db-client | DynamoDBアクセスの共通ラッパー |
| M-02 | line-client | LINE Messaging API呼び出し共通ラッパー（Push/Reply/fetchMessageContent） |
| M-03 | master-data | マスターデータ（JSON）の読み込み・キャッシュ |
| M-04 | game-config | ゲームバランスパラメータの読み込み（JSON） |
| M-05 | s3-client | S3アップロード/取得の共通ラッパー（おつかい報告画像用） |
| M-06 | auth | LIFF ID tokenの検証・LINE userId抽出。全API Lambda共通の前処理 |

---

## インフラ（AWS CDK / TypeScript）

| # | コンポーネント | 責務 |
|---|---|---|
| I-01 | CloudFront + S3 | フロントエンド配信（React SPA） |
| I-02 | API Gateway | REST API エンドポイント（Lambda統合） |
| I-03 | DynamoDB Tables | データ永続化（5テーブル） |
| I-04 | EventBridge Rules | スケジュール実行（通知・おつかい、計5ルール） |
| I-05 | S3 (Assets) | おつかい報告画像の保存 |
| I-06 | Lambda Functions | 全Lambda関数のデプロイ |
| I-07 | IAM Roles | Lambda実行ロール、DynamoDB/S3/Bedrockアクセス権限 |
| I-08 | CloudWatch Logs | Lambda実行ログ |
