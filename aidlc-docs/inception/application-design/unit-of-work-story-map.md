# Unit of Work Story Map

## ストーリー → ユニット対応（3軸マトリクス）

各ストーリーを画面（FE）・API/ロジック（BE）・インフラ・アセットの軸で分解し、実装範囲を可視化する。

### 凡例
- ◎ 主責務
- ○ 関連あり
- — 不要

---

## Epic 1: オンボーディング・初回体験

| ストーリー | FE（画面） | BE（API/ロジック） | Infra | Assets |
|---|---|---|---|---|
| **US-01 絵本オンボーディング** | ◎ OnboardingDialog（3ページ） | ○ getUser（初回判定）、completeOnboarding | — | ◎ 絵本イラスト×3 |
| **US-02 初回スワイプ体験** | ◎ SwipeReportPage、SwipeCard | ◎ startSwipe（isFirstTime=true、固定配置） | — | ○ カードUI |
| **US-03 初回ナマケモノ召喚** | ◎ ForestPage、SummonAnimation | ◎ submitSwipeResult（ネムリン即時出現特例） + getForestState | — | ◎ ネムリン、モグモグ立ち絵 |

---

## Epic 2: スワイプ報告

| ストーリー | FE（画面） | BE（API/ロジック） | Infra | Assets |
|---|---|---|---|---|
| **US-04 通常スワイプ報告** | ◎ SwipeReportPage、SwipeCard | ◎ startSwipe、submitSwipeResult、GameCoreService（NMK計算、checkAppearanceConditions、pending登録） | — | ○ カードUI、コイン演出 |
| **US-05 途中離脱ボーナス** | ◎ SwipeReportPage（×ボタン） | ◎ submitSwipeResult（abandoned=true、途中離脱NMK計算、メンドクサウルス条件カウント） | — | — |

---

## Epic 3: ナマケコイン経済圏

| ストーリー | FE（画面） | BE（API/ロジック） | Infra | Assets |
|---|---|---|---|---|
| **US-06 ショップでアイテム購入** | ◎ ShopPage | ◎ getShopItems、purchaseItem、ShopService | — | ◎ アイテムアイコン |
| **US-07 日次コイン収穫** | ◎ ForestPage（HarvestAnimation） | ◎ harvestCoins、GameCoreService.calculateHarvest（親密度ボーナス含む） | — | ○ コイン、収穫演出 |
| **US-08 放置ボーナス蓄積** | ○ ForestPage（初回ログイン時表示） | ◎ getUser、GameCoreService.calculateAndApplyIdleBonus | — | — |

---

## Epic 4: ナマケモノ・コレクション

| ストーリー | FE（画面） | BE（API/ロジック） | Infra | Assets |
|---|---|---|---|---|
| **US-09 ナマケモノ出現・図鑑登録** | ◎ ForestPage、EncyclopediaPage、SummonAnimation | ◎ getForestState（resolvePendingAppearances）、getEncyclopedia | — | ◎ 全5体立ち絵 + シルエット |
| **US-10 ナマケモノのつぶやき** | ◎ NamekemonoSprite（タップ） | ◎ getWhisper、master-data（つぶやき5種/キャラ） | — | — |

---

## Epic 5: 親密度・手紙

| ストーリー | FE（画面） | BE（API/ロジック） | Infra | Assets |
|---|---|---|---|---|
| **US-11 アイテム贈呈・親密度UP** | ◎ ForestPage、InventoryDialog、AffinityMeter | ◎ giftItem、AffinityService、ShopService.consumeItem | — | ○ 親密度演出 |
| **US-12 手紙を受け取る** | ◎ LetterCollectionPage、LetterCard | ◎ giftItem（手紙解放判定）、getLetters、master-data（手紙テンプレート） | — | ○ 手紙UI |

---

## Epic 6: 通知

| ストーリー | FE（画面） | BE（API/ロジック） | Infra | Assets |
|---|---|---|---|---|
| **US-13 夜のスワイプ通知** | — | ◎ night-notification Lambda、NotificationService、LINE Push | ◎ EventBridge（21:00）、LINE Messaging API権限 | — |
| **US-14 朝の到着通知** | — | ◎ morning-notification Lambda、pendingAppearances判定 | ◎ EventBridge（8:00） | — |

---

## Epic 7: おつかい（AIエージェント）

| ストーリー | FE（画面） | BE（API/ロジック） | Infra | Assets |
|---|---|---|---|---|
| **US-15 おつかい受信** | — | ◎ otukai-scheduler Lambda、OtukaiService.generateOtukai（Bedrock）、天気API/祝日API連携、LINE Push | ◎ EventBridge×3（10/13/16:00）、Bedrock IAM、Secrets Manager | — |
| **US-16 おつかい報告・AI応答** | — | ◎ line-webhook Lambda、OtukaiService.processReply（テキスト/画像）、s3-client、Bedrock（マルチモーダル） | ◎ API Gateway（Webhook URL）、S3（画像保存）、Bedrock IAM | — |
| **US-17 森の日記** | ◎ DiaryPage | ◎ getDiaryEntries、OtukaiService.recordDiary | — | ○ 日記UI |

---

## 横断的要素（全ストーリー共通）

| 要素 | FE | BE | Infra | Assets |
|---|---|---|---|---|
| **タブナビゲーション** | ◎ TabBar | — | — | ○ タブアイコン |
| **認証（LIFF）** | ◎ LIFF初期化、ID token取得 | ◎ M-06 auth（token検証） | ○ LIFF ID設定 | — |
| **LINE Webhook** | — | ◎ handleWebhook、handleFollow、handleUnfollow | ◎ API Gateway（Webhook URL）、署名検証設定 | — |
| **開発用API** | ○ デバッグ画面（任意） | ◎ dev-api（L-12） | ○ 共有シークレット設定 | — |
| **森の成長段階表示** | ◎ ForestPage背景切替 | — | — | ◎ 森の背景×4段階 |
| **CSSアニメーション** | ◎ 呼吸感、揺れ、zzz | — | — | — |

---

## ユニット別ストーリー集約

### U1 Frontend（主責務）
- US-01, US-02, US-03, US-04, US-05, US-06, US-07, US-09, US-10, US-11, US-12, US-17
- 12ストーリー（ほぼ全てのUI寄りストーリー）

### U2 Backend（主責務）
- US-02, US-03, US-04, US-05, US-06, US-07, US-08, US-09, US-10, US-11, US-12, US-13, US-14, US-15, US-16, US-17
- 16ストーリー（ほぼ全てにAPI/ロジックが絡む）

### U3 Infrastructure（主責務）
- US-13, US-14, US-15, US-16
- 4ストーリー（EventBridge・Bedrock・Webhook・S3が絡むもの）

### U4 Assets（主責務）
- US-01, US-03, US-06, US-09
- 4ストーリー（新規画像アセットが必要なもの）

---

## ストーリー完成判定

各ストーリーが「Done」となるための最低条件：
- 主責務の◎がついたユニット全てで実装完了
- 受け入れ基準（Given-When-Then）が全てパス
- デモシナリオに沿った動作確認完了

### デモシナリオとストーリーの対応

| デモシナリオ | 関連ストーリー |
|---|---|
| Day 1 初回起動 | US-01, US-02, US-03, US-05, US-06, US-11 |
| Day 2 朝 | US-07, US-09, US-10, US-14 |
| Day 2 夜 | US-04, US-13 |
| Day 3 | US-12 |
| 3週間後の世界（事前セットアップ） | US-15, US-16, US-17 |
