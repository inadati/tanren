# mcts-harness

MCTS（モンテカルロ木探索）× PRM（プロセス報酬モデル）駆動の品質鍛錬ハーネス。

コード・ブログ記事・スライド等あらゆる成果物に対して使える、自律的品質改善ループを Claude Code に追加するプラグイン。

## 概要

従来の「一発生成」ではなく、AlphaGo / rStar-Math と同様のアプローチで成果物を反復的に鍛え上げる。

- **MCTS（UCB1）** — 探索と活用のトレードオフを数学的に最適化し、最良の実装パスを効率的に発見する
- **PRM（プロセス報酬モデル）** — 最終成果物だけでなく各ステップを評価し、誤った方向を早期に検出・修正する
- **悪魔の代弁者エージェント** — 評価基準を表面的に満たすだけの「報酬ハッキング」を検出し、スコアを補正する

## インストール

### Claude Code プラグインとしてインストール

```bash
claude plugin install https://github.com/inadati/mcts-harness
```

### 手動インストール

```bash
# リポジトリをクローン
git clone https://github.com/inadati/mcts-harness

# プラグインをインストール
cd mcts-harness
claude plugin install ./plugins/mcts-harness
```

インストール後、Claude Code を再起動すると `/mcts-harness` と `/mcts-harness:run` の 2 つのスキルが使えるようになる。

## 使い方

### ステップ 1: 準備フェーズ

品質改善したい成果物のプロジェクトルートで起動する。

```
/mcts-harness
```

対話形式で以下を行う:

1. **wantree ヒアリング** — 要件定義（何を作りたいか）
2. **成果物プランニング** — 実装方針・構成・ステップを策定
3. **PRM 評価軸設計** — 3名の評価者の採点基準を設計（厳格な品質ゲート）

完了するとコンテキストリセットを促される。

### ステップ 2: 実行フェーズ

コンテキストをリセット後、同じプロジェクトルートで起動する。

```
/mcts-harness:run
```

以降は完全自動で MCTS ループが走る:

```
Selection（UCB1でノード選択）
  → Expansion（3候補を並列生成）
  → Rollout（成果物を並列実装）
  → 悪魔の代弁者検証（報酬ハッキング検出）
  → PRM 評価（3評価エージェント並列）
  → Backpropagation（スコアを伝播）
  → 収束判定（threshold 到達 or 上限回数）
```

合格ライン（デフォルト 0.90）を超えたら自動収束し、最良成果物を提示する。

## 生成ファイル

実行すると以下がプロジェクトルートに生成される:

```
プロジェクトルート/
├── .wantree/
│   └── 0/
│       └── wantree.yml          # 要件定義
└── .mcts-harness/
    ├── plans/
    │   └── prm-criteria.yml     # PRM 評価軸定義
    ├── tree.yml                 # MCTS ツリー状態
    └── snapshots/               # 各ノードの成果物スナップショット
        ├── <node-id>.md
        ├── <node-id>_devil.md   # 悪魔の代弁者レポート
        └── <node-id>_feedback.md  # PRM 評価フィードバック
```

## スキル一覧

| スキル | コマンド | 役割 |
|--------|---------|------|
| mcts-harness | `/mcts-harness` | 準備フェーズ。wantree〜PRM評価軸設計まで |
| run | `/mcts-harness:run` | 実行フェーズ。MCTSループによる自律的品質改善 |

## 動作要件

- Claude Code（最新版）
- Claude API へのアクセス（Agent ツール使用のため）

## ライセンス

MIT
