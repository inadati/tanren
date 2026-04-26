# CLAUDE.md

このファイルは Claude Code が tanren リポジトリで作業する際のガイダンスを提供する。

---

## プロジェクト概要

**種別**: Claude Code プラグイン
**目的**: MCTS（モンテカルロ木探索）× PRM（プロセス報酬モデル）駆動の品質鍛錬ハーネス。
コード・ブログ記事・スライド等あらゆる成果物に対して汎用的に使える自律的品質改善ループ。

---

## ディレクトリ構造

```
tanren/
├── plugins/
│   └── tanren/
│       └── skills/
│           ├── tanren/
│           │   └── SKILL.md       ← 準備スキル（/tanren）
│           └── run/
│               └── SKILL.md       ← 実行スキル（/tanren:run）
├── CLAUDE.md（このファイル）
└── README.md
```

---

## スキル一覧

| スキル | 呼び出し | 役割 |
|--------|---------|------|
| `tanren` | `/tanren` | 準備フェーズ。wantree〜PRM評価軸レビューまで |
| `run` | `/tanren:run` | 実行フェーズ。MCTSループによる自律的品質改善 |

---

## フロー概要

```
[人間] /tanren 起動
  └── wantree（対話的ヒアリング）
  └── 成果物プランニング（EnterPlanMode）
  └── plan-review AUTO-TRIGGER（1回目）
  └── PRM評価軸設計（EnterPlanMode）
  └── plan-review AUTO-TRIGGER（2回目）
  └── MEMORY.md に記録 → ユーザーに tanren:run 案内

[人間] コンテキストリセット後 /tanren:run 起動
  └── MEMORY.md からプラン読み込み
  └── MCTSループ（完全自動）
        Selection（UCB1）
        → Expansion（実装Agent並列）
        → PRM評価（3評価Agent並列）
        → Rollout（実装Agent並列）
        → Backpropagation
        → 収束判定
  └── 最良成果物の採用・提示
```

---

## 設計思想

### なぜMCTS × PRMか

- **MCTS**: UCB1アルゴリズムにより探索と活用のトレードオフを数学的に保証。最良パスを効率的に発見する
- **PRM（Process Reward Model）**: 最終成果物だけでなく各ステップを評価することで、早期に誤った方向を検出・修正できる
- **組み合わせ**: MCTSがPRMをノード価値関数として使うことで、AlphaGo/rStar-Mathと同等の探索効率を実現する

### なぜ準備と実行を分離するか

コンテキストをリセットしてから実行フェーズに入ることで：
1. 準備フェーズの長い会話がMCTSループのノイズにならない
2. 実行フェーズが常にクリーンな状態から始まる
3. MEMORY.md経由でプランを永続化することでセッションを跨げる

### 評価者が3名固定の理由

- 奇数なので多数決が成立する
- PRM評価軸設計フェーズで定義した3つの観点を1人ずつ担当
- コスト（MCTSループで何度も走る）と精度のバランス

### 実装Agentを最大並列化する理由

MCTSのExpansionとRolloutは実装の並列度が高いほど速い。
役割・数を固定せず「可能な限り並列」にすることで、成果物の種類に関わらず最大効率を発揮する。

---

## ターゲットプロジェクト側の生成物

`/tanren` を実行したプロジェクトルートに以下が生成される:

```
プロジェクトルート/
├── .wantree/
│   └── 0/
│       └── wantree.yml          ← 要件定義
└── .tanren/
    ├── plans/
    │   └── prm-criteria.yml     ← PRM評価軸定義
    ├── tree.yml                 ← MCTSツリー状態
    └── snapshots/               ← 各ノードの成果物スナップショット
        └── <node-id>.*
```

---

## 開発履歴

### 2026-04-26（v0.1.0）

- tanren ハーネス設計・初回実装
- 準備スキル（/tanren）と実行スキル（/tanren:run）を作成
- i-harnessの設計を踏まえ、MCTS × PRMベースに全面刷新
- 汎用成果物対応（コード・ブログ・スライド等）
