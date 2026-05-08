# 要件確認質問

AWSハッカソン「人をダメにするサービス」のアイデアと要件を明確にするため、以下の質問にお答えください。
各質問の `[Answer]:` タグの後に、選択肢のアルファベットを記入してください。

---

## Question 1
6つのパターンのうち、どの方向性でサービスを作りたいですか？（複数選択可：カンマ区切りで記入）

A) 便利すぎて自分でやらなくなる系（例：考えなくても全部やってくれる）
B) 快適すぎて動けなくなる系（例：ソファから出られない的な体験）
C) 中毒性があってやめられなくなる系（例：ついつい見続けてしまう）
D) 面倒なことを極限まで排除する系（例：あらゆる手間をゼロにする）
E) ユーモア・ネタ寄りで笑いを取りに行く系（例：ダメ人間度を競う）
F) ダメな自分を肯定する系（例：メンタルヘルス・頑張り過ぎない）
G) まだ決めていない — アイデア出しから一緒に考えたい
X) Other (please describe after [Answer]: tag below)

[Answer]: F

## Question 2
ハッカソンの制約条件について教えてください。開発期間はどのくらいですか？

A) 1日（日帰りハッカソン）
B) 2日間（週末ハッカソン）
C) 1週間程度
D) 2週間以上
X) Other (please describe after [Answer]: tag below)

[Answer]: D

## Question 3
チーム構成を教えてください。

A) 1人（ソロ参加）
B) 2〜3人チーム
C) 4〜5人チーム
D) 6人以上のチーム
X) Other (please describe after [Answer]: tag below)

[Answer]: B

## Question 4
チームの技術スキルセットで最も得意な領域はどれですか？（複数選択可：カンマ区切りで記入）

A) フロントエンド（React, Vue, Next.js など）
B) バックエンド（Python, Node.js, Java など）
C) モバイルアプリ（iOS, Android, Flutter など）
D) AI/ML（機械学習、生成AI、LLM など）
E) インフラ/クラウド（AWS サービス全般）
F) デザイン/UX
X) Other (please describe after [Answer]: tag below)

[Answer]: X 現時点では技術スタックに囚われたくない

## Question 5
AWSサービスの使用に関して、特に使いたい・使う予定のサービスはありますか？

A) Amazon Bedrock（生成AI）
B) AWS Lambda + API Gateway（サーバーレス）
C) Amazon DynamoDB / RDS（データベース）
D) Amazon S3 + CloudFront（静的ホスティング）
E) 特にこだわりなし — 最適なものを提案してほしい
X) Other (please describe after [Answer]: tag below)

[Answer]: X 現時点では使うAWSサービスに囚われたくない

## Question 6
ハッカソンの審査基準で重視されるポイントは何ですか？（わかる範囲で）

A) 技術的な完成度・実装力
B) アイデアの独創性・面白さ
C) ビジネス的な実用性・市場性
D) プレゼンテーション・デモのインパクト
E) 審査基準は不明 — 一般的なハッカソン基準で考えたい
X) Other (please describe after [Answer]: tag below)

[Answer]: A～Dのすべて

## Question 7
ターゲットユーザーのイメージはありますか？

A) 社会人（仕事で疲れている人）
B) 学生（勉強や就活で疲れている人）
C) エンジニア・開発者
D) 特定のターゲットなし — 幅広い層
E) まだ決めていない
X) Other (please describe after [Answer]: tag below)

[Answer]: D　広く世の中に提案できるものが良いと思います

## Question 8
サービスの形態として、どのようなものを想定していますか？

A) Webアプリケーション
B) モバイルアプリ
C) LINE Bot / Slack Bot などのチャットボット
D) Chrome拡張 / ブラウザ拡張
E) まだ決めていない — 最適な形態を提案してほしい
X) Other (please describe after [Answer]: tag below)

[Answer]: X 現時点ではサービス形態に囚われたくない

## Question 9
「優勝を目指す」とのことですが、過去のハッカソンで印象に残った受賞作品や、参考にしたいサービスはありますか？

A) ある（具体的に [Answer]: の後に記述してください）
B) 特にない — ゼロから考えたい
C) ハッカソン参加は初めて
X) Other (please describe after [Answer]: tag below)

[Answer]: B

## Question 10
アイデア出しのアプローチについて、どのように進めたいですか？

A) 6パターンそれぞれに対して複数のアイデアを出してもらい、その中から選びたい
B) 特定のパターン（Question 1で選んだもの）に絞って深掘りしたい
C) パターンにこだわらず、テーマに対して自由にアイデアを出してほしい
D) 自分のアイデアの種がある — それをブラッシュアップしたい（[Answer]: の後に記述してください）
X) Other (please describe after [Answer]: tag below)

[Answer]: B

---

## Question 11: セキュリティ拡張
このプロジェクトにセキュリティ拡張ルールを適用しますか？

A) はい — すべてのセキュリティルールをブロッキング制約として適用する（本番グレードのアプリケーション向け推奨）
B) いいえ — セキュリティルールをスキップする（PoC、プロトタイプ、実験的プロジェクト向け）
X) Other (please describe after [Answer]: tag below)

[Answer]: B

## Question 12: プロパティベーステスト拡張
このプロジェクトにプロパティベーステスト（PBT）ルールを適用しますか？

A) はい — すべてのPBTルールをブロッキング制約として適用する（ビジネスロジック、データ変換、シリアライゼーション、ステートフルコンポーネントを持つプロジェクト向け推奨）
B) 部分的 — 純粋関数とシリアライゼーションのラウンドトリップにのみPBTルールを適用する
C) いいえ — PBTルールをスキップする（シンプルなCRUDアプリ、UIのみのプロジェクト、薄い統合レイヤー向け）
X) Other (please describe after [Answer]: tag below)

[Answer]: C
