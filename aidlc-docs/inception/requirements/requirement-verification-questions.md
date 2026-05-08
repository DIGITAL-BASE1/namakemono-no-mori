# 要件確認質問

以下の質問に回答してください。各質問の [Answer]: タグの後に選択肢の文字を記入してください。
どの選択肢にも当てはまらない場合は、最後の選択肢（Other）を選び、[Answer]: タグの後に説明を記述してください。

---

## Question 1
MVP v1のスコープとして、READMEに記載されている全機能を含めますか？それとも段階的にリリースしますか？

A) READMEに記載のMVP v1スコープ全体を一度に実装する（スワイプ報告、ナマケモノ出現、ショップ、親密度、手紙、おつかい全て）
B) コアループのみ先行実装（スワイプ報告 → ナマケコイン → ナマケモノ出現 → 日次収穫）、おつかい・LINE連携は後続フェーズ
C) フロントエンドのみ先行（バックエンドはモック）でプロトタイプを作る
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

## Question 2
LINE連携（LIFF）の実装範囲はどこまでですか？

A) LIFFアプリとしてフル実装（認証、通知、おつかいトーク全て）
B) LIFFで認証のみ、通知はPush API、おつかいトークはMessaging API
C) 初期はWebアプリとして独立実装し、LINE連携は後から追加
X) Other (please describe after [Answer]: tag below)

[Answer]: X AとBの違いが良くわかりません。解説した上でもう一回質問してください。

---

## Question 3
おつかい機能のAIエージェント（Amazon Bedrock）の実装レベルはどの程度ですか？

A) フル実装（おつかい生成 + パーソナライズ + 応答生成 + 達成判定 + ガードレール全て）
B) 基本実装（おつかい生成 + 応答生成 + 達成判定のみ、ガードレールは簡易版）
C) MVP最小限（事前定義のおつかいテンプレートから選択、AI応答のみ動的生成）
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

## Question 4
ナマケモノのイラスト・画像素材はどのように用意しますか？

A) 既に素材が用意されている（assetsディレクトリに配置予定）
B) プレースホルダー画像で開発し、後から差し替える
C) AI生成画像を使用する
X) Other (please describe after [Answer]: tag below)

[Answer]: X 画像作成担当がチームにいます。必要な画像は担当者が作成します。

---

## Question 5
ユーザーデータの永続化について、DynamoDBのテーブル設計の方針は？

A) シングルテーブルデザイン（1テーブルに全エンティティを格納）
B) エンティティごとにテーブルを分ける（ユーザー、ナマケモノ、おつかい等）
C) 設計はAI-DLCのApplication Designフェーズに委ねる
X) Other (please describe after [Answer]: tag below)

[Answer]: C

---

## Question 6
デプロイ環境とCI/CDについて、どの程度の構成を想定していますか？

A) 本番環境のみ（AWS CDKで直接デプロイ）
B) 開発環境 + 本番環境（ステージ分離あり）
C) 開発環境のみ（ハッカソン・プロトタイプ用途）
X) Other (please describe after [Answer]: tag below)

[Answer]: C

---

## Question 7
フロントエンドのUI/UXデザインについて、どのアプローチを取りますか？

A) デザインシステム（Tailwind CSS等）を使い、モバイルファーストで実装
B) コンポーネントライブラリ（MUI、Chakra UI等）を活用
C) カスタムCSS/SCSSでオリジナルデザイン
D) デザインはAI-DLCのDesignフェーズに委ねる
X) Other (please describe after [Answer]: tag below)

[Answer]: D

---

## Question 8
テスト戦略について、どのレベルまで実装しますか？

A) ユニットテスト + 統合テスト + E2Eテスト
B) ユニットテスト + 統合テストのみ
C) ユニットテストのみ
D) テストはMVP後に追加する
X) Other (please describe after [Answer]: tag below)

[Answer]: C

---

## Question 9
同時接続ユーザー数・スケーラビリティの想定は？

A) 小規模（〜100ユーザー、ハッカソンデモ用途）
B) 中規模（〜1,000ユーザー、限定公開）
C) 大規模（10,000+ユーザー、一般公開）
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

## Question 10
開発チームの構成と開発期間の想定は？

A) 1人開発、1〜2週間（ハッカソン）
B) 1人開発、1ヶ月以上
C) チーム開発（2人以上）、1ヶ月以上
X) Other (please describe after [Answer]: tag below)

[Answer]: X 開発期間は約2週間でメンバーは3人ですが、内1名（画像作成担当）が海外出張でほとんどの期間が不在となります。ハッカソンのレギュレーションはdocs\HACKATHON.mdを確認してください

---

## Question 11: Security Extensions
セキュリティ拡張ルールをこのプロジェクトに適用しますか？

A) Yes — 全てのセキュリティルールをブロッキング制約として適用する（本番グレードのアプリケーション向け推奨）
B) No — セキュリティルールをスキップする（PoC、プロトタイプ、実験的プロジェクト向け）
X) Other (please describe after [Answer]: tag below)

[Answer]: B

---

## Question 12: Property-Based Testing Extension
プロパティベーステスト（PBT）ルールをこのプロジェクトに適用しますか？

A) Yes — 全てのPBTルールをブロッキング制約として適用する（ビジネスロジック、データ変換、シリアライゼーション、ステートフルコンポーネントを持つプロジェクト向け推奨）
B) Partial — 純粋関数とシリアライゼーションのラウンドトリップにのみPBTルールを適用する（アルゴリズムの複雑さが限定的なプロジェクト向け）
C) No — 全てのPBTルールをスキップする（シンプルなCRUDアプリ、UIのみのプロジェクト、重要なビジネスロジックのない薄い統合レイヤー向け）
X) Other (please describe after [Answer]: tag below)

[Answer]: C

---
