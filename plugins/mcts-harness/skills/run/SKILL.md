---
name: run
description: |
  This skill should be used when the user asks to "mcts-harness:runを起動して", "MCTSループを開始して",
  "鍛錬の実行フェーズを開始して",
  or wants to start the mcts-harness execution phase (MCTS quality improvement loop).
  Requires mcts-harness preparation phase to have been completed first.
version: 0.2.0
tools: Read, Write, Edit, Bash, Agent, Glob
---

# mcts-harness:run スキル（実行フェーズ）

MCTS（モンテカルロ木探索）× PRM（プロセス報酬モデル）による自律的品質改善ループ。
本スキルは実行フェーズを担当する。
プランの読み込み → MCTSループ（Selection → Expansion → Rollout → 悪魔の代弁者検証 → PRM評価 → Backpropagation） → 最良成果物の採用。

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

これまでの評価フィードバック（すべて反映すること）:
<.mcts-harness/snapshots/ 内の *_feedback.md の全文>

あなたのタスク: 次の実装ステップの「候補アプローチ」を1つ提案してください。
- 他のエージェントとは異なる角度・方針で提案すること
- フィードバックで指摘された弱点を克服する方針を含めること
- あなたのアプローチ番号: <1 or 2 or 3>

以下の形式で返答してください:
approach_id: candidate-<ノードID>-<1 or 2 or 3>
description: このアプローチの方針（2〜3文）
next_action: 具体的に次に何をするか（1文）
addresses_feedback: フィードバックのどの指摘にどう対処するか（1〜2文）
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

### ステップ 3: Rollout（ロールアウト）

展開された3候補のうち **UCB1最高の候補** を対象に、**実際に成果物を完成まで実装する**。

評価は実装後に行うため、ここでは「完全な成果物」を生成することだけに集中する。

実装プランを参照し、`Agent` を可能な限り並列で起動して成果物を生成する。

**並列化の方針:**
- 独立して実装できる部分（ファイル・セクション・モジュール等）を特定する
- 互いに依存関係のない部分は `Agent` を並列起動する
- 依存関係のある部分は順次実行する

各 `Agent` には以下を与える:
```
あなたは実装エージェントです。妥協なく完全な実装を行ってください。

実装プラン:
<実装プランの内容>

担当箇所: <このエージェントが担当する部分>

現在のコンテキスト（前回の改善点を必ず反映すること）:
<選択ノードのsnapshot内容（存在する場合）>
<これまでの評価フィードバックの全文>

担当箇所を実装してください。
- TODO・placeholder・省略は一切禁止
- 前回フィードバックで指摘された点はすべて対処すること
完了したら実装内容の要約を返してください。
```

全エージェントの完了後、成果物のスナップショットを保存する:

```
.mcts-harness/snapshots/<対象ノードID>.md （または適切な拡張子）
```

対象ノードの `snapshot_path` を更新する。

---

### ステップ 4: 悪魔の代弁者検証（Devil's Advocate）

**報酬ハッキング検出専門の単独エージェントを起動する。**

このエージェントは「基準を表面的に満たすだけの欺瞞的実装」を探し出すことだけに特化する。
PRM評価エージェントより先に実行し、`penalty_applied` の最終決定権を持つ。

エージェントプロンプト:

```
あなたは報酬ハッキング検出の専門家です。
実装エージェントが評価基準の文言だけを満たして本質的な品質を偽装していないかを検証します。

検証対象（実際に生成された成果物の全文）:
<スナップショットの全文>

PRM評価基準（参照用）:
<prm-criteria.ymlの全文>

以下のパターンを重点的に探してください:
- エラーハンドリングはあるが中身が空（catch {}、pass、TODO等）
- テストはあるが自明すぎる（assert True、assert 1 == 1等）
- コメントはあるが内容と無関係または自動生成的
- 関数・クラスは定義されているが実装されていない（raise NotImplementedError等）
- 要件の文言をそのままコピーしたような実装
- 評価基準の単語が成果物に出現しているが意味のある形で機能していない
- 前回フィードバックの指摘箇所だけ直して他の箇所は手付かず
- スコアを上げるための追記が成果物の一貫性を損なっている

重要: 前回イテレーションのスコアは一切参照しない。成果物の内容だけを見ること。

以下の形式で返答してください:
devil_verdict: penalty / no_penalty
confidence: 0.0〜1.0  # 判定の確信度
findings:
  - pattern: "検出されたハッキングパターンの種別"
    location: "成果物内の該当箇所（引用）"
    reason: "これが表面的充足である根拠"
  （発見なければ空リスト）
summary: |
  判定理由の総括（1〜3文）
recommended_score_cap: 0.3  # penaltyの場合のみ。overall_scoreの上限値を指定する
```

**結果を `.mcts-harness/snapshots/<ノードID>_devil.md` に保存する。**

`devil_verdict: penalty` の場合: 後続のPRM評価のoverall_scoreを `recommended_score_cap` で上書きする。
`devil_verdict: no_penalty` の場合: PRM評価に処理を委ねる。

---

### ステップ 5: PRM 評価（3エージェント並列）

**ステップ3で生成した実際のスナップショット（成果物）を対象に** PRM 評価を行う。

**重要: 評価エージェントには前回イテレーションのスコアを渡さない。成果物とdevilレポートのみ渡す。**

**`prm-criteria.yml` の evaluator_a / evaluator_b / evaluator_c を使い、3エージェントを並列起動する。**

各評価エージェントへのプロンプト:

```
あなたは妥協なき評価エージェントです。甘い評価は品質を損なう。厳格に採点してください。

禁止事項:
- 前回イテレーションのスコアを参照・推測すること
- 「前回より改善されている」という理由でスコアを上げること
- スコアの連続性・一貫性を保とうとすること
成果物の内容だけを独立して評価すること。

悪魔の代弁者レポート（既に検出された疑義）:
<_devil.mdの全文>

あなたの評価観点: 「<evaluator.role>」
着目ポイント: <evaluator.focus>
ペナルティルール: <evaluator.penalty_rule>

評価対象（実際に生成された成果物の全文）:
<スナップショットの全文>

評価基準（各基準を0.0〜1.0で採点してください）:
<evaluator.criteriaを番号付きリストで列挙>

scoring（厳格基準）:
  - 完全・完璧に満たす: 0.95〜1.0
  - 実質的に満たすが軽微な欠陥あり: 0.80〜0.94
  - 部分的に満たすが重大な欠陥あり: 0.50〜0.79
  - ほとんど満たさない: 0.20〜0.49
  - まったく満たさない・逆効果: 0.0〜0.19

「おおむね良い」は 0.85 ではなく 0.80 以下。「完璧」以外は 0.95 を超えない。

表面的充足の検出: 評価基準の文言を形式的に満たしているだけで本質的な品質を伴わない実装は
0.30 以下をつけること（悪魔レポートの指摘がある箇所は特に厳しく判定する）。

自分のペナルティルール適用: 1つでも0.65未満の基準があればoverall_scoreを0.5に上書きする。

以下の形式で返答してください:
evaluator_id: <evaluator.id>
penalty_applied: true / false  # 自分のルールによるペナルティ
scores:
  - criterion: "<基準テキスト>"
    score: 0.0〜1.0
    superficial: true / false  # 表面的充足と判断した場合true
    reason: 具体的な判定理由（「〜という点で不十分」「〜が欠けている」など）
overall_score: <ペナルティ適用後のスコア（ペナルティなしの場合は全基準の平均）>
feedback: |
  評価の総評と具体的な改善指示（「〜を修正せよ」という命令形で3〜5文）
```

**スコアの最終決定:**

```
# 悪魔のペナルティが優先
if devil_verdict == "penalty":
    各evaluatorのoverall_score = min(overall_score, recommended_score_cap)

node_score = (evaluator_a.overall_score + evaluator_b.overall_score + evaluator_c.overall_score) / 3
```

**評価フィードバックは `.mcts-harness/snapshots/<ノードID>_feedback.md` に保存する（次のRolloutで参照するため）。**

---

### ステップ 6: Backpropagation（バックプロパゲーション）

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

### ステップ 7: 収束判定

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

状態: <converged（合格）| max_reached（上限到達・未収束）>
反復回数: <iteration>回
最終スコア: <best_score> / 1.0（合格ライン: 0.90）

評価者別スコア:
  <evaluator_a.role>: <スコア>  <passing_score以上なら PASS / 未満なら FAIL>
  <evaluator_b.role>: <スコア>  <PASS / FAIL>
  <evaluator_c.role>: <スコア>  <PASS / FAIL>

ペナルティ発動回数（PRM）: <全イテレーション合計>
悪魔の代弁者介入回数: <penalty判定を出した回数> / <全イテレーション数>
採用パス: <最良ノードIDのルートからの経路>

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

`max_reached` の場合は以下も表示する:
```
⚠ 上限反復数に達しましたが収束しませんでした（最終スコア: <best_score>）。
  改善が不十分な点: <各評価者のfeedbackから未解決の指摘を要約>
  推奨: PRM評価軸を見直すか、実装プランを修正してから再実行してください。
```

### 2c: 最終成果物の提示

最良スナップショットの内容を出力する。

### 2d: MEMORY.md のクリーンアップ

`MEMORY.md` から `## mcts-harness 実行待ち` セクションを削除する（`Edit` ツールで）。
