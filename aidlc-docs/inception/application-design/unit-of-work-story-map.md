# Unit of Work Story Map — ナマケモノの森（改訂版）

## 概要

本ドキュメントは、User Stories / Application Design の各成果物と論理 9 Unit の対応関係を 3 軸マトリクスで整理する。

### 提供するマトリクス

1. **ストーリー × 論理 Unit × FE/BE/Infra/Assets 軸** — 全 US-01〜US-17 と FS-01〜FS-03 の担当 Unit 特定
2. **Component → Unit マトリクス** — components.md の全要素（7 画面 / 10 UI / 11 Lambda / 6 共通モジュール / 8 インフラ）を Unit にマッピング
3. **デモシナリオ × 論理 Unit** — 1.5 日体験のシーンごとに必要な Unit を特定

---

## 1. ストーリー × 論理 Unit マトリクス

### 凡例
- ◎ 主責務（ストーリーの中核ロジック）
- ○ 副責務（必須だが中核ではない）
- △ 補助（UI 表示、インフラ基盤など）

### Epic 1: オンボーディング・初回体験

| Story | U0 Ctr | U1 Shell | U2 Swipe | U3 Forest | U4 Shop | U5 Otukai | U6 LINE | U7 Infra | U8 Assets |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| **US-01** 絵本オンボーディング | ○ | ◎ | — | △ | — | — | — | △ | ◎ |
| **US-02** 初回スワイプ体験 | ○ | ○ | ◎ | — | — | — | — | △ | ○ |
| **US-03** 初回ナマケモノ召喚 | ○ | ○ | ◎ | ◎ | — | — | — | △ | ◎ |

**Note**: US-01/US-02/US-03 は ForestPage 上にオーバーレイされる OnboardingDialog（P-07）で実現。P-07 の帰属は **U1 LIFF Shell**（アプリ初期化フロー全体の一部として）。

### Epic 2: スワイプ報告

| Story | U0 Ctr | U1 Shell | U2 Swipe | U3 Forest | U4 Shop | U5 Otukai | U6 LINE | U7 Infra | U8 Assets |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| **US-04** 通常スワイプ報告 | ○ | △ | ◎ | — | — | — | — | △ | ○ |
| **US-05** 途中離脱ボーナス | ○ | △ | ◎ | — | — | — | — | △ | ○ |

### Epic 3: ナマケコイン経済圏

| Story | U0 Ctr | U1 Shell | U2 Swipe | U3 Forest | U4 Shop | U5 Otukai | U6 LINE | U7 Infra | U8 Assets |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| **US-06** ショップでアイテム購入 | ○ | △ | — | — | ◎ | — | — | △ | ○ |
| **US-07** 日次コイン収穫 | ○ | △ | — | ◎ | — | — | — | △ | ○ |
| **US-08** 放置ボーナス蓄積 | ○ | △ | ○ | — | — | — | — | △ | — |

**Note**: US-08 放置ボーナスは user-api（L-01）で自動付与。NMK 計算ロジックの一部なので **U2 副責務**。

### Epic 4: ナマケモノ・コレクション

| Story | U0 Ctr | U1 Shell | U2 Swipe | U3 Forest | U4 Shop | U5 Otukai | U6 LINE | U7 Infra | U8 Assets |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| **US-09** ナマケモノ出現・図鑑登録 | ○ | △ | ○ | ◎ | — | — | — | △ | ◎ |
| **US-10** ナマケモノのつぶやき | ○ | △ | — | ◎ | — | — | — | △ | ○ |

### Epic 5: 親密度・手紙

| Story | U0 Ctr | U1 Shell | U2 Swipe | U3 Forest | U4 Shop | U5 Otukai | U6 LINE | U7 Infra | U8 Assets |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| **US-11** アイテム贈呈・親密度UP | ○ | △ | — | ○ | ◎ | — | — | △ | ○ |
| **US-12** 手紙を受け取る | ○ | △ | — | ◎ | — | — | — | △ | ○ |

### Epic 6: 通知

| Story | U0 Ctr | U1 Shell | U2 Swipe | U3 Forest | U4 Shop | U5 Otukai | U6 LINE | U7 Infra | U8 Assets |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| **US-13** 夜のスワイプ通知 | ○ | — | — | ○ | — | — | ◎ | ◎ | — |
| **US-14** 朝の到着通知 | ○ | — | ○ | ○ | — | — | ◎ | ◎ | — |

**Note**: U7 Infra が ◎ なのは、EventBridge スケジュールが通知の発火源であるため。

### Epic 7: おつかい

| Story | U0 Ctr | U1 Shell | U2 Swipe | U3 Forest | U4 Shop | U5 Otukai | U6 LINE | U7 Infra | U8 Assets |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| **US-15** おつかい受信 | ○ | — | — | ○ | — | ◎ | ○ | ◎ | — |
| **US-16** おつかい報告・AI応答 | ○ | — | — | — | — | ◎ | ○ | ◎ | — |
| **US-17** 森の日記 | ○ | △ | — | — | — | ◎ | — | △ | △ |

### MVP v2（将来ストーリー）

| Story | U0 Ctr | U1 Shell | U2 Swipe | U3 Forest | U4 Shop | U5 Otukai | U6 LINE | U7 Infra | U8 Assets |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| **FS-01** 好き/嫌いによる親密度変動 | — | — | — | ○ | ◎ | — | — | — | — |
| **FS-02** ナマケモノ同士のイベント | — | — | — | ◎ | — | — | — | — | ○ |
| **FS-03** 個人設定画面 | — | ◎ | — | — | — | — | ○ | — | △ |

**Note**: MVP v1 ではデータ構造のみ対応。UI 実装は MVP v2。

---

## 2. Component → Unit マトリクス（精密）

components.md の全要素を論理 Unit に一対一でマッピングする。これにより、実装時の「どの Unit が何を作るのか」が曖昧にならない。

### フロントエンド画面コンポーネント（7 画面）

| Component | 名称 | 担当 Unit | 物理担当 |
|---|---|---|---|
| **P-01** | ForestPage | U3 Forest/Collection/Affinity | A |
| **P-02** | SwipeReportPage | U2 Swipe/NMK Core | A |
| **P-03** | ShopPage | U4 Shop/Inventory | A |
| **P-04** | EncyclopediaPage | U3 Forest/Collection/Affinity | A |
| **P-05** | LetterCollectionPage | U3 Forest/Collection/Affinity | A |
| **P-06** | DiaryPage | U5 Otukai/Bedrock | A |
| **P-07** | OnboardingDialog | U1 LIFF Frontend Shell | A |

### フロントエンド UI コンポーネント（10 コンポーネント）

| Component | 名称 | 担当 Unit | 物理担当 | 備考 |
|---|---|---|---|---|
| **U-01** | TabBar | U1 LIFF Frontend Shell | A | グローバルナビ |
| **U-02** | SwipeCard | U2 Swipe/NMK Core | A | |
| **U-03** | NamekemonoSprite | U3 Forest/Collection/Affinity | A | |
| **U-04** | CoinDisplay | U2 Swipe/NMK Core | A | 多画面で使用するが NMK の概念帰属は U2 |
| **U-05** | InventoryDialog | U4 Shop/Inventory | A | |
| **U-06** | SummonAnimation | U3 Forest/Collection/Affinity | A | |
| **U-07** | AffinityMeter | U3 Forest/Collection/Affinity | A | |
| **U-08** | LetterCard | U3 Forest/Collection/Affinity | A | |
| **U-09** | HarvestAnimation | U3 Forest/Collection/Affinity | A | |
| **U-10** | NotificationBadge | U1 LIFF Frontend Shell | A | グローバル UI |

### バックエンド Lambda（11 関数 + dev-api）

| Component | 名称 | 担当 Unit | 物理担当 |
|---|---|---|---|
| **L-01** | user-api | U6 LINE Notification/Webhook（UserService 経由）と U2 Swipe/NMK Core（放置ボーナス計算）に跨る。主責務は **U6**（UserService の帰属が U6 のため）。放置ボーナスロジックは **U2 の副責務** | B |
| **L-02** | swipe-api | U2 Swipe/NMK Core | B |
| **L-03** | forest-api | U3 Forest/Collection/Affinity | B |
| **L-04** | shop-api | U4 Shop/Inventory | B |
| **L-05** | affinity-api | U3 と U4 に跨る。「贈り物（giveItem）」部分 = **U4**、「親密度更新・手紙解放」部分 = **U3** | B |
| **L-06** | encyclopedia-api | U3 Forest/Collection/Affinity | B |
| **L-07** | diary-api | U5 Otukai/Bedrock | B |
| **L-08** | night-notification | U6 LINE Notification/Webhook | B |
| **L-09** | morning-notification | U6 LINE Notification/Webhook | B |
| **L-10** | otukai-scheduler | U5 Otukai/Bedrock | B |
| **L-11** | line-webhook | U6（Follow/Unfollow、署名検証、dispatch）と U5（おつかい会話処理）に跨る。**Webhook 骨格は U6、おつかい処理は U5** | B |
| **L-12** | dev-api | 横断ユーティリティ。実装の主担当は B、運用上は **U7 Infra に帰属させる**（開発インフラの一部として） | B |

> **注記**: L-01 / L-05 / L-11 は複数 Unit に跨る。per-unit ループでは、跨る Lambda について「主責務の Unit が全体設計」し、「副責務の Unit は該当部分のみ設計」する運用とする。

### バックエンド共通モジュール（6 モジュール）

| Component | 名称 | 担当 Unit | 物理担当 |
|---|---|---|---|
| **M-01** | db-client | U7 Infrastructure（共通基盤） | B |
| **M-02** | line-client | U6 LINE Notification/Webhook | B |
| **M-03** | master-data | 読み込み機構は U7、マスターデータ本体は各 Unit が担当（cards.json/namekemono.json=U2、letters.json=U3、items.json=U4、otukai_prompts.json=U5） | B |
| **M-04** | game-config | U7 Infrastructure（設定読み込み機構として）、game-config.json の内容管理は U2/U3/U4 共有 | B |
| **M-05** | s3-client | U5 Otukai/Bedrock（画像処理のために新設） | B |
| **M-06** | auth | U0 Contracts（仕様）、実装は U7 Infrastructure（共通基盤として） | B |

### インフラ（8 コンポーネント）

| Component | 名称 | 担当 Unit | 物理担当 |
|---|---|---|---|
| **I-01** | CloudFront + S3（SPA 配信） | U7 Infrastructure | B |
| **I-02** | API Gateway | U7 Infrastructure | B |
| **I-03** | DynamoDB Tables（5 テーブル） | U7 Infrastructure | B |
| **I-04** | EventBridge Rules（5 ルール） | U7 Infrastructure | B |
| **I-05** | S3（画像保存） | U7 Infrastructure | B |
| **I-06** | Lambda Functions（デプロイ） | U7 Infrastructure | B |
| **I-07** | IAM Roles | U7 Infrastructure | B |
| **I-08** | CloudWatch Logs | U7 Infrastructure | B |

---

## 3. DynamoDB テーブル × 論理 Unit

| テーブル | 論理 Unit（主な更新元） | 備考 |
|---|---|---|
| **T-01** Users | U6（getOrCreateUser, Follow/Unfollow）+ U2（NMK 残高更新）+ U5（currentOtukaiId）+ U7（テーブル定義） | マルチドメインの中心テーブル |
| **T-02** UserNamekemono | U3（親密度・収穫日時）+ U2（新規出現時の実体化）+ U7（テーブル定義） | |
| **T-03** UserProgress | U2（スワイプ進行、pendingAppearances 登録）+ U3（pendingAppearances 実体化・削除）+ U4（所持品）+ U7（テーブル定義） | 複合キーで複数ドメインを収容 |
| **T-04** Letters | U3（手紙解放時に追加）+ U7（テーブル定義） | |
| **T-05** OtukaiHistory | U5（おつかい配信時追加、会話ログ更新、日記記録）+ U7（テーブル定義） | |

---

## 4. マスターデータ × 論理 Unit

| マスターデータ | 論理 Unit | 備考 |
|---|---|---|
| `cards.json`（全 30 種ダメカード） | U2 Swipe/NMK Core | |
| `namekemono.json`（ナマケモノ定義・出現条件・つぶやき） | U2（出現条件） + U3（つぶやき） | 論理的には 2 Unit 分担だが物理的には 1 ファイル |
| `items.json`（ショップアイテム） | U4 Shop/Inventory | |
| `letters.json`（手紙マスタ） | U3 Forest/Collection/Affinity | |
| `otukai_prompts.json`（キャラ別プロンプトテンプレート） | U5 Otukai/Bedrock | |
| `game-config.json`（NMK 計算、親密度、ガチャ等のバランスパラメータ） | U2/U3/U4 共有 | NFR-006 対応のため外部化 |

---

## 5. EventBridge ルール × 論理 Unit

| ルール | トリガー | 起動 Lambda | 担当 Unit |
|---|---|---|---|
| 夜通知 | 21:00 JST | L-08 night-notification | U6 LINE Notification/Webhook |
| 朝通知 | 8:00 JST | L-09 morning-notification | U6 LINE Notification/Webhook |
| おつかい朝 | 10:00 JST | L-10 otukai-scheduler | U5 Otukai/Bedrock |
| おつかい昼 | 13:00 JST | L-10 otukai-scheduler | U5 Otukai/Bedrock |
| おつかい夕 | 16:00 JST | L-10 otukai-scheduler | U5 Otukai/Bedrock |

ルール定義は U7 Infra、起動 Lambda は U5/U6 に帰属。

---

## 6. デモシナリオ × 論理 Unit

1.5 日体験の各シーンで必要な Unit を示す。書類審査・予選プレゼンで「どのシーンが何 Unit の成果か」を説明可能にする。

### Day 1 初回起動

| シーン | 必要な Unit | 備考 |
|---|---|---|
| LINE 友だち追加→初回起動 | U6（Follow）+ U1（LIFF 初期化）+ U7（Infra） | |
| 絵本オンボーディング（3 ページ） | U1（P-07 OnboardingDialog）+ U8（絵本イラスト） | |
| チュートリアルスワイプ（5 枚・強制ねむねむ/もぐもぐ） | U2（スワイプロジック分岐）+ U8（キャラ、カード） | |
| ネムリン召喚演出 | U3（U-06 SummonAnimation）+ U8（ネムリン画像） | |
| 図鑑にネムリン登録（実体化）、モグモグはシルエット→pending | U3（図鑑）+ U2（pendingAppearances 登録） | |
| ショップでおやつ購入 | U4（ショップ、所持品）+ U8（アイテムアイコン） | |
| 所持品ダイアログから贈り物→親密度 UP | U4（贈り物）+ U3（親密度表示、U-07 AffinityMeter） | |
| 「森に足跡が…」表示 | U3（UI 表示）+ U2（pendingAppearances の明示） | |

### Day 2 朝

| シーン | 必要な Unit | 備考 |
|---|---|---|
| LINE Push 通知「森に新しい仲間が来ているよ！」 | U6（朝通知 L-09）+ U7（EventBridge） | |
| アプリ再起動→モグモグ出現 | U3（pendingAppearances 実体化 = forest-api 呼び出し時）+ U8（モグモグ画像） | |
| 2 匹分の日次収穫（HarvestAnimation） | U3（日次収穫、U-09）+ U2（NMK 加算） | |
| ナマケモノタップでつぶやき表示 | U3（U-03 NamekemonoSprite、つぶやき表示） | |

### Day 2 夜

| シーン | 必要な Unit | 備考 |
|---|---|---|
| LINE Push 通知（ナマケモノ固有セリフ） | U6（夜通知 L-08、キャラランダム選択） | |
| スワイプ報告画面（通常 10 枚） | U2（SwipeCard、10 枚選出） | |
| 途中離脱→残り枚数 × 3NMK | U2（離脱判定、NMK 計算） | |

### Day 3 朝

| シーン | 必要な Unit | 備考 |
|---|---|---|
| 朝通知→新しいナマケモノ出現 | U6 + U3 + U8 | |
| 親密度★3 到達→手紙受信 | U3（手紙解放判定、U-08 LetterCard）+ U8（手紙演出） | |
| 手紙コレクション画面 | U3（P-05 LetterCollectionPage） | |

### 3 週間後の世界（プレゼン事前セットアップ）

| シーン | 必要な Unit | 備考 |
|---|---|---|
| LINE にモグモグから「コンビニで食べたことないお菓子買ってきて」 | U5（おつかいスケジューラ L-10、Bedrock 生成）+ U6（Webhook）+ U7 | |
| ユーザーが写真で返信→モグモグ AI 応答 | U5（LINE Webhook おつかい処理、画像 S3 保存、Bedrock 応答） | |
| アプリの森の日記に記録 | U5（P-06 DiaryPage、日記生成） | |

---

## 7. 全ストーリー・全コンポーネント網羅性の検証

### ストーリー網羅チェック

| カテゴリ | 対象 | 網羅状況 |
|---|---|---|
| MVP v1 ストーリー（US-01〜US-17） | 17 件 | ✅ 全件 Unit 割り当て済み |
| MVP v2 ストーリー（FS-01〜FS-03） | 3 件 | ✅ 全件 Unit 割り当て済み（データ構造のみ MVP v1 で対応） |

### コンポーネント網羅チェック

| カテゴリ | 対象 | 網羅状況 |
|---|---|---|
| フロントエンド画面（P-01〜P-07） | 7 件 | ✅ 全件 Unit 割り当て済み |
| UI コンポーネント（U-01〜U-10） | 10 件 | ✅ 全件 Unit 割り当て済み |
| Lambda（L-01〜L-12） | 12 件 | ✅ 全件 Unit 割り当て済み（L-01/L-05/L-11 は複数 Unit に跨る旨明記） |
| 共通モジュール（M-01〜M-06） | 6 件 | ✅ 全件 Unit 割り当て済み |
| インフラ（I-01〜I-08） | 8 件 | ✅ 全件 U7 に帰属 |
| DynamoDB テーブル（T-01〜T-05） | 5 件 | ✅ 全件 Unit 割り当て済み |
| EventBridge ルール | 5 件 | ✅ 全件 Unit 割り当て済み |
| マスターデータ | 6 件 | ✅ 全件 Unit 割り当て済み |

### 逆引きチェック: 各 Unit の責務合計

| Unit | 画面 | UI | Lambda | モジュール | インフラ | 帰属マスターデータ |
|---|---|---|---|---|---|---|
| U0 Contracts | — | — | — | M-06（仕様） | — | — |
| U1 LIFF Shell | P-07 | U-01, U-10 | — | — | — | — |
| U2 Swipe/NMK | P-02 | U-02, U-04 | L-02（+L-01副） | — | — | cards.json, namekemono.json（出現条件）, game-config 一部 |
| U3 Forest/Affinity | P-01, P-04, P-05 | U-03, U-06, U-07, U-08, U-09 | L-03, L-05（親密度・手紙）, L-06 | — | — | letters.json, namekemono.json（つぶやき）, game-config 一部 |
| U4 Shop | P-03 | U-05 | L-04, L-05（贈り物） | — | — | items.json, game-config 一部 |
| U5 Otukai | P-06 | — | L-07, L-10, L-11（おつかい部分） | M-05 | — | otukai_prompts.json |
| U6 LINE | — | — | L-08, L-09, L-11（Follow/Unfollow・署名検証・dispatch）, L-01（主） | M-02 | — | — |
| U7 Infra | — | — | L-12 | M-01, M-03（読み込み機構）, M-04（読み込み機構）, M-06（実装） | I-01〜I-08 | — |
| U8 Assets | — | — | — | — | — | —（画像素材のみ） |

全コンポーネントが過不足なく割り当てられている。
