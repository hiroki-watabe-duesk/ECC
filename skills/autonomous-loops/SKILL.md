---
name: autonomous-loops
description: "自律的なClaude Codeループのパターンとアーキテクチャ——シンプルなシーケンシャルパイプラインからRFC駆動のマルチエージェントDAGシステムまで。"
origin: ECC
---

# 自律ループスキル

> 互換性に関する注記（v1.8.0）：`autonomous-loops`は1リリース分保持されます。
> 正規スキル名は現在`continuous-agent-loop`です。新しいループガイダンスは
> そちらに執筆し、このスキルは既存ワークフローを壊さないために引き続き利用可能です。

Claude Codeを自律的にループで実行するためのパターン、アーキテクチャ、リファレンス実装。シンプルな`claude -p`パイプラインからRFC駆動のフルマルチエージェントDAGオーケストレーションまでをカバーします。

## 使用場面

- 人間の介入なしに実行する自律的な開発ワークフローのセットアップ
- 問題に適したループアーキテクチャの選択（シンプル vs. 複雑）
- CI/CD方式の継続的開発パイプラインの構築
- マージ調整を伴う並列エージェントの実行
- ループ反復をまたいだコンテキストの永続化
- 自律ワークフローへの品質ゲートとクリーンアップパスの追加

## ループパターンスペクトラム

最もシンプルから最も高度なものまで：

| パターン | 複雑さ | 最適な用途 |
|---------|-----------|----------|
| [シーケンシャルパイプライン](#1-シーケンシャルパイプライン-claude--p) | 低 | 日常的な開発ステップ、スクリプト化されたワークフロー |
| [NanoClaw REPL](#2-nanoclaw-repl) | 低 | インタラクティブな永続セッション |
| [Infinite Agentic Loop](#3-infinite-agentic-loop) | 中 | 並列コンテンツ生成、スペック駆動の作業 |
| [Continuous Claude PRループ](#4-continuous-claude-pr-loop) | 中 | CIゲート付きの複数日にわたる反復プロジェクト |
| [De-Sloppifyパターン](#5-the-de-sloppify-パターン) | アドオン | 各Implementerステップ後の品質クリーンアップ |
| [Ralphinho / RFC駆動DAG](#6-ralphinho--rfc駆動dagオーケストレーション) | 高 | 大規模機能、マージキュー付きのマルチユニット並列作業 |

---

## 1. シーケンシャルパイプライン（`claude -p`）

**最もシンプルなループ。** 日常的な開発を一連の非インタラクティブな`claude -p`呼び出しに分解します。各呼び出しは明確なプロンプトを持つ集中したステップです。

### コアインサイト

> このようなループを理解できないなら、インタラクティブモードでもLLMを使ってコードを修正できないということです。

`claude -p`フラグはClaude Codeをプロンプト付きで非インタラクティブに実行し、完了時に終了します。呼び出しをチェーンしてパイプラインを構築します：

```bash
#!/bin/bash
# daily-dev.sh — Sequential pipeline for a feature branch

set -e

# Step 1: Implement the feature
claude -p "Read the spec in docs/auth-spec.md. Implement OAuth2 login in src/auth/. Write tests first (TDD). Do NOT create any new documentation files."

# Step 2: De-sloppify (cleanup pass)
claude -p "Review all files changed by the previous commit. Remove any unnecessary type tests, overly defensive checks, or testing of language features (e.g., testing that TypeScript generics work). Keep real business logic tests. Run the test suite after cleanup."

# Step 3: Verify
claude -p "Run the full build, lint, type check, and test suite. Fix any failures. Do not add new features."

# Step 4: Commit
claude -p "Create a conventional commit for all staged changes. Use 'feat: add OAuth2 login flow' as the message."
```

### 主要な設計原則

1. **各ステップは分離されている** — `claude -p`呼び出しごとに新鮮なコンテキストウィンドウを使用するため、ステップ間のコンテキスト混入がありません。
2. **順序が重要** — ステップは順次実行されます。各ステップは前のステップが残したファイルシステム状態の上に構築されます。
3. **否定的な指示は危険** — 「型システムをテストするな」とは言わないでください。代わりに別のクリーンアップステップを追加してください（[De-Sloppifyパターン](#5-the-de-sloppify-パターン)参照）。
4. **終了コードは伝播する** — `set -e`は失敗時にパイプラインを停止します。

### バリエーション

**モデルルーティングを使った場合：**
```bash
# Research with Opus (deep reasoning)
claude -p --model opus "Analyze the codebase architecture and write a plan for adding caching..."

# Implement with Sonnet (fast, capable)
claude -p "Implement the caching layer according to the plan in docs/caching-plan.md..."

# Review with Opus (thorough)
claude -p --model opus "Review all changes for security issues, race conditions, and edge cases..."
```

**環境コンテキストを使った場合：**
```bash
# Pass context via files, not prompt length
echo "Focus areas: auth module, API rate limiting" > .claude-context.md
claude -p "Read .claude-context.md for priorities. Work through them in order."
rm .claude-context.md
```

**`--allowedTools`制限を使った場合：**
```bash
# Read-only analysis pass
claude -p --allowedTools "Read,Grep,Glob" "Audit this codebase for security vulnerabilities..."

# Write-only implementation pass
claude -p --allowedTools "Read,Write,Edit,Bash" "Implement the fixes from security-audit.md..."
```

---

## 2. NanoClaw REPL

**ECCのビルトイン永続ループ。** 完全な会話履歴を持ちながら`claude -p`を同期的に呼び出す、セッション対応のREPLです。

```bash
# Start the default session
node scripts/claw.js

# Named session with skill context
CLAW_SESSION=my-project CLAW_SKILLS=tdd-workflow,security-review node scripts/claw.js
```

### 仕組み

1. `~/.claude/claw/{session}.md`から会話履歴を読み込む
2. 各ユーザーメッセージは完全な履歴をコンテキストとして`claude -p`に送信される
3. レスポンスはセッションファイルに追記される（データベースとしてのMarkdown）
4. セッションは再起動をまたいで永続する

### NanoClaw vs. シーケンシャルパイプラインの使い分け

| ユースケース | NanoClaw | シーケンシャルパイプライン |
|----------|----------|-------------------|
| インタラクティブな探索 | はい | いいえ |
| スクリプト化された自動化 | いいえ | はい |
| セッション永続化 | 組み込み | 手動 |
| コンテキスト蓄積 | ターンごとに増加 | ステップごとに新鮮 |
| CI/CD統合 | 不適 | 優秀 |

詳細については`/claw`コマンドのドキュメントを参照してください。

---

## 3. Infinite Agentic Loop

**2プロンプトシステム**で、スペック駆動の生成のために並列サブエージェントをオーケストレートします。dislerによって開発されました（クレジット：@disler）。

### アーキテクチャ：2プロンプトシステム

```
PROMPT 1 (Orchestrator)              PROMPT 2 (Sub-Agents)
┌─────────────────────┐             ┌──────────────────────┐
│ Parse spec file      │             │ Receive full context  │
│ Scan output dir      │  deploys   │ Read assigned number  │
│ Plan iteration       │────────────│ Follow spec exactly   │
│ Assign creative dirs │  N agents  │ Generate unique output │
│ Manage waves         │             │ Save to output dir    │
└─────────────────────┘             └──────────────────────┘
```

### パターン

1. **スペック分析** — オーケストレーターが何を生成するかを定義した仕様ファイル（Markdown）を読む
2. **ディレクトリ偵察** — 既存の出力をスキャンして最高の反復番号を見つける
3. **並列デプロイ** — N個のサブエージェントを起動し、それぞれが以下を受け取る：
   - 完全なスペック
   - ユニークなクリエイティブな方向性
   - 特定の反復番号（競合なし）
   - 既存反復のスナップショット（ユニーク性のため）
4. **ウェーブ管理** — 無限モードの場合、コンテキストが枯渇するまで3〜5エージェントのウェーブを展開

### Claude Codeコマンドによる実装

`.claude/commands/infinite.md`を作成：

```markdown
Parse the following arguments from $ARGUMENTS:
1. spec_file — path to the specification markdown
2. output_dir — where iterations are saved
3. count — integer 1-N or "infinite"

PHASE 1: Read and deeply understand the specification.
PHASE 2: List output_dir, find highest iteration number. Start at N+1.
PHASE 3: Plan creative directions — each agent gets a DIFFERENT theme/approach.
PHASE 4: Deploy sub-agents in parallel (Task tool). Each receives:
  - Full spec text
  - Current directory snapshot
  - Their assigned iteration number
  - Their unique creative direction
PHASE 5 (infinite mode): Loop in waves of 3-5 until context is low.
```

**実行：**
```bash
/project:infinite specs/component-spec.md src/ 5
/project:infinite specs/component-spec.md src/ infinite
```

### バッチ戦略

| カウント | 戦略 |
|-------|----------|
| 1-5 | 全エージェントを同時に |
| 6-20 | 5エージェントのバッチで |
| infinite | 3〜5のウェーブで、段階的に洗練させる |

### 主要インサイト：割り当てによるユニーク性

エージェントの自己差別化に頼らないでください。オーケストレーターが各エージェントに特定のクリエイティブな方向性と反復番号を**割り当て**ます。これにより並列エージェント間での概念の重複を防ぎます。

---

## 4. Continuous Claude PRループ

**本番グレードのシェルスクリプト**で、Claude Codeを継続的なループで実行し、PRを作成し、CIを待ち、自動的にマージします。AnandChowdharyによって作成されました（クレジット：@AnandChowdhary）。

### コアループ

```
┌─────────────────────────────────────────────────────┐
│  CONTINUOUS CLAUDE ITERATION                        │
│                                                     │
│  1. Create branch (continuous-claude/iteration-N)   │
│  2. Run claude -p with enhanced prompt              │
│  3. (Optional) Reviewer pass — separate claude -p   │
│  4. Commit changes (claude generates message)       │
│  5. Push + create PR (gh pr create)                 │
│  6. Wait for CI checks (poll gh pr checks)          │
│  7. CI failure? → Auto-fix pass (claude -p)         │
│  8. Merge PR (squash/merge/rebase)                  │
│  9. Return to main → repeat                         │
│                                                     │
│  Limit by: --max-runs N | --max-cost $X             │
│            --max-duration 2h | completion signal     │
└─────────────────────────────────────────────────────┘
```

### インストール

> **警告：** コードをレビューした後にcontinuous-claudeをそのリポジトリからインストールしてください。外部スクリプトをbashに直接パイプしないでください。

### 使用方法

```bash
# Basic: 10 iterations
continuous-claude --prompt "Add unit tests for all untested functions" --max-runs 10

# Cost-limited
continuous-claude --prompt "Fix all linter errors" --max-cost 5.00

# Time-boxed
continuous-claude --prompt "Improve test coverage" --max-duration 8h

# With code review pass
continuous-claude \
  --prompt "Add authentication feature" \
  --max-runs 10 \
  --review-prompt "Run npm test && npm run lint, fix any failures"

# Parallel via worktrees
continuous-claude --prompt "Add tests" --max-runs 5 --worktree tests-worker &
continuous-claude --prompt "Refactor code" --max-runs 5 --worktree refactor-worker &
wait
```

### 反復間のコンテキスト：SHARED_TASK_NOTES.md

重要なイノベーション：`SHARED_TASK_NOTES.md`ファイルが反復をまたいで永続します：

```markdown
## Progress
- [x] Added tests for auth module (iteration 1)
- [x] Fixed edge case in token refresh (iteration 2)
- [ ] Still need: rate limiting tests, error boundary tests

## Next Steps
- Focus on rate limiting module next
- The mock setup in tests/helpers.ts can be reused
```

Claudeは反復開始時にこのファイルを読み、反復終了時に更新します。これにより独立した`claude -p`呼び出し間のコンテキストギャップを橋渡しします。

### CI失敗の回復

PRチェックが失敗すると、Continuous Claudeは自動的に：
1. `gh run list`経由で失敗した実行IDを取得
2. CIの修正コンテキストとともに新しい`claude -p`を生成
3. Claudeが`gh run view`経由でログを検査し、コードを修正し、コミットし、プッシュ
4. チェックを再待機（`--ci-retry-max`の試行回数まで）

### 完了シグナル

Claudeは「完了した」という魔法のフレーズを出力することでシグナルを送れます：

```bash
continuous-claude \
  --prompt "Fix all bugs in the issue tracker" \
  --completion-signal "CONTINUOUS_CLAUDE_PROJECT_COMPLETE" \
  --completion-threshold 3  # Stops after 3 consecutive signals
```

3回連続して完了をシグナルすると、ループが停止し、完了した作業への無駄な実行を防ぎます。

### 主要設定

| フラグ | 目的 |
|------|---------|
| `--max-runs N` | N回の成功した反復後に停止 |
| `--max-cost $X` | $X使用後に停止 |
| `--max-duration 2h` | 経過時間後に停止 |
| `--merge-strategy squash` | squash、merge、またはrebase |
| `--worktree <name>` | gitワークツリー経由の並列実行 |
| `--disable-commits` | ドライランモード（git操作なし） |
| `--review-prompt "..."` | 反復ごとにレビュアーパスを追加 |
| `--ci-retry-max N` | CI失敗の自動修正（デフォルト：1） |

---

## 5. The De-Sloppify パターン

**任意のループへのアドオンパターン。** 各Implementerステップの後に専用のクリーンアップ/リファクタステップを追加します。

### 問題点

LLMにTDDで実装するよう依頼すると、「テストを書く」を文字通りに受け取ります：
- TypeScriptの型システムが機能することを検証するテスト（`typeof x === 'string'`のテスト）
- 型システムがすでに保証していることへの過度に防御的なランタイムチェック
- ビジネスロジックではなくフレームワークの動作のためのテスト
- 実際のコードを曖昧にする過剰なエラーハンドリング

### 否定的な指示がなぜ機能しないのか

「型システムをテストするな」または「不必要なチェックを追加するな」をImplementerプロンプトに追加すると、副作用があります：
- モデルがすべてのテストについて消極的になる
- 正当なエッジケーステストをスキップする
- 品質が予測不可能な方法で低下する

### 解決策：別のパス

Implementerを制約する代わりに、徹底的に実装させてください。そして集中したクリーンアップエージェントを追加します：

```bash
# Step 1: Implement (let it be thorough)
claude -p "Implement the feature with full TDD. Be thorough with tests."

# Step 2: De-sloppify (separate context, focused cleanup)
claude -p "Review all changes in the working tree. Remove:
- Tests that verify language/framework behavior rather than business logic
- Redundant type checks that the type system already enforces
- Over-defensive error handling for impossible states
- Console.log statements
- Commented-out code

Keep all business logic tests. Run the test suite after cleanup to ensure nothing breaks."
```

### ループコンテキストでの使用

```bash
for feature in "${features[@]}"; do
  # Implement
  claude -p "Implement $feature with TDD."

  # De-sloppify
  claude -p "Cleanup pass: review changes, remove test/code slop, run tests."

  # Verify
  claude -p "Run build + lint + tests. Fix any failures."

  # Commit
  claude -p "Commit with message: feat: add $feature"
done
```

### 主要インサイト

> 否定的な指示を追加するとダウンストリームの品質効果があるため、代わりにde-sloppifyパスを追加してください。焦点を絞った2つのエージェントは、制約された1つのエージェントより優れています。

---

## 6. Ralphinho / RFC駆動DAGオーケストレーション

**最も高度なパターン。** RFC駆動のマルチエージェントパイプラインで、スペックを依存関係DAGに分解し、各ユニットを段階的な品質パイプラインに通し、エージェント駆動のマージキュー経由でランディングします。enitratによって作成されました（クレジット：@enitrat）。

### アーキテクチャ概要

```
RFC/PRD Document
       │
       ▼
  DECOMPOSITION (AI)
  Break RFC into work units with dependency DAG
       │
       ▼
┌──────────────────────────────────────────────────────┐
│  RALPH LOOP (up to 3 passes)                         │
│                                                      │
│  For each DAG layer (sequential, by dependency):     │
│                                                      │
│  ┌── Quality Pipelines (parallel per unit) ───────┐  │
│  │  Each unit in its own worktree:                │  │
│  │  Research → Plan → Implement → Test → Review   │  │
│  │  (depth varies by complexity tier)             │  │
│  └────────────────────────────────────────────────┘  │
│                                                      │
│  ┌── Merge Queue ─────────────────────────────────┐  │
│  │  Rebase onto main → Run tests → Land or evict │  │
│  │  Evicted units re-enter with conflict context  │  │
│  └────────────────────────────────────────────────┘  │
│                                                      │
└──────────────────────────────────────────────────────┘
```

### RFC分解

AIがRFCを読んで作業ユニットを生成します：

```typescript
interface WorkUnit {
  id: string;              // kebab-case identifier
  name: string;            // Human-readable name
  rfcSections: string[];   // Which RFC sections this addresses
  description: string;     // Detailed description
  deps: string[];          // Dependencies (other unit IDs)
  acceptance: string[];    // Concrete acceptance criteria
  tier: "trivial" | "small" | "medium" | "large";
}
```

**分解ルール：**
- より少なく、凝集性の高いユニットを優先する（マージリスクを最小化）
- ユニット間のファイル重複を最小化する（競合を回避）
- テストを実装と一緒に保持する（「Xを実装する」+「Xをテストする」を分離しない）
- 実際のコード依存関係が存在する場合のみ依存関係を設定

依存関係DAGが実行順序を決定します：
```
Layer 0: [unit-a, unit-b]     ← no deps, run in parallel
Layer 1: [unit-c]             ← depends on unit-a
Layer 2: [unit-d, unit-e]     ← depend on unit-c
```

### 複雑さのティア

異なるティアは異なるパイプラインの深さを得ます：

| ティア | パイプラインステージ |
|------|----------------|
| **trivial** | implement → test |
| **small** | implement → test → code-review |
| **medium** | research → plan → implement → test → PRD-review + code-review → review-fix |
| **large** | research → plan → implement → test → PRD-review + code-review → review-fix → final-review |

これにより、シンプルな変更に対する高コストな操作を防ぎつつ、アーキテクチャ的な変更が徹底的な精査を受けることを保証します。

### 別個のコンテキストウィンドウ（著者バイアスの排除）

各ステージは独自のエージェントプロセスと独自のコンテキストウィンドウで実行されます：

| ステージ | モデル | 目的 |
|-------|-------|---------|
| Research | Sonnet | コードベース+RFCを読み、コンテキストドキュメントを生成 |
| Plan | Opus | 実装ステップを設計 |
| Implement | Codex | 計画に従ってコードを書く |
| Test | Sonnet | ビルド+テストスイートを実行 |
| PRD Review | Sonnet | スペック適合チェック |
| Code Review | Opus | 品質+セキュリティチェック |
| Review Fix | Codex | レビュー問題に対処 |
| Final Review | Opus | 品質ゲート（largeティアのみ） |

**重要な設計：** レビュアーは自分がレビューするコードを書きません。これにより著者バイアスを排除します。著者バイアスはセルフレビューで問題が見逃される最も一般的な原因です。

### エビクション付きマージキュー

品質パイプラインの完了後、ユニットはマージキューに入ります：

```
Unit branch
    │
    ├─ Rebase onto main
    │   └─ Conflict? → EVICT (capture conflict context)
    │
    ├─ Run build + tests
    │   └─ Fail? → EVICT (capture test output)
    │
    └─ Pass → Fast-forward main, push, delete branch
```

**ファイル重複のインテリジェンス：**
- 非重複ユニットは投機的に並列でランディング
- 重複ユニットは毎回リベースしながら1つずつランディング

**エビクション回復：**
エビクション時は完全なコンテキスト（競合ファイル、差分、テスト出力）がキャプチャされ、次のRalphパスで実装者にフィードバックされます：

```markdown
## MERGE CONFLICT — RESOLVE BEFORE NEXT LANDING

Your previous implementation conflicted with another unit that landed first.
Restructure your changes to avoid the conflicting files/lines below.

{full eviction context with diffs}
```

### ステージ間のデータフロー

```
research.contextFilePath ──────────────────→ plan
plan.implementationSteps ──────────────────→ implement
implement.{filesCreated, whatWasDone} ─────→ test, reviews
test.failingSummary ───────────────────────→ reviews, implement (next pass)
reviews.{feedback, issues} ────────────────→ review-fix → implement (next pass)
final-review.reasoning ────────────────────→ implement (next pass)
evictionContext ───────────────────────────→ implement (after merge conflict)
```

### ワークツリーの分離

すべてのユニットは分離されたワークツリーで実行されます（gitではなくjj/Jujutsuを使用）：
```
/tmp/workflow-wt-{unit-id}/
```

同じユニットのパイプラインステージは**同一の**ワークツリーを**共有**し、research → plan → implement → test → reviewをまたいで状態（コンテキストファイル、計画ファイル、コード変更）を保持します。

### 主要な設計原則

1. **決定論的な実行** — 事前の分解が並列性と順序をロックイン
2. **レバレッジポイントでの人間レビュー** — 作業計画が最も影響力の高い介入ポイント
3. **関心の分離** — 各ステージは別個のコンテキストウィンドウの別個のエージェント
4. **コンテキスト付きの競合回復** — 完全なエビクションコンテキストがブラインドリトライではなくインテリジェントな再実行を可能にする
5. **ティア駆動の深さ** — 些細な変更はresearch/reviewをスキップ、大きな変更は最大の精査を得る
6. **再開可能なワークフロー** — 完全な状態をSQLiteに保持、任意のポイントから再開可能

### RalphinhoをシンプルなパターンにいつRalphinhoを使うべきか

| シグナル | Ralphinho | シンプルなパターン |
|--------|--------------|-------------------|
| 複数の相互依存する作業ユニット | はい | いいえ |
| 並列実装が必要 | はい | いいえ |
| マージ競合が発生しやすい | はい | いいえ（シーケンシャルで十分） |
| 単一ファイルの変更 | いいえ | はい（シーケンシャルパイプライン） |
| 複数日のプロジェクト | はい | たぶん（continuous-claude） |
| スペック/RFCがすでに書かれている | はい | たぶん |
| 一つのことへの素早い反復 | いいえ | はい（NanoClaw またはパイプライン） |

---

## 適切なパターンの選択

### 意思決定マトリックス

```
Is the task a single focused change?
├─ Yes → Sequential Pipeline or NanoClaw
└─ No → Is there a written spec/RFC?
         ├─ Yes → Do you need parallel implementation?
         │        ├─ Yes → Ralphinho (DAG orchestration)
         │        └─ No → Continuous Claude (iterative PR loop)
         └─ No → Do you need many variations of the same thing?
                  ├─ Yes → Infinite Agentic Loop (spec-driven generation)
                  └─ No → Sequential Pipeline with de-sloppify
```

### パターンの組み合わせ

これらのパターンはうまく組み合わさります：

1. **シーケンシャルパイプライン + De-Sloppify** — 最も一般的な組み合わせ。すべての実装ステップがクリーンアップパスを得ます。

2. **Continuous Claude + De-Sloppify** — 各反復に`--review-prompt`でde-sloppifyディレクティブを追加します。

3. **任意のループ + 検証** — コミット前のゲートとしてECCの`/verify`コマンドまたは`verification-loop`スキルを使用します。

4. **シンプルなループでRalphinhoのティア型アプローチ** — シーケンシャルパイプラインでも、シンプルなタスクをHaikuにルーティングし、複雑なタスクをOpusにルーティングできます：
   ```bash
   # Simple formatting fix
   claude -p --model haiku "Fix the import ordering in src/utils.ts"

   # Complex architectural change
   claude -p --model opus "Refactor the auth module to use the strategy pattern"
   ```

---

## アンチパターン

### よくある間違い

1. **終了条件のない無限ループ** — 常にmax-runs、max-cost、max-duration、または完了シグナルを持ってください。

2. **反復間のコンテキストブリッジなし** — 各`claude -p`呼び出しは新鮮に始まります。コンテキストを橋渡しするために`SHARED_TASK_NOTES.md`またはファイルシステム状態を使用してください。

3. **同じ失敗のリトライ** — 反復が失敗した場合は、ただリトライしないでください。エラーコンテキストをキャプチャして次の試みにフィードバックしてください。

4. **クリーンアップパスの代わりに否定的な指示** — 「Xするな」と言わないでください。Xを削除する別のパスを追加してください。

5. **1つのコンテキストウィンドウ内の全エージェント** — 複雑なワークフローでは、関心を別のエージェントプロセスに分離してください。レビュアーは著者であるべきではありません。

6. **並列作業でのファイル重複を無視** — 2つの並列エージェントが同じファイルを編集する可能性がある場合、マージ戦略（シーケンシャルランディング、リベース、または競合解決）が必要です。

---

## 参考

| プロジェクト | 著者 | リンク |
|---------|--------|------|
| Ralphinho | enitrat | クレジット：@enitrat |
| Infinite Agentic Loop | disler | クレジット：@disler |
| Continuous Claude | AnandChowdhary | クレジット：@AnandChowdhary |
| NanoClaw | ECC | このリポジトリの`/claw`コマンド |
| Verification Loop | ECC | このリポジトリの`skills/verification-loop/` |
