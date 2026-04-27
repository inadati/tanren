---
name: run
description: |
  This skill should be used when the user asks to "mcts-harness:runを起動して", "MCTSループを開始して",
  "鍛錬の実行フェーズを開始して",
  or wants to start the mcts-harness execution phase (MCTS quality improvement loop).
  Requires mcts-harness preparation phase to have been completed first.
version: 0.1.0
tools: Read, Write, Edit, Bash, Agent, Glob
---

# mcts-harness:run スキル（実行フェーズ）

MCTS（モンテカルロ木探索）× PRM（プロセス報酬モデル）による自律的品質改善ループ。
本スキルは実行フェーズを担当する。
プランの読み込み → MCTSループ（Selection → Expansion → PRM評価 → Rollout → Backpropagation） → 最良成果物の採用。

---

## フェーズ 0: 初期化

### 0a: プランの読み込み

`Read` で `/Users/ittan/.claude/projects/-Users-ittan-Asweed/memory/MEMORY.md` を読み込む。

`## mcts-harness 実行待ち` セクションから以下を取得する:
- 実装プランファイルのパス
- PRM評価軸ファイルのパス

両ファイルを `Read` で読み込む。

ファイルが見つからない場合:
```
mcts-harness の準備フェーズが完了していません。
先に /mcts-harness を実行してください。
```
と出力してスキルを終了する。

### 0b: MCTSツリーの初期化

`.mcts-harness/` ディレクトリを作成し、ツリーファイルを初期化する:

```bash
mkdir -p .mcts-harness/snapshots
```

`.mcts-harness/tree.yml` を以下の内容で作成する:

```yaml
# mcts-harness MCTS ツリー
iteration: 0
best_score: 0.0
status: running  # running | converged | max_reached

nodes:
  - id: root
    parent_id: null
    description: "初期状態（実装プランに基づく出発点）"
    visits: 0
    total_score: 0.0
    q_value: 0.0
    ucb1: 999.0  # 未探索ノードは最高優先度
    status: unexplored  # unexplored | exploring | explored
    snapshot_path: null
    children: []
```

ユーザーに開始を通知する:

```
MCTSループを開始します。

実装プラン: <プランファイル名>
評価軸: <evaluator_a.role> / <evaluator_b.role> / <evaluator_c.role>
収束閾値: <threshold>
最大反復: <max_iterations>回

---
```

---

## フェーズ 1: MCTSループ

`iteration` が `max_iterations` に達するか、`best_score >= threshold` になるまで繰り返す。

各反復の開始時に以下を表示する:
```
[Iteration N / max_iterations] best_score: X.XX
```

---

### ステップ 1: Selection（選択）

`.mcts-harness/tree.yml` を `Read` で読み込む。

**UCB1 スコアの計算:**

```
UCB1(n) = Q(n) + √2 × √(ln(N_parent) / N(n))

Q(n)      = total_score / visits （平均スコア）
N_parent  = 親ノードの visits
N(n)      = ノード n の visits

未探索ノード（visits=0）は UCB1 = 999.0（最高優先度）
ルートノードは UCB1 = 0.0（選択しない）
```

全リーフノード（`children` が空のノード）の UCB1 を計算し、**最高UCB1のノード**を選択する。

選択したノードを `tree.yml` 内で `status: exploring` に更新する。

---

### ステップ 2: Expansion（展開）

選択ノードから **候補アプローチを3つ生成する**。

実装プランを参照し、選択ノードの現在状態から「次に取りうる実装の方向性」を3つ並列 `Agent` で生成する。

**並列 `Agent` プロンプト（3つ同時起動）:**

```
あなたは成果物の実装担当エージェントです。

実装プラン:
<実装プランの内容>

現在の状態（選択ノード）:
<選択ノードのdescriptionとsnapshot内容（存在する場合）>

あなたのタスク: 次の実装ステップの「候補アプローチ」を1つ提案してください。
- 他のエージェントとは異なる角度・方針で提案すること
- あなたのアプローチ番号: <1 or 2 or 3>

以下の形式で返答してください:
approach_id: candidate-<ノードID>-<1 or 2 or 3>
description: このアプローチの方針（2〜3文）
next_action: 具体的に次に何をするか（1文）
```

3エージェントの結果を子ノードとして `tree.yml` に追加する:

```yaml
- id: candidate-<parent_id>-1
  parent_id: <選択ノードID>
  description: "<エージェントのdescription>"
  visits: 0
  total_score: 0.0
  q_value: 0.0
  ucb1: 999.0
  status: unexplored
  snapshot_path: null
  children: []
```

---

### ステップ 3: PRM 評価（3エージェント並列）

展開された3候補のうち、**UCB1最高の候補**を対象に PRM 評価を行う。

**`prm-criteria.yml` の evaluator_a / evaluator_b / evaluator_c を使い、3エージェントを並列起動する。**

各評価エージェントへのプロンプト:

```
あなたは厳格な評価エージェントです。

あなたの評価観点: 「<evaluator.role>」
着目ポイント: <evaluator.focus>

評価対象のアプローチ:
<対象候補のdescriptionとnext_action>

現在の実装コンテキスト:
<選択ノードのsnapshot内容（存在する場合）>

評価基準（各基準を0.0〜1.0で採点してください）:
<evaluator.criteriaを番号付きリストで列挙>

scoring:
  - 全基準を満たす: 1.0
  - おおむね満たす: 0.7〜0.9
  - 部分的に満たす: 0.4〜0.6
  - ほとんど満たさない: 0.1〜0.3

以下の形式で返答してください:
evaluator_id: <evaluator.id>
scores:
  - criterion: "<基準テキスト>"
    score: 0.0〜1.0
    reason: 判定理由（1行）
overall_score: <全基準の平均>
feedback: |
  評価の総評と改善点（2〜3文）
```

3エージェントの `overall_score` の平均を計算し、対象ノードのスコアとする:

```
node_score = (evaluator_a.overall_score + evaluator_b.overall_score + evaluator_c.overall_score) / 3
```

---

### ステップ 4: Rollout（ロールアウト）

PRM 評価の対象ノードから**成果物を完成まで展開する**。

実装プランを参照し、`Agent` を可能な限り並列で起動して成果物を生成する。

**並列化の方針:**
- 独立して実装できる部分（ファイル・セクション・モジュール等）を特定する
- 互いに依存関係のない部分は `Agent` を並列起動する
- 依存関係のある部分は順次実行する

各 `Agent` には以下を与える:
```
あなたは実装エージェントです。

実装プラン:
<実装プランの内容>

担当箇所: <このエージェントが担当する部分>

現在のコンテキスト:
<選択ノードのsnapshot + 評価フィードバック>

担当箇所を実装してください。
完了したら実装内容の要約を返してください。
```

全エージェントの完了後、成果物のスナップショットを保存する:

```
.mcts-harness/snapshots/<対象ノードID>.md （または適切な拡張子）
```

対象ノードの `snapshot_path` を更新する。

---

### ステップ 5: Backpropagation（バックプロパゲーション）

ロールアウト完了後、リーフノードから根ノードへスコアを伝播する。

対象ノードから根ノードまでのパス上の全ノードを更新する:

```
visits += 1
total_score += node_score
q_value = total_score / visits
```

`tree.yml` を `Edit` で更新する。

`best_score` を更新する:
```
best_score = max(全リーフノードのq_value)
```

---

### ステップ 6: 収束判定

以下の順で判定する:

**収束（品質達成）:**
```
best_score >= threshold
```
→ `tree.yml` の `status: converged` に更新してフェーズ2へ

**上限到達:**
```
iteration >= max_iterations
```
→ `tree.yml` の `status: max_reached` に更新してフェーズ2へ

**継続:**
→ `iteration += 1` にして ステップ1へ戻る

---

## フェーズ 2: 最良成果物の採用

### 2a: 最良パスの特定

`tree.yml` を読み込み、`q_value` が最高のリーフノードを特定する。
そのノードの `snapshot_path` から最終成果物を読み込む。

### 2b: 完了レポート

以下を表示する:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
鍛錬完了
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

状態: <converged | max_reached>
反復回数: <iteration>回
最終スコア: <best_score> / 1.0
採用パス: <最良ノードIDのルートからの経路>

評価者別スコア:
  <evaluator_a.role>: <スコア>
  <evaluator_b.role>: <スコア>
  <evaluator_c.role>: <スコア>

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### 2c: 最終成果物の提示

最良スナップショットの内容を出力する。

### 2d: MEMORY.md のクリーンアップ

`MEMORY.md` から `## mcts-harness 実行待ち` セクションを削除する（`Edit` ツールで）。
