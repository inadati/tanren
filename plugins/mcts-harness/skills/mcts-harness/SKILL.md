---
name: mcts-harness
description: |
  This skill should be used when the user asks to "mcts-harnessを起動して", "鍛錬を始めて",
  "mcts-harnessで実装したい", "品質を鍛え上げて",
  or wants to design PRM evaluation criteria and hand off to mcts-harness:run.
  Autonomously collects context (existing plans, wantree) and starts from PRM criteria design.
version: 0.4.0
tools: Read, Write, Edit, Bash, Agent, EnterPlanMode, ExitPlanMode, Glob
---

# mcts-harness スキル（準備フェーズ）

MCTS（モンテカルロ木探索）× PRM（プロセス報酬モデル）駆動の品質鍛錬ハーネス。
本スキルは準備フェーズを担当する。
コンテキスト収集 → PRM 評価軸設計 → mcts-harness:run への引き渡し。

---

## フェーズ 0: コンテキスト収集

スキル起動直後に以下を自律的に実行する（ユーザーへの質問なし）。

### 0a: 既存プランファイルの探索

`Glob` で `~/.claude/plans/` 内のファイルを探し、最新のものを `Read` で読み込む。
見つからない場合は会話の文脈から実装プランを把握する。

### 0b: wantree.yml の探索

`Glob` で `.wantree/` 配下の最大番号ディレクトリ内の `wantree.yml` を探す。
見つかれば `Read` で読み込む。

### 0c: コンテキストサマリーの表示

収集した情報を整理し、以下をユーザーに表示する:

```
コンテキストを収集しました。

実装プラン: <ファイル名 または「会話の文脈から把握」>
wantree: <ファイルパス または「なし」>
成果物の種類: <コード / ブログ記事 / その他（把握できた場合）>

PRM評価軸の設計に入ります。
```

---

## フェーズ 1: PRM 評価軸設計（プランモード）

`EnterPlanMode` に入る。フェーズ0で収集したコンテキストを踏まえて評価軸を設計する。

評価者の数は固定しない。「この成果物を網羅的に評価するために何軸必要か」だけを基準に決める。

### 設計方針

- 収集したコンテキストから、この成果物に適した評価観点を導出する
- 各観点が互いに**重複せず・網羅的に**なるよう設計する（MECE）
- 各評価基準は **0.0〜1.0 のスコアで採点できる粒度** で書く
- 基準は **最低10項目以上**・極限まで厳しく・二値判定可能な粒度で列挙する
- 「できている」「わかりやすい」など曖昧な基準は禁止。必ず具体的・測定可能な形にする
- 合格スコアのデフォルトは **0.92**（競合する観点同士でも 0.90 以下には下げない）
- 1つでも0.65未満の基準があればその評価者のoverall_scoreは0.5上限とする（ペナルティルール）

### カバレッジチェック（必須）

評価軸の提案後、ExitPlanModeの前に以下を自問すること:

- 重要な品質次元が漏れていないか？
- コード開発であれば: セキュリティ・堅牢性、パフォーマンス、テスタビリティ、アクセシビリティ等
- ブログ記事であれば: 正確性・深さ、読者価値・CTA、SEO、独自性等
- 漏れている次元があれば軸を追加するか、既存軸のcriteriaに組み込む

最終的に「なぜこの軸数でこの成果物を網羅できるか」を説明できる状態にしてからExitPlanModeに進む。

### PRM 評価軸プランの形式

```yaml
prm_evaluators:
  - id: evaluator_1
    role: "評価観点の名前（例: コード品質）"
    focus: "何に着目して評価するか（1〜2文）"
    penalty_rule: "1つでも0.65未満の基準があればoverall_scoreは0.5を上限とする"
    criteria:
      - "採点基準1（具体的・二値判定可能な粒度で）"
      - "採点基準2"
      - （最低10項目以上。曖昧な基準は禁止。測定・検証可能な形で書く）
    passing_score: 0.92

  - id: evaluator_2
    role: "評価観点の名前"
    focus: "何に着目して評価するか"
    penalty_rule: "1つでも0.65未満の基準があればoverall_scoreは0.5を上限とする"
    criteria:
      - "採点基準1"
      - （最低10項目以上。曖昧な基準は禁止）
    passing_score: 0.92

  # 網羅的なカバレッジに必要な数だけ追加する（奇数・偶数の制約なし、上限なし）

convergence:
  threshold: 0.90     # 全評価者の平均がこれを超えたら収束
  max_iterations: 15  # MCTSループの上限回数
```

`ExitPlanMode` で評価軸プランを提出する。

---

## フェーズ 2: plan-review

`ExitPlanMode` 後に `plan-review` が AUTO-TRIGGER される（設計通り）。
各評価者の `criteria` と `passing_score` も確認される。
ユーザーが全項目を承認したら評価軸プランが確定。承認後にフェーズ3へ進む。

---

## フェーズ 3: プランの保存と引き渡し

### 3a: プランファイルの保存

確定した評価軸プランを `.mcts-harness/plans/prm-criteria.yml` に書き出す。

```bash
mkdir -p .mcts-harness/plans
```

フェーズ0で読み込んだ実装プランファイルのパスも記録しておく。

### 3b: MEMORY.md への記録

`/Users/ittan/.claude/projects/-Users-ittan-Asweed/memory/MEMORY.md` に以下を追記する（Editツールで）:

```markdown
## mcts-harness 実行待ち

- **実装プラン**: `<フェーズ0で特定したプランファイルパス 또は「会話の文脈から把握（概要: ...）」>`
- **PRM評価軸**: `<カレントディレクトリ>/.mcts-harness/plans/prm-criteria.yml`
- **次のアクション**: `mcts-harness:run` を起動して MCTSループを開始する
```

### 3c: ユーザーへの案内

以下を出力してスキルを終了する:

```
準備フェーズが完了しました。

PRM 評価軸を保存しました。
コンテキストをリセットしてから、以下のコマンドで実行フェーズを開始してください:

  /mcts-harness:run

mcts-harness:run は MEMORY.md からプランを自動で読み込み、
MCTS ループによる自律的な品質改善を開始します。
```
