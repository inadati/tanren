---
name: mcts-harness
description: |
  This skill should be used when the user asks to "mcts-harnessを起動して", "鍛錬を始めて",
  "mcts-harnessで実装したい", "品質を鍛え上げて",
  or wants to start the mcts-harness preparation phase (wantree through PRM criteria review).
  Covers all steps up to handing off to mcts-harness:run.
version: 0.2.0
tools: Read, Write, Edit, Bash, Agent, AskUserQuestion, EnterPlanMode, ExitPlanMode, Glob
---

# mcts-harness スキル（準備フェーズ）

MCTS（モンテカルロ木探索）× PRM（プロセス報酬モデル）駆動の品質鍛錬ハーネス。
本スキルは準備フェーズを担当する。
wantree による要件定義 → 成果物プランニング → PRM 評価軸設計 → mcts-harness:run への引き渡し。

---

## フェーズ 0: 起動確認

`AskUserQuestion` で現在地を確認する:

```
どこから始めますか？

A. 要件定義から始める（wantreeで要件を整理したい）
B. 既存の実装プランがある（フェーズ2から開始）
```

`.mcts-harness/plans/` が存在する場合、`Glob` で確認してユーザーに伝える:

```
既存の mcts-harness プランが見つかりました。
新しいプランを作成します（既存は上書きしません）。
```

---

## フェーズ 1: wantree（要件定義）

**フェーズ0でA（要件定義から始める）を選択した場合のみ実行。**

wantree スキルの手順をインラインで実行する。

### wantree.yml フォーマット

```yaml
want: ここに目的・要望を書く   # ← 必ずテキスト（配列にしない）
tree:
  - want: サブ要件1
  - want: サブ要件2
    tree:
      - want: さらに詳細
```

**重要**: `want` フィールドは常に文字列。複数の要件は `tree:` リストの各要素の `want` として表現する。

### 1a: 初期化

カレントディレクトリで `.wantree/` の状態を確認する:

- **`.wantree/` が存在しない** → 初期化してヒアリング開始
- **`.wantree/` が存在する** → 一覧を表示してユーザーに選択させる
  - 新しいwantreeを作成 → ヒアリング開始
  - 既存のwantreeを編集 → 該当ファイルを開いてヒアリング再開

初期化時は以下の構造を作成する:

```
.wantree/
└── 0/
    └── wantree.yml   ← want:\ntree:\n の初期内容
```

### 1b: ヒアリング

ユーザーに伝える:

```
まず、はじめのwantを教えてください。
```

ユーザーがwantを伝えるたびに以下を繰り返す:

1. **入力の判定**:
   - `x` 一文字 → ヒアリング終了
   - 「取り消し」「削除」「消して」 → 直前のwantを削除してヒアリング再開
   - それ以外 → wantとして追記

2. **wantの追記**:
   - 音声入力の表記ミスや誤字脱字を自動修正する（意味は変えない）
   - 親子関係が明確な場合は `tree:` で階層構造を表現
   - `Edit` ツールで `wantree.yml` を更新する

3. **返答フォーマット**（毎回このフォーマットのみ）:

```
（更新後のwantree.ymlのコードブロック）

次のwantをどうぞ。
---
終了: x
```

**絶対禁止**: コードブロック内の `wantree.yml` を `...` や `# 省略` などで省略してはならない。**常に全文を表示する。**

### 1c: wantree.yml の特定

ヒアリング完了後、`Glob` で `.wantree/` 配下の最大番号ディレクトリ内の `wantree.yml` を特定し、
パスをメモしてフェーズ2へ進む。

---

## フェーズ 2: 成果物プランニング（プランモード）

`EnterPlanMode` を使い成果物の実装プランを策定する。

プランモード内で以下を参照する:
- wantree.yml（フェーズ1を経た場合）
- 既存のコード・ファイル構成（フェーズ0でBを選択した場合）

**成果物の種類を特定する（コード・ブログ記事・スライド等）。**
種類に応じて以下を含むプランを作成する:

| 項目 | 内容 |
|------|------|
| 成果物の種類 | コード / ブログ記事 / スライド / その他 |
| 実装方針 | なぜその方針か、技術選定の理由 |
| 構成 | ファイル構成 or 記事構成 or スライド構成 |
| 実装ステップ | 番号付き手順（MCTSの各ノードで進める単位） |
| リスク・注意点 | 考慮すべき落とし穴 |

`ExitPlanMode` でプランを提出する。

---

## フェーズ 3: plan-review（1回目）

`ExitPlanMode` 後に `plan-review` が AUTO-TRIGGER される（設計通り）。
ユーザーが全項目を承認したら実装プランが確定。承認後にフェーズ4へ進む。

---

## フェーズ 4: PRM 評価軸設計（プランモード）

確定した実装プランを踏まえ、再び `EnterPlanMode` に入る。

プランモード内で以下を読む:
1. 確定した実装プランファイル（`~/.claude/plans/` 内の最新プラン）
2. wantree.yml（存在する場合）

**評価者は3名固定。** 各評価者の役割と採点基準を設計する。

### 設計方針

- 実装プランの内容から、この成果物に適した評価観点を導出する
- 3つの観点が互いに**重複せず・網羅的に**なるよう設計する（MECE）
- 各評価基準は **0.0〜1.0 のスコアで採点できる粒度** で書く
- 基準は思いつく限り多く・厳しく・具体的に列挙する
- 合格スコアのデフォルトは **0.85**（競合する観点同士のみ引き下げを検討）

### 成果物別の評価軸の例

**コード開発の場合:**
- 評価者A: 機能要件充足・正確性
- 評価者B: コード品質・保守性
- 評価者C: セキュリティ・堅牢性

**ブログ記事の場合:**
- 評価者A: 読みやすさ・構成・文章品質
- 評価者B: 内容の正確性・深さ・独自性
- 評価者C: 読者価値・CTA・目的達成度

### PRM 評価軸プランの形式

```yaml
prm_evaluators:
  - id: evaluator_a
    role: "評価観点の名前（例: コード品質）"
    focus: "何に着目して評価するか（1〜2文）"
    criteria:
      - "採点基準1（具体的・二値判定可能な粒度で）"
      - "採点基準2"
      - "採点基準3"
      - （思いつく限り列挙する）
    passing_score: 0.85

  - id: evaluator_b
    role: "評価観点の名前"
    focus: "何に着目して評価するか"
    criteria:
      - "採点基準1"
      - （思いつく限り列挙する）
    passing_score: 0.85

  - id: evaluator_c
    role: "評価観点の名前"
    focus: "何に着目して評価するか"
    criteria:
      - "採点基準1"
      - （思いつく限り列挙する）
    passing_score: 0.85

convergence:
  threshold: 0.85     # 全評価者の平均がこれを超えたら収束
  max_iterations: 10  # MCTSループの上限回数
```

`ExitPlanMode` で評価軸プランを提出する。

---

## フェーズ 5: plan-review（2回目）

評価軸プランに対しても `plan-review` が AUTO-TRIGGER される。
各評価者の `criteria` と `passing_score` も確認される。
ユーザーが全項目を承認したら評価軸プランが確定。承認後にフェーズ6へ進む。

---

## フェーズ 6: プランの保存と引き渡し

### 6a: プランファイルの保存

確定した評価軸プランを `.mcts-harness/plans/prm-criteria.yml` に書き出す。

```bash
mkdir -p .mcts-harness/plans
```

`~/.claude/plans/` の最新プランファイルのパスも記録しておく。

### 6b: MEMORY.md への記録

`/Users/ittan/.claude/projects/-Users-ittan-Asweed/memory/MEMORY.md` に以下を追記する（Editツールで）:

```markdown
## mcts-harness 実行待ち

- **実装プラン**: `~/.claude/plans/<プランファイル名>`
- **PRM評価軸**: `<カレントディレクトリ>/.mcts-harness/plans/prm-criteria.yml`
- **次のアクション**: `mcts-harness:run` を起動して MCTSループを開始する
```

### 6c: ユーザーへの案内

以下を出力してスキルを終了する:

```
準備フェーズが完了しました。

実装プランと PRM 評価軸を保存しました。
コンテキストをリセットしてから、以下のコマンドで実行フェーズを開始してください:

  /mcts-harness:run

mcts-harness:run は MEMORY.md からプランを自動で読み込み、
MCTS ループによる自律的な品質改善を開始します。
```
