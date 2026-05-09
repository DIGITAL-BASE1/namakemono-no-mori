# Execution Plan

## Detailed Analysis Summary

### Change Impact Assessment
- **User-facing changes**: Yes — 新規ゲームアプリ全体
- **Structural changes**: Yes — フロントエンド + バックエンド + インフラの新規構築
- **Data model changes**: Yes — DynamoDBテーブル設計が必要
- **API changes**: Yes — 全APIを新規設計
- **NFR impact**: 最小限（ハッカソンデモ用途、〜5ユーザー）

### Risk Assessment
- **Risk Level**: Medium
- **主なリスク**: LIFF開発経験の不足、AI（Bedrock）統合の複雑さ、2週間の期間制約
- **Rollback Complexity**: Easy（新規プロジェクトのため）
- **Testing Complexity**: Simple（ユニットテストのみ、ハッピーパス中心）

---

## Workflow Visualization

```mermaid
flowchart TD
    Start(["User Request"])
    
    subgraph INCEPTION["🔵 INCEPTION PHASE"]
        WD["Workspace Detection<br/><b>COMPLETED</b>"]
        RA["Requirements Analysis<br/><b>COMPLETED</b>"]
        US["User Stories<br/><b>COMPLETED</b>"]
        WP["Workflow Planning<br/><b>COMPLETED</b>"]
        AD["Application Design<br/><b>COMPLETED</b>"]
        UG["Units Generation<br/><b>COMPLETED</b>"]
    end
    
    subgraph CONSTRUCTION["🟢 CONSTRUCTION PHASE"]
        FD["Functional Design<br/><b>EXECUTE</b>"]
        NFRA["NFR Requirements<br/><b>SKIP</b>"]
        NFRD["NFR Design<br/><b>SKIP</b>"]
        ID["Infrastructure Design<br/><b>EXECUTE</b>"]
        CG["Code Generation<br/><b>EXECUTE</b>"]
        BT["Build and Test<br/><b>EXECUTE</b>"]
    end
    
    Start --> WD
    WD --> RA
    RA --> US
    US --> WP
    WP --> AD
    AD --> UG
    UG --> FD
    FD --> ID
    ID --> CG
    CG --> BT
    BT --> End(["Complete"])

    style WD fill:#4CAF50,stroke:#1B5E20,stroke-width:3px,color:#fff
    style RA fill:#4CAF50,stroke:#1B5E20,stroke-width:3px,color:#fff
    style US fill:#4CAF50,stroke:#1B5E20,stroke-width:3px,color:#fff
    style WP fill:#4CAF50,stroke:#1B5E20,stroke-width:3px,color:#fff
    style AD fill:#4CAF50,stroke:#1B5E20,stroke-width:3px,color:#fff
    style UG fill:#4CAF50,stroke:#1B5E20,stroke-width:3px,color:#fff
    style FD fill:#FFA726,stroke:#E65100,stroke-width:3px,stroke-dasharray: 5 5,color:#000
    style NFRA fill:#BDBDBD,stroke:#424242,stroke-width:2px,stroke-dasharray: 5 5,color:#000
    style NFRD fill:#BDBDBD,stroke:#424242,stroke-width:2px,stroke-dasharray: 5 5,color:#000
    style ID fill:#FFA726,stroke:#E65100,stroke-width:3px,stroke-dasharray: 5 5,color:#000
    style CG fill:#4CAF50,stroke:#1B5E20,stroke-width:3px,color:#fff
    style BT fill:#4CAF50,stroke:#1B5E20,stroke-width:3px,color:#fff
    style INCEPTION fill:#BBDEFB,stroke:#1565C0,stroke-width:3px,color:#000
    style CONSTRUCTION fill:#C8E6C9,stroke:#2E7D32,stroke-width:3px,color:#000
    style Start fill:#CE93D8,stroke:#6A1B9A,stroke-width:3px,color:#000
    style End fill:#CE93D8,stroke:#6A1B9A,stroke-width:3px,color:#000
    linkStyle default stroke:#333,stroke-width:2px
```

---

## Phases to Execute

### 🔵 INCEPTION PHASE
- [x] Workspace Detection (COMPLETED)
- [x] Requirements Analysis (COMPLETED)
- [x] User Stories (COMPLETED)
- [x] Workflow Planning (COMPLETED)
- [x] Application Design (COMPLETED)
  - **Rationale**: 新規アプリ。コンポーネント構成、API設計、DynamoDBテーブル設計、サービスレイヤー定義が必要
- [x] Units Generation (COMPLETED — 改訂版: 論理9Unit × 物理担当の二層構造)
  - **Rationale**: 外部レビュー（他ハッカソンチームとの比較採点）を受け、初版の4ユニット（FE/BE/Infra/Assets）から論理9 Unit × 物理担当マッピングに改訂。U0 Contracts（OpenAPI/共通型）、U1 LIFF Shell、U2 Swipe/NMK Core、U3 Forest/Collection/Affinity、U4 Shop/Inventory、U5 Otukai/Bedrock、U6 LINE Notification/Webhook、U7 Infrastructure、U8 Assets の 9 論理 Unit。物理担当はメンバーA（FE）/メンバーB（フルスタック+AWS）/メンバーC（画像・非同期）の3名で、担当レイヤー分離を主、ハイブリッド越境を保険とする

### 🟢 CONSTRUCTION PHASE（per-unit）

per-unit ループは論理 9 Unit について実施する。Q1=A 採択により、Functional Design は全論理 Unit で独立実施。承認ゲートは Functional Design / Code Generation の各終了時に計 18 回発生。

- [ ] Functional Design - **EXECUTE（9 Unit ごとに独立実施）**
  - **Rationale**: ゲームロジック（NMK計算、出現条件判定、親密度計算、日次収穫、pendingAppearances、おつかいAI、通知、ガードレール）の詳細設計が必要。9論理Unitの各ドメインで独立して設計密度を確保する
- [ ] NFR Requirements - **SKIP**
  - **Rationale**: ハッカソンデモ用途（〜5ユーザー）。パフォーマンス・スケーラビリティの詳細設計は不要
- [ ] NFR Design - **SKIP**
  - **Rationale**: NFR Requirementsをスキップするため
- [ ] Infrastructure Design - **EXECUTE（U7 Infrastructure に集約）**
  - **Rationale**: AWS CDK構成（Lambda、API Gateway、DynamoDB、S3、CloudFront、EventBridge、Secrets Manager）の詳細設計はU7の per-unit ループで実施
- [ ] Code Generation - **EXECUTE（9 Unit ごとに独立実施、ALWAYS）**
  - **Rationale**: 実装コード生成。各論理Unitで独立実施
- [ ] Build and Test - **EXECUTE**（ALWAYS）
  - **Rationale**: ビルド・テスト手順の生成

### per-unit ループの着手順（Tier 別）

| Tier | Unit | 着手タイミング | 完了目安 |
|---|---|---|---|
| 🔴 Tier 1 | U0 Contracts, U7 Infra スケルトン, U1 LIFF Shell | 最優先 | Day 1-3 |
| 🔴 Tier 2 | U2 Swipe/NMK Core | Tier 1 完了後 | Day 5 |
| 🟠 Tier 3 | U3 Forest, U4 Shop（並列可） | Tier 2 完了後 | Day 7-8 |
| 🟡 Tier 4 | U5 Otukai, U6 LINE（並列可） | Tier 3 完了後 | Day 10-11 |
| 独立 | U8 Assets | 常時並列（非同期担当 C） | Day 1-14 通期 |

---

## Success Criteria
- **Primary Goal**: 1.5日体験が動作するMVPをAWSにデプロイし、プレゼン動画を撮影できる状態にする
- **Key Deliverables**:
  - 動作するLIFFアプリ（React + Vite）
  - バックエンドAPI（Lambda + API Gateway）
  - DynamoDBにデータ永続化
  - LINE Push通知が動作
  - おつかいAI（Bedrock）が応答
  - AWS CDKでデプロイ可能
- **Quality Gates**:
  - ハッピーパスが完全に動作する
  - プレゼン動画撮影に必要な全シーンが再現可能
  - ユニットテストがビジネスロジックをカバー
