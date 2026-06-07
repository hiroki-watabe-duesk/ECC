# 第08章 セッション・コンテキスト・トークン経済

## この章で学ぶこと

- セッション永続化の仕組み（保存先・ファイル名パターン・フォーマット）
- SessionStart / Stop フックによるセッション間の文脈継承
- contexts/ ディレクトリのモード切り替えパターン
- トークン経済：モデル選択・`MAX_THINKING_TOKENS` 調整・MCP 置換
- コンテキストのライフサイクルと `/compact` の戦略的活用
- pass@k vs pass^k 検証ループの使い分け
- サブエージェント・オーケストレーションのフェーズ駆動パターン
- 並列化：最小限の並列化で最大成果を得る方法
- 長時間・多日タスクをセッション分割して実行する実践手順

---

## Claude Code の文脈・セッション・トークンの基礎（初心者向け）

> この節は Claude Code のセッション・コンテキスト・トークンの概念に初めて触れる方向けの基礎解説です。Claude Code 自体の入門（ハーネス・エージェントループ・設定の仕組み）については [03章](./03-components-overview.md) を先にご覧ください。

### コンテキストウィンドウとは何か

Claude（LLM）は会話を「一定の長さのテキスト」として扱います。この「一度に見ることができるテキストの上限」を **コンテキストウィンドウ**（文脈窓）と呼びます。

```
┌─────────────────────────────────────────────────────┐
│ コンテキストウィンドウ（有限）                          │
│                                                     │
│  ・これまでのやり取り（会話履歴）                       │
│  ・ツールの実行結果（ファイル内容・Bash出力 等）          │
│  ・システムプロンプト・スキル・ルール                    │
│  ・MCPツール定義                                     │
│                                     ↑ すべてここに乗る │
└─────────────────────────────────────────────────────┘
```

コンテキストウィンドウは **有限** です。会話が長くなったり、大量のファイルを読んだり、多くのツールを有効化したりすると、窓が埋まっていきます。窓が満杯に近づくと Claude は古い情報を「見えない」状態になり、指示を忘れたり一貫性が下がったりすることがあります。

### なぜ「有限」が問題になるのか

トークン（LLMがテキストを処理する最小単位）の消費は**コストと品質の両方**に影響します。

- **コスト面**: トークンを多く消費するほど API 料金が上がります。不要に大きなファイルを読み込む・拡張思考（extended thinking）を常時オンにする・MCP ツール定義を大量に読み込むといった操作は、知らないうちにコストを押し上げます。
- **品質面**: 窓が埋まるほど、モデルは「いま本当に重要な情報」を取り出しにくくなります。長い会話の後半で指示ブレが起きるのは、多くの場合これが原因です。

### 圧縮（Compaction）と要約

コンテキストが一定量を超えると、Claude Code は自動的に会話を **圧縮（compaction）** します。これは「これまでの会話を要約した短いテキスト」に置き換える操作です。圧縮によって窓の空きが増える一方、細かい詳細は失われます。

手動で圧縮するコマンドが `/compact` です。圧縮のタイミングは重要で、誤った場面で実行すると作業中の文脈が失われます。タイミングのルールは後述の「[戦略的 /compact の使いどころ](#戦略的-compact-の使いどころ)」で詳しく説明します。

### セッションとは何か／継続・再開の仕組み

**セッション**とは、Claude Code との一連の会話のまとまりです。ターミナルを閉じたり、別の日に作業を再開したりすると、通常は前の会話が引き継がれません。

ECC はこの問題を解決するため、会話の要約を `~/.claude/session-data/` に自動保存し、次のセッション開始時に注入します。

```
セッション A（今日）               セッション B（明日）
━━━━━━━━━━━━━━━━━━                ━━━━━━━━━━━━━━━━━━
会話が進む                         claude を起動
  ↓                                  ↓
Stop フックが要約を保存 → → → SessionStart が要約を注入
  (~/.claude/session-data/)            ↓
                                  前回の文脈を把握した状態で再開
```

このため、「昨日どこまで実装したか」「どのファイルを変更したか」をいちいち説明し直す手間が省けます。

### サブエージェントで文脈を節約する

Claude Code の **サブエージェント**（Task ツールで起動される子エージェント）は、**メインセッションとは独立したコンテキストウィンドウ**で動作します。

```
メインセッション（コンテキスト節約）
     │
     └─ Task ツールでサブエージェントを起動
           │
           ├─ 独立した窓で 20 ファイル読む・調査する
           │
           └─ 結果の要約だけをメインに返す
                 ↓
       メインの窓には「要約」しか追加されない
```

大量のファイル読み込みや調査はサブエージェントに委譲することで、メインセッションのコンテキストを大幅に節約できます。「20 ファイル読んでも要約しか返ってこない」という構造がポイントです。

### 拡張思考（Extended Thinking）とトークン

Claude の **拡張思考**（extended thinking）は、モデルが最終回答の前に内部で推論を展開する機能です。品質が上がる反面、推論プロセス自体がトークンを大量消費します。設定の `MAX_THINKING_TOKENS` でその上限を調整できます。複雑なアーキテクチャ設計には有効ですが、単純なタスクでは高コストになります（詳細は後述の「[MAX_THINKING_TOKENS の調整](#max_thinking_tokens-の調整)」を参照）。

---

## セッション永続化

### 保存先と命名規則

ECC のセッションデータは以下のディレクトリに保存されます（`scripts/lib/utils.js`）。

```
~/.claude/session-data/   ← 現行（正規）ディレクトリ
~/.claude/sessions/       ← 旧ディレクトリ（後方互換のため読み取り対象）
```

ファイル名パターン（`session-manager.js`）:

```
YYYY-MM-DD-<shortId>-session.tmp   ← 現行形式
YYYY-MM-DD-session.tmp             ← 旧形式（レガシー）
```

`<shortId>` に使える文字: `a-z`、`A-Z`、`0-9`、`-`、`_`（先頭はハイフン不可）。最低 1 文字。推奨は 8 文字以上の英数字とハイフンの組み合わせです。

実際の生成例（`session-end.js`）: トランスクリプトパスの UUID 末尾 8 文字を `sanitizeSessionId()` で処理したものが `shortId` になります。これにより親セッションとサブエージェントのファイルが衝突しません。

### セッションファイルの構造

セッションファイルは Markdown 形式です。

```markdown
# Session: 2026-01-17

**Date:** 2026-01-17
**Started:** 14:00
**Last Updated:** 16:30
**Project:** my-project
**Branch:** feature/auth
**Worktree:** /path/to/project

---

<!-- ECC:SUMMARY:START -->
## Session Summary

### Tasks
- [ユーザーメッセージ要約 ...]

### Files Modified
- src/auth.ts
- tests/auth.test.ts

### Tools Used
Edit, Write, Bash

### Stats
- Total user messages: 12
<!-- ECC:SUMMARY:END -->

### Notes for Next Session
-

### Context to Load
```
[relevant files]
```
```

`<!-- ECC:SUMMARY:START -->` と `<!-- ECC:SUMMARY:END -->` マーカーで囲まれた部分は、Stop フックが応答ごとに上書き更新します。マーカー外のセクション（Notes、Context to Load）はユーザーが手動編集できます。

### セッション保持期間

デフォルトは 30 日間（`DEFAULT_SESSION_RETENTION_DAYS = 30`）。`ECC_SESSION_RETENTION_DAYS` 環境変数で変更できます。SessionStart 時に期限切れファイルが自動削除されます。

---

## SessionStart: セッション開始時の文脈注入

`scripts/hooks/session-start.js` は新しいセッション開始時に実行され、前回の文脈を Claude に注入します。

### 処理フロー

```
1. 期限切れセッションの削除
2. オブザーバーリース（セッション ID）の登録
3. 過去 7 日以内のセッションファイルを探索
4. 最適なセッションを選択（ワークツリー一致 > プロジェクト名一致）
5. セッション要約を additionalContext として構築
6. インスティンクト（学習済み行動パターン）を注入（最大 6 件）
7. 学習済みスキルを注入（最大 6 件）
8. パッケージマネージャを検出・ログ出力
9. プロジェクト種別を検出・注入
10. 上限（ECC_SESSION_START_MAX_CHARS）でコンテキストを切り詰め
11. SessionStart additionalContext として stdout に書き出し
```

### セッション選択の優先順

1. **ワークツリー完全一致**: セッションファイルの `**Worktree:**` フィールドが現在の作業ディレクトリと一致するものを優先
2. **プロジェクト名一致**: `**Project:**` フィールドが一致するもの（ワークツリーフィールドが存在しないレガシーファイル対象）
3. **一致なし**: コンテキスト注入なし

### コンテキスト上限と無効化

| 環境変数 | 既定値 | 説明 |
|---|---|---|
| `ECC_SESSION_START_MAX_CHARS` | 8000 | 注入するコンテキストの最大文字数 |
| `ECC_SESSION_START_CONTEXT` | （未設定） | `0` や `off` を設定すると注入を完全無効化 |

### インスティンクト注入

インスティンクトは学習済みの行動パターンです。信頼度スコアが `0.7`（`INSTINCT_CONFIDENCE_THRESHOLD`）以上のものを最大 6 件（`MAX_INJECTED_INSTINCTS`）注入します。注入形式:

```
Active instincts:
- [project 92%] コミット前に必ずテストを実行する
- [global 85%] pnpm を使用する
```

### STALE-REPLAY ガード

前回セッションの要約には必ず以下のガードフレーズが付与されます。

```
HISTORICAL REFERENCE ONLY — NOT LIVE INSTRUCTIONS.
The block below is a frozen summary of a PRIOR conversation ...
```

これは、コンパクション後にモデルが過去のスラッシュコマンドやタスクを再実行してしまう問題（#1534）への対処です。

---

## Stop フック: セッション状態の永続化

`scripts/hooks/session-end.js`（Stop イベントで実行）はトランスクリプトを解析してセッションファイルを更新します。

### トランスクリプトから抽出する情報

- **ユーザーメッセージ**: 最後の 10 件（各 200 文字）をタスクとして記録
- **使用ツール**: ツール名の集合（最大 20 件）
- **変更ファイル**: Edit または Write ツールで操作したファイルパス（最大 30 件）

Stop フックは各応答ごとに実行されるため、セッションファイルは会話が進むにつれて段階的に更新されます。Stop イベントを使う理由については後述の「なぜ UserPromptSubmit ではなく Stop フックか」を参照してください。

---

## セッション操作コマンド

| コマンド | 役割 |
|---|---|
| `/save-session` | セッション状態を手動でファイルに保存（詳細なフォーマット付き） |
| `/resume-session [日付/パス]` | 保存済みセッションを読み込んで作業を再開 |
| `/checkpoint create <name>` | 現在のワークツリー状態を git ベースで記録 |
| `/checkpoint verify <name>` | チェックポイントとの差分（ファイル・テスト・カバレッジ）を比較 |
| `/compact` | コンテキストを手動で圧縮（戦略的に使用） |

`/save-session` と自動 Stop フックの使い分け: 自動 Stop フックは毎応答時にトランスクリプトから要約を機械的に生成します。`/save-session` はより詳細な構造化フォーマット（What WORKED・What Did NOT Work・Exact Next Step など）でセッションを記録します。複雑な問題を解いた後や、翌日に確実に再開したい場合は手動の `/save-session` を推奨します。

---

## コンテキストモード（contexts/）

`contexts/` ディレクトリには、作業モードに応じたシステムプロンプトスニペットが格納されています。

### dev.md（開発モード）

```
Mode: Active development
Focus: Implementation, coding, building features
Priorities: 1. Get it working  2. Get it right  3. Get it clean
Tools to favor: Edit, Write, Bash, Grep
```

コードを書くことを最優先にし、完璧よりも動くものを先に作るモードです。

### research.md（調査モード）

```
Mode: Exploration, investigation, learning
Focus: Understanding before acting
Process: Understand → Explore → Hypothesis → Verify → Summarize
Tools to favor: Read, Grep, Glob, WebSearch
```

コードを書く前に広く調査し、仮説を立てて検証するモードです。

### review.md（レビューモード）

```
Mode: PR review, code analysis
Focus: Quality, security, maintainability
Output: Group findings by file, severity first (critical > high > medium > low)
```

問題を重大度順に整理し、指摘と修正案を同時に提示するモードです。

### 切り替え方法

コマンドラインエイリアスで動的に注入できます（`the-longform-guide.md`）。

```bash
alias claude-dev='claude --system-prompt "$(cat ~/.claude/contexts/dev.md)"'
alias claude-review='claude --system-prompt "$(cat ~/.claude/contexts/review.md)"'
alias claude-research='claude --system-prompt "$(cat ~/.claude/contexts/research.md)"'
```

システムプロンプトの内容はユーザーメッセージよりも高い権限を持ちます。

---

## トークン経済（ベストプラクティス）

> ここからの節は、コンテキスト・トークンを実際にどう管理するかの **実践的ベストプラクティス** を詳しく解説します。上で学んだ基礎知識を踏まえて読んでください。

### モデル選択戦略

使用するモデルはタスクの複雑さに応じて選択します（`docs/token-optimization.md`）。

| モデル | 最適なタスク | コスト |
|---|---|---|
| **Haiku** | サブエージェントの探索・ファイル読み取り・シンプルな検索 | 最低 |
| **Sonnet** | 日常のコーディング・レビュー・テスト作成・実装 | 中 |
| **Opus** | 複雑なアーキテクチャ・微妙なバグのデバッグ・セキュリティ分析 | 最高 |

既定モデルを `sonnet` に設定することで、タスクの約 80-90% をカバーしつつコストを約 60% 削減できます。

```json
// ~/.claude/settings.json
{
  "model": "sonnet",
  "env": {
    "CLAUDE_CODE_SUBAGENT_MODEL": "haiku"
  }
}
```

`CLAUDE_CODE_SUBAGENT_MODEL=haiku` を設定すると、サブエージェント（Task ツール）が Haiku で実行されます。探索・ファイル読み取り・テスト実行などは Haiku で十分であり、~80% の節約になります。

### MAX_THINKING_TOKENS の調整

| 設定 | 値 | 効果 |
|---|---|---|
| デフォルト | 31,999 | 各リクエストで最大 31,999 トークンの内部推論を実行 |
| 推奨 | 10,000 | 隠れたコストを ~70% 削減 |
| 不要時 | 0 | 拡張思考を完全無効化 |

```json
{
  "env": {
    "MAX_THINKING_TOKENS": "10000"
  }
}
```

複雑なアーキテクチャ設計や多段階推論が必要な場合は高めの値を設定し、単純な実装タスクでは `0` も選択肢です。

### MCP を CLI と Skill で置換してコンテキストを節約

有効化された MCP サーバーは、接続済みでなくてもツール定義がコンテキストウィンドウを消費します。上限の目安は**プロジェクトあたり 10 個以内**です（`docs/token-optimization.md`）。

置換の考え方:

```
GitHub MCP → gh コマンド + /gh-pr スラッシュコマンド
Supabase MCP → Supabase CLI + skill
Vercel MCP → vercel CLI + skill
```

CLI ベースの方法は MCP と同等の機能を提供しながら、コンテキストウィンドウへの影響がゼロです。

---

## コンテキストのライフサイクル

### 4 段階サイクル

長期タスクは以下の段階で管理します。

```
探索 (research) → 計画 (plan) → 実装 (implement) → 検証 (verify)
```

各段階の移行時に重要なのは、**生の探索文脈を次の段階に持ち越さない**ことです。探索段階で集めた大量のコードリーディング結果や試行錯誤は、計画段階には不要です。

### 戦略的 /compact の使いどころ

`/compact` を実行するべき場面:

- 探索が終わり、実装に移る前
- マイルストーンを完了した後
- デバッグが終わり、次の作業に移る前
- 大きなコンテキストシフトの前

`/compact` を実行してはいけない場面:

- 関連する変更の実装中
- アクティブな問題のデバッグ中
- 複数ファイルにまたがるリファクタリング中

ECC の `strategic-compact` スキル（`skills/strategic-compact/`）は、Edit/Write フックを通じて論理的な区切りで `/compact` を提案します（`pre:edit-write:suggest-compact`）。

### PreCompact フック

`pre:compact` フックは `/compact` が実行される直前に発火し、重要な状態をファイルに保存します。これにより圧縮後も文脈が失われません。

---

## 検証ループ: pass@k vs pass^k（ベストプラクティス）

> **なぜ検証ループが必要か**: 有限なコンテキストの中で「何度試行すれば十分か」を定量的に判断するための枠組みです。試行を増やせばトークンコストが上がるため、用途に応じた選択が重要です。

`the-longform-guide.md` で定義されている 2 つの評価指標です。

### pass@k: 少なくとも 1 回成功

k 回の試行のうち**少なくとも 1 回**成功すれば合格とする基準です。

```
pass@k の例
k=1: 70%  k=3: 91%  k=5: 97%
```

**使いどころ**: 動けばよいケース。プロトタイプ、探索的なコード生成、何らかの実装が得られれば十分な場合。

### pass^k: 全回成功

k 回の試行**すべて**が成功しなければならない厳格な基準です。

```
pass^k の例
k=1: 70%  k=3: 34%  k=5: 17%
```

**使いどころ**: 一貫性が必須なケース。本番環境のリグレッションテスト、セキュリティ検証、再現性が求められる処理。

### チェックポイント評価 vs 連続評価

| 方式 | 説明 | コマンド |
|---|---|---|
| チェックポイント評価 | 定義した基準に対して現在の状態を検証 | `/checkpoint verify <name>` |
| 連続評価 | N 分ごと、または主要な変更後に自動で実行 | 継続的テスト + ループ |

`/checkpoint create "feature-start"` → 実装 → `/ checkpoint verify "feature-start"` というフローで進行状況を定量的に追跡します。

---

## サブエージェント・オーケストレーション

### 反復取得（Iterative Retrieval）

サブエージェントに探索させる際は、**目的のコンテキストを明示的に渡す**ことが重要です。「何を調べるのか」だけでなく「なぜ調べるのか」を伝えることで、サブエージェントが関連性の高い情報のみを返します。

```
悪い例: "auth.ts を読んで"
良い例: "JWT 検証ロジックを理解して、session-manager.js との統合方法を調べて。
         目的は次の認証フローの実装。結果は要点のみ返して。"
```

サブエージェント（Task ツール）は 20 ファイルを読んでも、返すのは要約のみです。メインセッションのコンテキストは汚染されません（`docs/token-optimization.md`）。

### フェーズ駆動オーケストレーション

複雑なタスクはフェーズに分割し、各段階の出力を次段階の入力にします。

```
RESEARCH  →  PLAN  →  IMPLEMENT  →  REVIEW  →  VERIFY
```

各フェーズの境界でセッションを圧縮（または保存・再開）することで、蓄積した文脈が次のフェーズに持ち込まれません。

実装例（多日タスクの場合）:

```
Day 1: RESEARCH フェーズ
  └─ /save-session で「調査結果・方針決定」を保存

Day 2: PLAN + IMPLEMENT フェーズ
  └─ /resume-session で Day 1 の要点を読み込み
  └─ /checkpoint create "plan-done"
  └─ 実装後 /checkpoint create "implement-done"

Day 3: REVIEW + VERIFY フェーズ
  └─ /resume-session で実装状態を確認
  └─ /checkpoint verify "implement-done"
  └─ セッション完了 /save-session
```

---

## 並列化: 最小限で最大成果（ベストプラクティス）

> **並列化とコンテキストの関係**: 並列セッションはそれぞれ独立したコンテキストウィンドウを持ちます。分割が多すぎると管理コストが上がり、逆に全体の整合性が崩れやすくなります。「最小限の並列化」という原則はトークン節約とアーキテクチャ整合性の両方に直結します。

### 基本方針

ターミナルを増やすことが目的ではなく、**真に並列化が必要な場合にのみ** 並列化します（`the-longform-guide.md`）。

> あなたのゴールは「最小限の並列化で最大の成果」を得ることです。

最初は 1 つのセッションで進め、ボトルネックが生じたときだけ分割します。

### git worktree による並列インスタンス

```bash
# feature ブランチと refactor ブランチを並列で作業
git worktree add ../project-feature-a feature-a
git worktree add ../project-refactor refactor-branch

# 各ワークツリーで独立した Claude セッション
cd ../project-feature-a && claude
cd ../project-refactor && claude
```

**重要**: コードが重複する箇所では並列化を避けます。各インスタンスのスコープを明確に定義します。

### Cascade メソッド

複数のインスタンスを管理する場合:

1. 新しいタスクを右のタブで開く
2. 左から右に向かって古い順に確認する
3. 同時に注目するのは最大 3-4 タスク

### 主な用途

- **コード変更**: メインセッションで行う
- **コードベースへの質問**: フォーク（サブエージェント）で行う
- **外部サービスのリサーチ**: 別セッションまたはサブエージェント

---

## なぜ UserPromptSubmit ではなく Stop フックか

継続学習やセッション記録に `UserPromptSubmit` ではなく `Stop` フックを使う理由:

- `UserPromptSubmit`: **毎メッセージ**に発火 → すべてのプロンプトに遅延が加わる
- `Stop`: **各応答終了後**に一度だけ発火 → セッション中の速度に影響しない

`the-longform-guide.md` より:

> UserPromptSubmit runs on every single message - adds latency to every prompt. Stop runs once at session end - lightweight, doesn't slow you down during the session.

ECC では継続学習（`observe-runner.js`）・セッション記録（`session-end.js`）・パターン評価（`evaluate-session.js`）・コストトラッキング（`cost-tracker.js`）のいずれも Stop フックで実行します。

---

## 実践: 長時間・多日タスクをセッション分割して回す

### ステップ 1: タスクを段階に分解する

大きなタスクを RESEARCH / PLAN / IMPLEMENT / REVIEW の段階に分解します。各段階の完了基準を先に定義します。

### ステップ 2: セッション開始時の儀式

```
1. /resume-session または SessionStart の自動注入を確認
2. 前回の "What Did NOT Work" を必ず読む
3. 今日の段階と完了基準を宣言する
```

### ステップ 3: 段階の変わり目で圧縮・保存

```bash
# 探索が終わったら
/compact   # または
/save-session  # より詳細な記録が必要な場合
```

### ステップ 4: チェックポイントで進捗を固定

```bash
# 主要なマイルストーンで
/checkpoint create "milestone-name"
```

### ステップ 5: セッション終了時の記録

Stop フックが自動的にトランスクリプトを解析して記録します。特に重要な情報（何が動いて何が動かなかったか）は `/save-session` で手動保存を推奨します。

### ステップ 6: 翌日の再開

```bash
# 新しいセッションを開始
claude
# SessionStart が自動的に前回の要約を注入する
# または手動で
/resume-session
```

### 具体例: 認証機能を 3 日で実装する場合

```
Day 1 (RESEARCH):
  午前: JWT と httpOnly Cookie の仕様調査（research モード）
  午後: 既存コードとの統合方法確認
  終了: /save-session で調査結果・方針を保存

Day 2 (IMPLEMENT):
  開始: /resume-session → Day 1 の調査結果を確認
  午前: 認証エンドポイントを実装
  /checkpoint create "auth-endpoints"
  午後: ミドルウェア実装
  /checkpoint create "middleware"
  終了: テスト実行確認

Day 3 (REVIEW + VERIFY):
  開始: /resume-session → 実装状態を確認
  午前: /checkpoint verify "auth-endpoints" でリグレッション確認
  午後: セキュリティレビュー（review モード）
  終了: PR 作成・/save-session で最終状態を記録
```

---

## 文脈・トークン管理の思想とベストプラクティス（まとめ）

Claude Code は「有限なコンテキストウィンドウの中でいかに品質を保つか」という問いに対し、複数の仕組みを組み合わせて答えています。ECC はその仕組みを活かすコンポーネント群を提供しています。初心者が最初に意識すべき原則を以下にまとめます。

### コンテキストを小さく保つ

- **不要なファイルを読まない**: 調査はサブエージェントに委譲し、要約だけを受け取る
- **MCPを絞る**: 有効化した MCP のツール定義はすべて窓を消費する。不要なサーバは無効化する（詳細は [09章](./09-mcp-and-integrations.md)）
- **適切なタイミングで `/compact`**: フェーズ移行時（探索→実装、実装→レビュー）に圧縮し、作業中は圧縮しない

### モデル階層を使い分ける

| 状況 | 推奨モデル | 理由 |
|---|---|---|
| 複雑なアーキテクチャ設計・難解なバグ解析 | Opus | 高難度タスクに高性能モデルを集中投入 |
| 日常的なコーディング・テスト・レビュー | Sonnet（既定） | コストと品質のバランスが最良 |
| 探索・ファイル読み取り・シンプルな検索 | Haiku | 大量の軽量タスクはコストを抑える |

サブエージェントを Haiku で動かす（`CLAUDE_CODE_SUBAGENT_MODEL=haiku`）と、同じ品質を低コストで実現できます。

### セッションを早めに要約・引き継ぐ

- **Stop フックの自動保存を信頼する**: 毎応答後にトランスクリプト要約が `~/.claude/session-data/` へ保存される
- **重要なフェーズの節目には `/save-session` を手動実行する**: 自動要約では残らない「何が機能しなかったか」「次の一手」を明示的に記録する
- **翌日の再開は `claude` を起動するだけ**: SessionStart フックが前回要約を自動注入する

### 検証はコスト意識を持って設計する

- **pass@k**: 「動けば OK」のプロトタイプ・探索フェーズで使う。試行ごとにコストがかかるため、k は必要最小限に
- **pass^k**: 本番リグレッション・セキュリティ検証など一貫性が必須の場面で使う。k を増やすほど信頼性は上がるがコストも上がる
- `/checkpoint` で進捗を定量的に固定し、後退を早期検知する

### 並列化は目的ではなく手段

- まず1セッションで進める。ボトルネックが生じて初めて分割を検討する
- 並列インスタンスのスコープを明確に区切り、コードの重複を避ける
- フォーク（サブエージェント）は「コードベースへの質問」に使い、「コード変更」はメインセッションで行う

### ECC の仕組みとの対応

| 原則 | ECCの仕組み |
|---|---|
| 文脈を小さく保つ | `strategic-compact` スキル、MCP→CLI 置換ガイド |
| セッションを引き継ぐ | `session-start.js` / `session-end.js` / `/save-session` / `/resume-session` |
| モデル階層を使い分ける | `settings.json` の `CLAUDE_CODE_SUBAGENT_MODEL` |
| 拡張思考を制御する | `MAX_THINKING_TOKENS` の環境変数 |
| フェーズを区切る | `/checkpoint create` / `/compact` / `contexts/` ディレクトリ |

---

## 関連章

- [07-hooks-and-runtime.md](./07-hooks-and-runtime.md) — SessionStart / Stop フックの実装詳細
- [05-agents.md](./05-agents.md) — サブエージェントのオーケストレーション
- [09-mcp-and-integrations.md](./09-mcp-and-integrations.md) — MCP の文脈コストと代替手段
- [13-applying-to-your-project.md](./13-applying-to-your-project.md) — 他プロジェクトへのセッション管理の適用

## 参照ソース

- `scripts/hooks/session-start.js` — SessionStart フック実装（コンテキスト注入・インスティンクト）
- `scripts/hooks/session-end.js` — Stop フック実装（トランスクリプト解析・セッション記録）
- `scripts/lib/session-manager.js` — セッション CRUD 操作ライブラリ
- `scripts/lib/utils.js` — `getSessionsDir()`、`getSessionSearchDirs()` など
- `contexts/dev.md` — 開発モードのコンテキスト
- `contexts/research.md` — 調査モードのコンテキスト
- `contexts/review.md` — レビューモードのコンテキスト
- `the-longform-guide.md` — トークン経済・並列化・検証ループの実践ガイド
- `docs/token-optimization.md` — トークン最適化の設定と手法
- `commands/save-session.md` — /save-session コマンド定義
- `commands/resume-session.md` — /resume-session コマンド定義
- `commands/checkpoint.md` — /checkpoint コマンド定義
