# Unit of Work — ナマケモノの森（改訂版）

## 設計意図（Design Rationale）

本プロジェクトの Unit 構造は、**「論理 Unit（設計境界）」×「物理担当（実行マッピング）」** の二層構造を採用する。純粋な機能分解でもなく、担当レイヤー分解でもなく、その両方を両立させる設計判断である。

### 二層構造の定義

| 層 | 定義 | 目的 |
|---|---|---|
| **論理 Unit** | AI-DLC の per-unit ループの単位。機能ドメイン（Swipe, Forest, Shop, Otukai, LINE, Infra…）で分割 | 設計の密度・トレーサビリティ・テスト境界を担保 |
| **物理担当** | 現実のチーム構成に基づく担当マッピング。1 人が複数論理 Unit を担当することも、複数人が 1 論理 Unit を協働することもある | チームの技能配分と稼働制約に適合させる |

### チーム構成（3名体制・ハッカソンレギュレーション上は最大4名）

| メンバー | 役割 | 技能プロファイル | 稼働特性 |
|---|---|---|---|
| **メンバーA（今野さん）** | フロントエンド開発 | FE寄りスタック、AWS初心者 | 同期稼働 |
| **メンバーB（PM）** | フルスタック開発 + インフラ + AI連携 | FE/BE 両方、AWS精通 | 同期稼働 |
| **メンバーC** | 画像・イラスト制作 | 画像系に強い | 非同期稼働（海外出張中のため非同期稼働に適する。画像制作は独立性が高く、最悪 A/B で巻き取り可能） |

### 協働スタイル

**主スタイル: 担当レイヤー分離（厳密分担）**
- AWS が絡む論理 Unit（U0 Contracts, U5 Otukai/Bedrock, U6 LINE, U7 Infra）は全て B が主担当
- これにより A の AWS 学習コストを進捗クリティカルパスから外す
- 機能ドメイン Unit（U2〜U5）は FE側=A / BE側=B で分離。OpenAPI 契約（U0）を介して並列実装

**保険スタイル: ハイブリッド越境（必要時）**
- ドメイン整合性（NMK 計算、親密度、出現条件などのゲームバランス数字）は B が最終決定権を持つが、A もレビュー参加
- A が FE 側でハマった時は B が越境してレスキュー
- B は必要に応じて FE の API クライアント層スケルトンを先行作成

### 代替案の検討と却下理由

| 案 | 概要 | 却下/採用理由 |
|---|---|---|
| **案1: 純粋機能分解** | 8 Unit を 1 人 1 Unit で担当 | **却下**。A が AWS 初心者なため、1 Unit を FE/BE 縦串で担当できない。結果として B に全 BE 責任が集中し、純粋機能分解の意味が薄れる |
| **案2: 担当レイヤー分解のみ（初版）** | FE / BE / Infra / Assets の 4 Unit | **却下**。U2 Backend が Lambda 12本・6サービス・Bedrock・LINE・ゲームロジック全部入りで肥大化。per-unit ループの粒度として重すぎ、設計密度・トレーサビリティが不足 |
| **案3: 論理×物理の二層構造（採用）** | 9 論理 Unit × 柔軟な担当マッピング | **採用**。設計密度（per-unit ループを論理 Unit 単位で回せる）と、実行可能性（担当者の技能配分に適合）を両立 |

### per-unit ループの回し方

- **Functional Design**: 全 9 論理 Unit で独立実施（Q1=A の採択による）。AI-DLC の原則に忠実
- **Code Generation**: 論理 Unit 毎に独立実施
- **承認ゲート**: Functional Design / Code Generation の各終了時に計 18 回発生
- **トレードオフ**: 承認回数は多いが、設計の密度・再現性・レビュー容易性が最大化される

---

## 論理 Unit 構成（9 Unit）

### 概要マトリクス

| # | 論理 Unit | 主担当 | 優先度 | 区分 |
|---|---|---|---|---|
| **U0** | Contracts（OpenAPI / Shared Types / 認証契約） | B | 🔴 Tier 1 | アプリ設計 |
| **U1** | LIFF Frontend Shell | A | 🔴 Tier 1 | アプリ設計 |
| **U2** | Swipe / NMK Core | FE:A / BE:B | 🔴 Tier 2 | アプリ設計 |
| **U3** | Forest / Collection / Affinity | FE:A / BE:B | 🟠 Tier 3 | アプリ設計 |
| **U4** | Shop / Inventory | FE:A / BE:B | 🟠 Tier 3 | アプリ設計 |
| **U5** | Otukai / Bedrock | FE:A / BE:B | 🟡 Tier 4 | アプリ設計 |
| **U6** | LINE Notification / Webhook | B | 🟡 Tier 4 | アプリ設計 |
| **U7** | Infrastructure | B | 🔴 Tier 1（スケルトン先行） | アプリ設計 |
| **U8** | Assets | C | 独立並列 | 制作分担（*別セクション参照） |

### Tier 優先度の定義

- **🔴 Tier 1（最優先・ブロッカー）**: U0 Contracts + U1 LIFF Shell + U7 Infra スケルトン。これが動かないと他の論理 Unit が着手できない
- **🔴 Tier 2**: U2 Swipe / NMK Core。デモシナリオの中核で、全フェーズで使われる基盤
- **🟠 Tier 3**: U3 Forest / U4 Shop。Tier 2 の NMK 経済圏を前提とする
- **🟡 Tier 4**: U5 Otukai / U6 LINE Notification。MVP v1 に含まれるが、他機能が揃った後に統合できる

---

## U0: Contracts Unit

### 責務
- OpenAPI 3.x 仕様書の作成・維持（`backend/openapi.yaml`）
- Shared Types（TypeScript型定義、`shared/types/`）
- 認証契約（LIFF ID Token 検証仕様、フロント→バック認証ヘッダ形式）
- エラーコード規約（`{code, message}` 形式）
- API Gateway エンドポイントパス体系の決定

### 成果物
- `backend/openapi.yaml` — 全 API の I/O 仕様
- `shared/types/` — FE/BE 共有する TypeScript 型定義
  - `shared/types/api.ts` — API リクエスト/レスポンス型
  - `shared/types/models.ts` — ドメインモデル型（User, Namekemono, Item, Letter, Otukai）
  - `shared/types/errors.ts` — エラーコード定義
- `backend/modules/auth/` の仕様ドキュメント（実装は U7/各 Unit で）

### 含まれるコンポーネント（Component → Unit マトリクス由来）
- （設計契約のみで実コンポーネントは含まない。auth モジュール M-06 の仕様定義はここ、実装は各 API Lambda が呼び出す形）

### 主担当
**B**（OpenAPI 設計は BE 視点が主、ただし A もレビュー必須）

### Definition of Done（DoD）
- [ ] `backend/openapi.yaml` に全 Lambda の I/O が記述されている（L-01〜L-12）
- [ ] `shared/types/` に最低限のモデル型（User, Namekemono, Item, Letter, Otukai, Card）が定義されている
- [ ] エラーコード体系が文書化されている（認証エラー、バリデーションエラー、リソース不在、外部 API エラー）
- [ ] LIFF ID Token 検証フローが文書化されている（M-06 auth の振る舞い仕様）
- [ ] A のレビューを通過している（FE 側から使いづらい点がないことの確認）

### 依存
- **上流**: Application Design（components.md, component-methods.md）
- **下流**: U1〜U7 全て（契約を参照する）

### スケジュール目安
- Day 1 前半〜中盤: OpenAPI 仕様書初版、Shared Types 初版
- Day 1 後半: A とレビュー、調整
- 以降: Functional Design での変更を随時反映

---

## U1: LIFF Frontend Shell Unit

### 責務
- Vite + React + TypeScript プロジェクト初期化
- LIFF SDK 初期化・LIFF ID Token 取得
- ルーティング（React Router）
- 状態管理基盤（UserContext, ForestContext, InventoryContext）
- API クライアント層の骨格（axios 等、認証ヘッダ自動付与）
- エラーハンドリング共通層
- TabBar（U-01）実装
- 共通レイアウト・グローバルスタイル

### 成果物
- `frontend/` プロジェクト初期化（package.json, vite.config.ts, tsconfig.json）
- `frontend/src/main.tsx`, `App.tsx`（LIFF 初期化、ルーティング）
- `frontend/src/contexts/` — UserContext, ForestContext, InventoryContext
- `frontend/src/api/client.ts` — 認証付き API クライアント
- `frontend/src/api/hooks/` — useGetUser 等の最小フック
- `frontend/src/components/TabBar.tsx`（U-01）
- `frontend/src/styles/globals.css`

### 含まれるコンポーネント
- **フロントエンド UIコンポーネント**: U-01 TabBar
- **共通基盤**（components.md に明示的には出ないが本 Unit で提供）: 認証クライアント、API ラッパー、Context、ルーティング

### 主担当
**A**（FE 専任）

### Definition of Done（DoD）
- [ ] LIFF アプリとしてスマホで起動し、LIFF ID Token を取得できる
- [ ] U0 の API クライアント契約に沿った axios ラッパーが動作する（認証ヘッダ自動付与、エラー変換）
- [ ] 7 画面分のルーティングが設定されている（実画面の中身は空で可）
- [ ] 3 つの Context（User/Forest/Inventory）がプロバイダ階層として機能している
- [ ] TabBar（U-01）が表示され、画面遷移ができる
- [ ] `/dev` パスが公開されておらず、`/forest` にデフォルトリダイレクトする

### 依存
- **上流**: U0 Contracts（API 契約・認証契約）、U7 Infra（CloudFront + S3 配信、LIFF チャネル設定）
- **下流**: U2〜U5 の FE 側（本 Unit の基盤上に機能画面が積み上がる）

### スケジュール目安
- Day 1 後半〜Day 2: プロジェクト初期化、LIFF 初期化、ルーティング、Context 基盤
- Day 3: API クライアント、TabBar
- 以降: U2〜U5 の実装に併走しながら拡張

---

## U2: Swipe / NMK Core Unit

### 責務
- スワイプ報告画面の全機能（U-02 SwipeCard 含む）
- 通常スワイプ（10枚）と初回チュートリアルスワイプ（5枚・1,2枚目は強制「やった」）の分岐
- 途中離脱判定（×ボタン押下時の残り枚数 × 3NMK 付与）
- ゼロスワイプ離脱（1枚もスワイプせず閉じる = 20NMK）
- NMK 計算ロジック（スワイプ報告、途中離脱、ゼロスワイプ、放置ボーナス蓄積）
- 出現条件進行ロジック（カテゴリ別スワイプカウント、途中離脱回数）
- pendingAppearances 登録（翌朝 8:00 JST 出現予定）
- ダメカード・ナマケモノ出現条件マスターデータ（`backend/master_data/cards.json`, `namekemono.json`）

### 成果物
- **FE**:
  - `frontend/src/pages/SwipeReportPage.tsx`（P-02）
  - `frontend/src/components/SwipeCard.tsx`（U-02）
  - `frontend/src/components/CoinDisplay.tsx`（U-04）
- **BE**:
  - `backend/functions/swipe_api/` — L-02 Lambda 実装
  - `backend/services/game_core/swipe.py` — スワイプ報酬計算
  - `backend/services/game_core/appearance.py` — 出現条件判定
  - `backend/master_data/cards.json` — 全 30 種ダメカード
  - `backend/master_data/namekemono.json` — MVP 5 体 + 将来枠のナマケモノ定義
  - `backend/tests/test_swipe.py`, `backend/tests/test_appearance.py`

### 含まれるコンポーネント
- **P-02** SwipeReportPage（FE）
- **U-02** SwipeCard（FE）
- **U-04** CoinDisplay（FE・多くの画面で使用するが実装帰属は U2）
- **L-02** swipe-api（BE）
- **GameCoreService.calculateSwipeReward, checkAppearanceConditions**（BE サービス層）
- **master-data**（M-03）の内 cards.json / namekemono.json 部分

### 主担当
- **FE**: A
- **BE**: B

### Definition of Done（DoD）
- [ ] `/swipe` 画面で通常 10 枚スワイプが動作し、途中離脱・ゼロ離脱が判定される
- [ ] 初回チュートリアル 5 枚モード（強制ねむねむ→強制もぐもぐ）が動作する
- [ ] NMK 計算が `game-config.json` の値を参照している（ハードコード禁止）
- [ ] 出現条件マスタに基づいて pendingAppearances が UserProgress に登録される
- [ ] swipe-api のユニットテストがハッピーパスをカバー（報酬計算、条件進行）
- [ ] OpenAPI（U0）に準拠している
- [ ] FE/BE 統合テスト: 10 枚スワイプ→NMK 獲得→出現条件進行が E2E で動作する

### 依存
- **上流**: U0 Contracts（API 契約）、U1 LIFF Shell（Context、ルーティング）、U7 Infra（DynamoDB, Lambda デプロイ）
- **下流**: U3 Forest（pendingAppearances の実体化先）、U6 LINE Notification（朝通知の対象判定）

### スケジュール目安
- Day 3〜Day 5: FE スワイプ UI、BE swipe-api、マスターデータ、NMK 計算

---

## U3: Forest / Collection / Affinity Unit

### 責務
- 森画面（P-01 ForestPage）— ナマケモノ配置、つぶやき、日次収穫、新着演出
- 図鑑画面（P-04 EncyclopediaPage）— ナマケモノ一覧、ヒント、シルエット/実体
- 手紙コレクション画面（P-05 LetterCollectionPage）
- 日次収穫ロジック（オンデマンド計算、親密度ボーナス連動）
- pendingAppearances 実体化（forest-api 呼び出し時）
- 親密度計算・手紙解放判定
- 手紙マスターデータ（`backend/master_data/letters.json`）

### 成果物
- **FE**:
  - `frontend/src/pages/ForestPage.tsx`（P-01）
  - `frontend/src/pages/EncyclopediaPage.tsx`（P-04）
  - `frontend/src/pages/LetterCollectionPage.tsx`（P-05）
  - `frontend/src/components/NamekemonoSprite.tsx`（U-03）
  - `frontend/src/components/SummonAnimation.tsx`（U-06）
  - `frontend/src/components/AffinityMeter.tsx`（U-07）
  - `frontend/src/components/LetterCard.tsx`（U-08）
  - `frontend/src/components/HarvestAnimation.tsx`（U-09）
- **BE**:
  - `backend/functions/forest_api/`（L-03）
  - `backend/functions/encyclopedia_api/`（L-06）
  - `backend/functions/affinity_api/`（L-05）の親密度・手紙部分
  - `backend/services/game_core/harvest.py`
  - `backend/services/game_core/pending.py`（resolvePendingAppearances）
  - `backend/services/affinity/`（一部、U4 と共有）
  - `backend/master_data/letters.json`
  - `backend/tests/test_harvest.py`, `backend/tests/test_pending.py`, `backend/tests/test_affinity.py`

### 含まれるコンポーネント
- **P-01** ForestPage、**P-04** EncyclopediaPage、**P-05** LetterCollectionPage（FE）
- **U-03** NamekemonoSprite、**U-06** SummonAnimation、**U-07** AffinityMeter、**U-08** LetterCard、**U-09** HarvestAnimation（FE）
- **L-03** forest-api、**L-06** encyclopedia-api（BE）
- **L-05** affinity-api の「親密度」「手紙」部分（「贈り物」は U4）
- **GameCoreService.resolvePendingAppearances, harvest**（BE）
- **AffinityService（一部、手紙解放判定）**
- **master-data** letters.json 部分

### 主担当
- **FE**: A
- **BE**: B

### Definition of Done（DoD）
- [ ] 森画面でナマケモノが画像表示（プレースホルダー可）、タップでつぶやき表示
- [ ] forest-api 呼び出し時に pendingAppearances が実体化される
- [ ] 日次収穫がオンデマンド計算で動作する（EventBridge ではない）
- [ ] 親密度ボーナスが日次収穫に反映される
- [ ] 図鑑画面で実体/シルエット、ヒント表示が動作する
- [ ] 親密度★3 到達で手紙がランダム配信され、手紙コレクションに保存される
- [ ] ユニットテスト: 日次収穫計算、pendingAppearances 実体化、親密度変動、手紙解放判定
- [ ] FE/BE 統合テスト: スワイプ→翌朝 forest-api→新ナマケモノ出現のフローが動作

### 依存
- **上流**: U0 Contracts、U1 LIFF Shell、U2 Swipe/NMK Core（pendingAppearances を登録する側）、U7 Infra
- **下流**: U4 Shop（親密度ゲージは Shop からの贈呈で上昇）、U5 Otukai（親密度 MAX 判定）、U6 LINE Notification（朝通知の対象判定）

### スケジュール目安
- Day 5〜Day 8: 森画面、図鑑、日次収穫、親密度、手紙

---

## U4: Shop / Inventory Unit

### 責務
- ショップ画面（P-03 ShopPage）
- 所持品ダイアログ（U-05 InventoryDialog）
- アイテム購入ロジック（NMK 消費、所持品追加）
- 贈り物ロジック（所持品消費、親密度上昇の起点）
- ショップマスターデータ（`backend/master_data/items.json`）

### 成果物
- **FE**:
  - `frontend/src/pages/ShopPage.tsx`（P-03）
  - `frontend/src/components/InventoryDialog.tsx`（U-05）
- **BE**:
  - `backend/functions/shop_api/`（L-04）
  - `backend/functions/affinity_api/`（L-05）の「贈り物」部分
  - `backend/services/shop/`（ShopService: purchase, consumeItem, getInventory）
  - `backend/services/affinity/`（giveItem, 親密度上昇量計算。U3 と共有だが、起点は U4）
  - `backend/master_data/items.json`
  - `backend/tests/test_shop.py`, `backend/tests/test_gift.py`

### 含まれるコンポーネント
- **P-03** ShopPage（FE）
- **U-05** InventoryDialog（FE）
- **L-04** shop-api（BE）
- **L-05** affinity-api の「giveItem」部分
- **ShopService 全体**
- **AffinityService（親密度上昇計算）**
- **master-data** items.json 部分

### 主担当
- **FE**: A
- **BE**: B

### Definition of Done（DoD）
- [ ] ショップ画面で 3 カテゴリ（おやつ/おもちゃ/特別な贈り物）のアイテムが表示される
- [ ] NMK 残高が足りない場合は購入ボタンが無効化される
- [ ] 購入後、所持品に追加され NMK 残高が減る
- [ ] 所持品ダイアログでアイテムと贈呈対象ナマケモノを選び、親密度が上昇する
- [ ] 親密度の上昇量が `game-config.json` の値で制御される
- [ ] ユニットテスト: 購入、消費、親密度上昇（アイテムカテゴリ別）

### 依存
- **上流**: U0 Contracts、U1 LIFF Shell、U2（NMK が存在する前提）、U3（親密度の持ち主）、U7 Infra
- **下流**: U3 の手紙解放判定（親密度★3 到達トリガー）、U5 Otukai（親密度 MAX 到達トリガー）

### スケジュール目安
- Day 5〜Day 7: ショップ、所持品、贈り物、親密度上昇

---

## U5: Otukai / Bedrock Unit

### 責務
- 森の日記画面（P-06 DiaryPage）
- おつかい対象者判定（親密度 MAX + 前回おつかいからの経過日数 + currentOtukaiId が空）
- おつかい生成（Bedrock 呼び出し、キャラクター口調、天気・祝日ガードレール）
- おつかい応答（LINE Reply、Bedrock マルチモーダル対応）
- 画像処理（line-client.fetchMessageContent → s3-client.putObject）
- 達成判定（Bedrock）
- 森の日記生成（Bedrock 要約）
- プロンプトインジェクション対策（NFR-007 対応：入力サニタイズ、ロール固定、出力フィルタリング、会話長制限、Bedrock Guardrails）
- 外部 API 連携（天気 API、祝日 API）
- `backend/modules/s3_client.py`（M-05）

### 成果物
- **FE**:
  - `frontend/src/pages/DiaryPage.tsx`（P-06）
- **BE**:
  - `backend/functions/otukai_scheduler/`（L-10）
  - `backend/functions/line_webhook/`（L-11）のおつかい会話部分
  - `backend/functions/diary_api/`（L-07）
  - `backend/services/otukai/`（OtukaiService: getEligibleUsers, generateOtukai, processReply, judgeCompletion, recordDiary）
  - `backend/modules/s3_client.py`（M-05）
  - `backend/modules/line_client/fetch_message_content.py`（M-02 の拡張、L-11 と共有）
  - `backend/services/otukai/guardrails.py`（天気・祝日連携、プロンプトインジェクション対策）
  - `backend/master_data/otukai_prompts.json`（キャラ別プロンプトテンプレート）
  - `backend/tests/test_otukai_generation.py`, `backend/tests/test_otukai_reply.py`, `backend/tests/test_guardrails.py`

### 含まれるコンポーネント
- **P-06** DiaryPage（FE）
- **L-07** diary-api、**L-10** otukai-scheduler、**L-11** line-webhook の「おつかい会話・画像」部分（BE）
- **M-05** s3-client（BE 共通モジュール）
- **OtukaiService 全体**
- **master-data** otukai_prompts.json 部分
- **外部依存**: Bedrock、天気 API、祝日 API、S3（画像保存）

### 主担当
**B**（Bedrock・LINE API・外部 API が集中するため）。FE DiaryPage のみ A も軽く担当

### Definition of Done（DoD）
- [ ] dev-api 経由で手動トリガーした場合、親密度 MAX ユーザーにおつかいが生成される
- [ ] Bedrock が指定キャラの口調でおつかいメッセージを返す
- [ ] 天気 API・祝日 API のレスポンスがガードレールとして反映される（雨の日に「散歩して」が出ない等）
- [ ] LINE Webhook 経由でテキスト返信・画像返信が受信でき、Bedrock がキャラクタートーンで応答する
- [ ] 画像は S3 に保存され、OtukaiHistory に URI 配列が記録される
- [ ] 達成判定ロジックが動作し、達成時のみ森の日記が生成・保存される
- [ ] プロンプトインジェクション対策 5 種が全て実装されている（NFR-007 対応）
- [ ] 森の日記画面で達成済みおつかいの時系列表示が動作する
- [ ] ユニットテスト: おつかい生成（モック Bedrock）、達成判定、ガードレール、画像処理フロー

### 依存
- **上流**: U0 Contracts、U1 LIFF Shell、U3 Forest（親密度 MAX 判定元）、U6 LINE（Webhook ルーティング共有、Messaging API 共通モジュール）、U7 Infra（Bedrock/S3/EventBridge）
- **下流**: （なし。最終層）

### スケジュール目安
- Day 8〜Day 11: おつかいスケジューラ、Webhook おつかい処理、画像処理、森の日記、ガードレール

---

## U6: LINE Notification / Webhook Unit

### 責務
- LINE Messaging API 共通ラッパー（`backend/modules/line_client/`、M-02）
- 署名検証（Webhook 受信時の LINE 署名ヘッダ検証）
- Follow/Unfollow 処理（getOrCreateUser + notificationEnabled 切替）
- 夜のスワイプ通知（21:00 JST、Push、キャラランダム選択、固有セリフ）
- 朝の到着通知（8:00 JST、Push、pendingAppearances 存在者のみ）
- Webhook ルーティング（Follow/Unfollow → U6、おつかい会話 → U5 に dispatch）

### 成果物
- **BE**:
  - `backend/functions/night_notification/`（L-08）
  - `backend/functions/morning_notification/`（L-09）
  - `backend/functions/line_webhook/`（L-11）の「Follow/Unfollow/ルーティング」部分
  - `backend/modules/line_client/`（M-02 全般: Push, Reply, 署名検証）
  - `backend/services/notification/`（NotificationService）
  - `backend/services/user/`（UserService: getOrCreateUser, setNotificationEnabled）
  - `backend/tests/test_night_notification.py`, `backend/tests/test_morning_notification.py`, `backend/tests/test_webhook_signature.py`

### 含まれるコンポーネント
- **L-08** night-notification、**L-09** morning-notification（BE）
- **L-11** line-webhook の「Follow/Unfollow/ルーティング/署名検証」部分（BE）
- **M-02** line-client（BE 共通モジュール）
- **NotificationService 全体**
- **UserService**（getOrCreateUser, setNotificationEnabled）

### 主担当
**B**（LINE API 絡みは全て B）

### Definition of Done（DoD）
- [ ] LINE 友だち追加で Users テーブルにレコードが作成される
- [ ] ブロック（Unfollow）で notificationEnabled=false に更新される
- [ ] 夜通知が `21:00 JST` にキャラランダム選択で Push 送信される
- [ ] 朝通知が `8:00 JST` に pendingAppearances を持つユーザーのみに Push 送信される
- [ ] Webhook 署名検証が動作し、不正リクエストが 401 で拒否される
- [ ] Webhook が Follow/Unfollow/Message を正しく dispatch する
- [ ] dev-api 経由の手動トリガーで通知を即時送信できる
- [ ] ユニットテスト: 署名検証、通知対象者取得、キャラランダム選択

### 依存
- **上流**: U0 Contracts（認証は不要だが Webhook 仕様の契約は U0 管理）、U7 Infra（EventBridge, Secrets Manager, Lambda デプロイ、LINE Webhook URL）
- **下流**: U5 Otukai（Webhook 経由でおつかい会話を dispatch する）

### スケジュール目安
- Day 4〜Day 5: line-client、Webhook 骨格、Follow/Unfollow
- Day 8〜Day 10: 夜通知、朝通知、EventBridge 連携

---

## U7: Infrastructure Unit

### 責務
- AWS CDK（TypeScript）によるインフラ定義
- S3 バケット + CloudFront ディストリビューション（SPA 配信）
- API Gateway REST API（Lambda 統合）
- DynamoDB テーブル 5 つ（Users, UserNamekemono, UserProgress, Letters, OtukaiHistory）
- EventBridge ルール 5 本（夜通知、朝通知、おつかい 3 時刻）
- S3 バケット（おつかい報告画像用）
- Lambda 関数のデプロイパイプライン
- IAM ロール（Lambda 実行、Bedrock/S3/DynamoDB アクセス）
- CloudWatch Logs
- Secrets Manager（LINE Channel Access Token、LINE Channel Secret、Bedrock 関連）
- LIFF チャネル設定・連携確認

### 成果物
- `infra/` プロジェクト初期化
- `infra/bin/` — CDK アプリエントリポイント
- `infra/lib/` — スタック定義
  - `frontend-stack.ts` — S3 + CloudFront
  - `api-stack.ts` — API Gateway + Lambda
  - `database-stack.ts` — DynamoDB
  - `schedule-stack.ts` — EventBridge
  - `storage-stack.ts` — S3（画像）
  - `secrets-stack.ts` — Secrets Manager
- `cdk.json`, `package.json`, `tsconfig.json`
- `infra/README.md` — デプロイ手順書

### 含まれるコンポーネント
- **I-01** CloudFront + S3、**I-02** API Gateway、**I-03** DynamoDB Tables、**I-04** EventBridge Rules、**I-05** S3 (Assets)、**I-06** Lambda Functions、**I-07** IAM Roles、**I-08** CloudWatch Logs（全てインフラ）

### 主担当
**B**

### Definition of Done（DoD）
- [ ] `cdk deploy` で全スタックが AWS アカウントにデプロイされる
- [ ] CloudFront URL にアクセスすると React ビルド成果物が返る
- [ ] API Gateway に全 Lambda が統合されている
- [ ] DynamoDB 5 テーブルが PK/SK 設定済みで作成される
- [ ] EventBridge 5 ルールが JST 基準の時刻で動作する（UTC 変換済み）
- [ ] Secrets Manager に必要なシークレットが登録されている
- [ ] Lambda が Bedrock / DynamoDB / S3 にアクセスできる IAM 権限を持つ
- [ ] LIFF チャネルが作成され、Endpoint URL に CloudFront URL が設定されている
- [ ] **スケルトン先行**: Day 1-2 時点で DynamoDB と CloudFront + S3 は稼働し、他 Unit が使える状態

### 依存
- **上流**: （なし。最上流）
- **下流**: U1〜U6 全て（インフラの上で動く）

### スケジュール目安
- **Day 1-2 スケルトンフェーズ**: DynamoDB, S3+CloudFront, 最低限の IAM。他 Unit がこれを使い始められる状態に
- **Day 3-5 API フェーズ**: API Gateway + Lambda 統合。Lambda 実装の進捗に合わせて追加
- **Day 6-10 拡張フェーズ**: EventBridge, Secrets Manager, Bedrock IAM, S3（画像）
- **Day 11-14 仕上げ**: デプロイスクリプト整備、動画撮影用環境の安定化

---

## U8: Assets Unit（制作分担 Unit）

> **📌 性質の明示**: U8 は他の 8 Unit（U0〜U7）とは性質が異なる。U0〜U7 が **論理設計境界を反映したコード実装 Unit** であるのに対し、U8 は **制作物（画像素材）の分担を反映した非コード Unit** である。AI-DLC の per-unit ループ（Functional Design → Code Generation）の対象ではなく、制作進行管理の単位として定義する。
>
> それでも Unit として切り出す理由は以下:
> 1. 画像素材は設計・実装と独立に並列生産できる（依存切断）
> 2. 海外出張中のメンバー C に非同期稼働で割り当てる合理的な枠である
> 3. 成果物の受け渡し先（`frontend/src/assets/`）と納品タイミング（Tier 1 ナマケモノから順）を設計として明記する意味がある

### 責務
- ナマケモノ立ち絵（MVP 5 体：ネムリン、モグモグ、アトデーノ、スクロン、メンドクサウルス）
  - 実体 + シルエット
  - 各キャラ待機モーション or 表情差分（最小限）
- アイテムアイコン（おやつ/おもちゃ/特別な贈り物、計 10〜15 枚）
- 森の背景（4 段階: 小さな森／にぎやかな森／豊かな森／伝説の森）
- UI 素材（タブアイコン、ボタン、コイン、通知バッジ、スワイプの× or ✓）
- オンボーディング絵本イラスト（3 ページ）
- トンマナ維持（ゆる優しい、押し付けない、頑張れと言わない世界観）

### 成果物
- `assets/` ディレクトリ以下（原本）
  - `assets/namekemono/` — キャラ画像
  - `assets/items/` — アイテムアイコン
  - `assets/forest/` — 森背景
  - `assets/ui/` — UI 素材
  - `assets/onboarding/` — 絵本イラスト
- 最終成果物は `frontend/src/assets/` に配置（U1/U2/U3 のプレースホルダーを差し替え）

### 主担当
**C**（海外出張中で非同期稼働に適する）

### Definition of Done（DoD）
- [ ] MVP 5 体のナマケモノ立ち絵（実体 + シルエット）が `frontend/src/assets/namekemono/` に格納されている
- [ ] アイテムアイコン 10 枚以上が `frontend/src/assets/items/` に格納されている
- [ ] 森背景 4 段階が `frontend/src/assets/forest/` に格納されている
- [ ] UI 素材一式が `frontend/src/assets/ui/` に格納されている
- [ ] オンボーディング絵本 3 ページ分が `frontend/src/assets/onboarding/` に格納されている
- [ ] 全素材でトンマナが統一されている（簡易レビューを A または B が実施）

### 依存
- **上流**: Application Design（ナマケモノ定義、森段階定義）
- **下流**: U1〜U5 の FE（プレースホルダー差し替え先）
- **並列性**: 他 Unit の進捗と完全独立。プレースホルダーで開発が進むため、最終日近くの差し替えでも成立

### スケジュール目安（非同期・優先順）
- Day 1-3: ネムリン、モグモグ（Day 1-2 デモシナリオで最優先）
- Day 4-7: アトデーノ、スクロン、メンドクサウルス、アイテムアイコン
- Day 8-10: 森背景、UI 素材
- Day 11-12: オンボーディング絵本
- Day 13-14: 微修正・調整

**リスク緩和（Fallback 大枠）**: C が完全に稼働停止した場合、A/B が画像生成 AI 等で最低限の素材を代替作成。トンマナは落ちるが、デモ動画撮影は成立する。

---

## リポジトリ構成（モノレポ）

```
namekemono-no-mori/
├── frontend/              # U1 LIFF Shell + U2〜U5 の FE 実装
│   ├── src/
│   │   ├── pages/         # P-01〜P-07
│   │   ├── components/    # U-01〜U-10
│   │   ├── contexts/      # UserContext, ForestContext, InventoryContext
│   │   ├── api/           # 認証クライアント、API フック
│   │   ├── hooks/         # カスタムフック
│   │   ├── styles/        # globals.css, animations
│   │   └── assets/        # U8 から配置
│   ├── public/
│   ├── package.json
│   ├── vite.config.ts
│   └── tsconfig.json
│
├── backend/               # U2〜U6 の BE 実装
│   ├── functions/         # Lambda 関数（L-01〜L-12）
│   ├── services/          # サービス層
│   │   ├── game_core/     # U2, U3
│   │   ├── shop/          # U4
│   │   ├── affinity/      # U3, U4 共有
│   │   ├── notification/  # U6
│   │   ├── otukai/        # U5
│   │   └── user/          # U6
│   ├── modules/           # 共通モジュール（M-01〜M-06）
│   ├── master_data/       # JSON マスター
│   ├── tests/
│   ├── openapi.yaml       # U0 Contracts
│   ├── requirements.txt
│   └── pyproject.toml
│
├── infra/                 # U7 Infrastructure（CDK）
│   ├── lib/               # CDK スタック
│   ├── bin/               # CDK エントリ
│   ├── cdk.json
│   ├── package.json
│   └── tsconfig.json
│
├── assets/                # U8 原本（最終的に frontend/src/assets/ へコピー）
│   ├── namekemono/
│   ├── items/
│   ├── forest/
│   ├── ui/
│   └── onboarding/
│
├── shared/                # U0 Contracts の共有型定義
│   └── types/             # TypeScript 型定義（FE/BE 共有用）
│       ├── api.ts
│       ├── models.ts
│       └── errors.ts
│
├── aidlc-docs/            # AI-DLC 設計ドキュメント
├── docs/                  # その他ドキュメント
├── .github/
├── README.md
└── AGENTS.md
```

---

## コード構成戦略

### モノレポ採用理由
- 9 論理 Unit 全てを 1 リポジトリで管理
- Git 履歴が一元化され、Unit 横断の変更（例: API 仕様変更 + FE/BE 対応）が追跡しやすい
- 3 名体制でリポジトリを分散させる運用コストを回避

### 並列開発を支える仕組み
1. **U0 Contracts 先行**: Day 1 に OpenAPI 仕様書と shared/types を作成。U1〜U6 はこれを参照して並列実装
2. **プレースホルダー運用**: U1〜U5 FE は U8 Assets を待たず、プレースホルダー画像で実装を進める
3. **環境変数経由のインフラ情報注入**: API Gateway URL、S3 バケット名、LIFF ID 等を環境変数で注入
4. **スケルトン先行（U7）**: Day 1-2 で DynamoDB + CloudFront + S3 を先行デプロイし、他 Unit が「触れる」基盤を早期提供

### ブランチ戦略（提案）
- `main` — 常にデプロイ可能
- `feature/u{番号}-{概要}` 形式（例: `feature/u2-swipe-core`, `feature/u5-otukai-bedrock`）
- PR レビューは主担当優先、越境部分のみクロスレビュー

### CI/CD（最小限）
- ハッカソン用途のため過度な CI/CD は不要
- 最低限: `main` へのマージ時に `cdk deploy` を手動実行
- リンター・ユニットテストはローカルで実行

---

## 担当マッピング早見表

| 論理 Unit | FE | BE | Infra | Assets |
|---|:---:|:---:|:---:|:---:|
| U0 Contracts | B(レビュー) | **B** | — | — |
| U1 LIFF Shell | **A** | — | — | — |
| U2 Swipe/NMK Core | **A** | **B** | — | — |
| U3 Forest/Collection/Affinity | **A** | **B** | — | — |
| U4 Shop/Inventory | **A** | **B** | — | — |
| U5 Otukai/Bedrock | A | **B** | — | — |
| U6 LINE Notification/Webhook | — | **B** | — | — |
| U7 Infrastructure | — | — | **B** | — |
| U8 Assets | — | — | — | **C** |

**太字**は主担当。AWS 絡みは全て B に集中している点が、本チームの技能配分を反映した設計判断である。
