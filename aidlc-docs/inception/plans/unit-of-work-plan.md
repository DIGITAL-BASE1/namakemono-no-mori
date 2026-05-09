# Unit of Work Plan（改訂版）

## 経緯

本プランは Units Generation のやり直し版です。初版（4ユニット: FE/BE/Infra/Assets）は外部レビューにて以下の指摘を受けました。

- U2 Backend が肥大化し、per-unit ループの粒度として重すぎる（Lambda 12本、6サービス、Bedrock、LINE、ゲームロジック全部入り）
- 機能ドメインと Unit の対応が設計書上で見えない
- 共通契約（OpenAPI/Shared Types）が Unit として独立していない
- Unit 毎の完了条件（DoD）が不明確
- Component → Unit の対応表・優先度 Tier・Fallback が弱い

ユーザーとの議論を経て、以下の方針で改訂します。

---

## 改訂方針: 論理 Unit × 物理担当の二層構造

### 核心概念

| 概念 | 定義 |
|---|---|
| **論理 Unit** | AI-DLC の per-unit ループの単位。機能ドメインで分割し、設計の密度・トレーサビリティ・テスト境界を担保する。 |
| **物理担当** | 現実のチーム構成に基づく担当マッピング。1 人が複数論理 Unit を担当することもあれば、複数人が 1 論理 Unit を協働することもある。 |

### チーム構成（3名）

| メンバー | 役割 | 技能 | 稼働 |
|---|---|---|---|
| メンバーA（今野さん） | フロントエンド開発 | FE寄り、AWSは初心者 | 同期 |
| メンバーB（PM） | フルスタック開発＋インフラ＋AI連携 | FE/BE両方、AWS精通 | 同期 |
| メンバーC | 画像・イラスト制作 | 画像系に強い | 非同期（海外出張中だから非同期稼働に適する。最悪A/Bで巻き取り可） |

### 協働スタイル: スタイル1（担当レイヤー分離）を主、スタイル3（ハイブリッド）を保険

- AWS が絡む領域（Contracts / Otukai-Bedrock / LINE / Infra）は全て B が主担当。A の AWS 学習コストを進捗クリティカルパスから外す
- 機能ドメイン Unit は FE側=A / BE側=B で分離。OpenAPI 契約を介して並列実装
- U1 LIFF Shell は A が専任
- U8 Assets は C が完全非同期

---

## 論理 Unit 構成（9 Unit）

| # | 論理 Unit | 主担当 | 概要 |
|---|---|---|---|
| U0 | Contracts（OpenAPI / Shared Types / 認証契約） | B | FE/BE 共通契約。先行作成により並列開発を成立させる |
| U1 | LIFF Frontend Shell | A | LIFF初期化、ルーティング、Context基盤、認証クライアント側 |
| U2 | Swipe / NMK Core | FE:A / BE:B | スワイプ報告、途中離脱判定、NMK計算、出現条件判定 |
| U3 | Forest / Collection / Affinity | FE:A / BE:B | 森状態、図鑑、親密度、手紙、日次収穫、つぶやき |
| U4 | Shop / Inventory | FE:A / BE:B | ショップ、所持品、贈り物、親密度UP連動 |
| U5 | Otukai / Bedrock | FE:A / BE:B | おつかい生成・応答・達成判定、森の日記、Bedrock連携、ガードレール |
| U6 | LINE Notification / Webhook | B | LINE Messaging API、署名検証、夜通知、朝通知、フォロー/アンフォロー |
| U7 | Infrastructure | B | CDK、DynamoDB、EventBridge、S3+CloudFront、API Gateway、IAM、Secrets Manager |
| U8 | Assets | C | ナマケモノ立ち絵、アイテムアイコン、森背景、UI素材、オンボーディング絵本 |

**補足**: U2〜U5 の BE 側は全て B が実装するが、論理 Unit としては機能ドメインで分離する。これにより per-unit ループ（Functional Design）は Unit ごとに独立して回せる。Code Generation は同一担当者が連続する場合に束ねる運用も可とする。

---

## 残存する確認事項

以下の質問にお答えください。`[Answer]:` の後に回答を記入してください。

### Q1: per-unit ループの回し方

同一物理担当（B）が複数論理 Unit（U0, U5, U6, U7 など）を担当する場合、Functional Design をどう回しますか？

A) 全論理 Unit で独立に Functional Design → 独立に Code Generation（原則忠実、per-unit ループ回数最多）
B) Functional Design は論理 Unit 毎に独立、Code Generation は担当者単位で束ねて実行（設計の密度を保ちつつ実装効率を確保）
C) 担当者が同じ連続論理 Unit は Functional Design も Code Generation も束ねる（per-unit ループ回数最小）

[Answer]: A

---

### Q2: 優先度 Tier の考え方

CONSTRUCTION フェーズでの着手順をどう設計しますか？

A) 契約先行・FE基盤と並列: Tier1 = U0+U1+U7スケルトン → Tier2 = U2 → Tier3 = U3,U4 → Tier4 = U5,U6
B) 垂直スライス: デモシナリオ Day1 に必要な最小機能（オンボーディング+スワイプ+初回ナマケモノ）を全 Unit 横断で先に作る
C) Tier 1/2/3/4 は A と同じだが、Tier2 以降の順序はチームの肌感で柔軟に決める（ゆるめ）

[Answer]: A

---

### Q3: U8 Assets の位置づけ明記

U8 Assets は「設計 Unit」ではなく「制作分担 Unit」としての性質が強いです。この違いを明示しますか？

A) 明示する（審査員に意図が伝わる）。他の論理 Unit とは別セクションで記述
B) 明示しない（シンプルに全 9 Unit を横並びで記述）

[Answer]: A

---

### Q4: Fallback シナリオの範囲

担当者が稼働できなくなった場合の Fallback 計画をどこまで書きますか？

A) 全論理 Unit について「メインが倒れた時の代替担当」を明記
B) クリティカルパス上の Unit（U0, U1, U7）のみ明記
C) Fallback は大枠のみ（「A倒れ→B引き継ぎ」「B倒れ→A引き継ぎ（AWS学習コスト付き）」「C倒れ→A/Bで画像巻き取り」）

[Answer]: C

---

### Q5: Component → Unit マトリクスの粒度

components.md の全コンポーネント（7画面、10 UI、11 Lambda、4共通モジュール、8インフラ）を Unit にマッピングします。

A) 全コンポーネントを表に展開（精密）
B) 主要コンポーネントのみ表に展開、細部は文章で補足（中程度）
C) カテゴリ単位（画面群 / UIコンポーネント群 / Lambda群 等）でマッピング（粗め）

[Answer]: A

---

## 設計プラン（承認後に実行）

- [x] 既存 `unit-of-work.md` を改訂版で上書き（設計意図、9論理Unit、物理担当マッピング、DoD、代替案検討と却下理由、コード構成戦略）
- [x] 既存 `unit-of-work-dependency.md` を改訂版で上書き（論理Unit間依存DAG、物理担当の並列性、Fallback、優先度Tier、クリティカルパス）
- [x] 既存 `unit-of-work-story-map.md` を改訂版で上書き（3軸マトリクス再構築、Component→Unit 対応表追加、デモシナリオ対応）
- [x] `execution-plan.md` の CONSTRUCTION per-unit ループ見通しを 9 論理 Unit で更新
- [x] ユニット境界と依存関係の再検証
- [x] 全ストーリー（US-01〜US-17）がユニットに割り当てられていることの再検証
