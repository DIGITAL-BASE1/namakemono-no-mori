# Story Generation Plan

## 方針

READMEおよび要件定義書（requirements.md）に基づき、ユーザーストーリーを生成する。

---

## 質問

以下の質問に回答してください。各質問の [Answer]: タグの後に選択肢の文字を記入してください。

### Question 1
ペルソナの数はどうしますか？

A) 1人（メインユーザーのみ）
B) 2人（メインユーザー + 開発者/運営者）
C) 3人以上（メインユーザー + サブユーザー + 運営者）
X) Other (please describe after [Answer]: tag below)

[Answer]: X 3人のメインユーザーにしてください

---

### Question 2
ストーリーの分割方針はどうしますか？

A) 機能ベース — 各機能（スワイプ報告、ショップ、親密度等）ごとにストーリーを作成
B) 時系列ベース — 1.5日体験の流れに沿ってストーリーを作成（Day1初回→Day1夜→Day2朝→…）
C) ハイブリッド — Epicは機能ベース、個別ストーリーは時系列の体験順で整理
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

### Question 3
ストーリーの粒度はどの程度にしますか？

A) 粗い（5〜8ストーリー。Epic相当の大きな単位）
B) 中程度（10〜15ストーリー。操作単位）
C) 細かい（16〜25ストーリー。画面遷移・アクション単位）
X) Other (please describe after [Answer]: tag below)

[Answer]: B

---

### Question 4
受け入れ基準のフォーマットはどうしますか？

A) Given-When-Then形式（BDD形式）
B) チェックリスト形式（箇条書き）
C) 自然言語の説明文
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

### Question 5
おつかい機能（LINE上のAIエージェント会話）のストーリーはどう扱いますか？

A) MVP v1のストーリーとして含める（親密度MAXで解禁される体験として）
B) MVP v2のストーリーとして別セクションに記載
C) 将来機能として軽く言及のみ
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

### Question 6
デモシナリオ（プレゼン用の1.5日体験フロー）とストーリーの対応関係を明示しますか？

A) はい — 各ストーリーにデモシナリオのどのシーンに対応するか記載
B) いいえ — ストーリーは独立して記述し、デモシナリオとの対応は別途管理
X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

## 生成計画チェックリスト

回答を受領後、以下の手順でストーリーを生成します：

- [x] Step 1: ペルソナ作成（personas.md）
- [x] Step 2: Epic定義
- [x] Step 3: ユーザーストーリー作成（INVEST基準準拠）
- [x] Step 4: 受け入れ基準の記述
- [x] Step 5: ストーリー間の依存関係整理
- [x] Step 6: デモシナリオとの対応マッピング（選択された場合）
- [x] Step 7: MVP v2 / 将来ストーリーの記載

---
