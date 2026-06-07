---
description: Claude を唯一のファイルシステム書き込み者として保持しながら、マルチモデルの実装計画を実行します。
---

# 実行 - マルチモデル協調実行

マルチモデル協調実行 - 計画からプロトタイプを取得 → Claude がリファクタリングして実装 → マルチモデルによる監査と納品。

$ARGUMENTS

---

## コアプロトコル

- **言語プロトコル**: ツール/モデルとのやり取りには**英語**を使用し、ユーザーとはユーザーの言語でコミュニケーションする
- **コード主権**: 外部モデルはファイルシステムへの**書き込みアクセスがゼロ**であり、全ての変更は Claude が行う
- **ダーティプロトタイプのリファクタリング**: Codex/Gemini の Unified Diff を「ダーティプロトタイプ」として扱い、本番グレードのコードにリファクタリングする必要がある
- **ストップロスメカニズム**: 現在のフェーズの出力が検証されるまで次のフェーズに進まない
- **前提条件**: ユーザーが `/ccg:plan` の出力に対して明示的に「Y」と返信した後にのみ実行する（欠如している場合、最初に確認しなければならない）

---

## マルチモデル呼び出し仕様

**呼び出し構文**（並列: `run_in_background: true` を使用）:

```
# Resume session call (recommended) - Implementation Prototype
Bash({
  command: "~/.claude/bin/codeagent-wrapper {{LITE_MODE_FLAG}}--backend <codex|gemini> {{GEMINI_MODEL_FLAG}}resume <SESSION_ID> - \"$PWD\" <<'EOF'
ROLE_FILE: <role prompt path>
<TASK>
Requirement: <task description>
Context: <plan content + target files>
</TASK>
OUTPUT: Unified Diff Patch ONLY. Strictly prohibit any actual modifications.
EOF",
  run_in_background: true,
  timeout: 3600000,
  description: "Brief description"
})

# New session call - Implementation Prototype
Bash({
  command: "~/.claude/bin/codeagent-wrapper {{LITE_MODE_FLAG}}--backend <codex|gemini> {{GEMINI_MODEL_FLAG}}- \"$PWD\" <<'EOF'
ROLE_FILE: <role prompt path>
<TASK>
Requirement: <task description>
Context: <plan content + target files>
</TASK>
OUTPUT: Unified Diff Patch ONLY. Strictly prohibit any actual modifications.
EOF",
  run_in_background: true,
  timeout: 3600000,
  description: "Brief description"
})
```

**監査呼び出し構文**（コードレビュー / 監査）:

```
Bash({
  command: "~/.claude/bin/codeagent-wrapper {{LITE_MODE_FLAG}}--backend <codex|gemini> {{GEMINI_MODEL_FLAG}}resume <SESSION_ID> - \"$PWD\" <<'EOF'
ROLE_FILE: <role prompt path>
<TASK>
Scope: Audit the final code changes.
Inputs:
- The applied patch (git diff / final unified diff)
- The touched files (relevant excerpts if needed)
Constraints:
- Do NOT modify any files.
- Do NOT output tool commands that assume filesystem access.
</TASK>
OUTPUT:
1) A prioritized list of issues (severity, file, rationale)
2) Concrete fixes; if code changes are needed, include a Unified Diff Patch in a fenced code block.
EOF",
  run_in_background: true,
  timeout: 3600000,
  description: "Brief description"
})
```

**モデルパラメータに関する注意事項**:
- `{{GEMINI_MODEL_FLAG}}`: `--backend gemini` を使用する場合、`--gemini-model gemini-3-pro-preview` に置き換える（末尾のスペースに注意）。codex の場合は空文字列を使用する

**ロールプロンプト**:

| フェーズ | Codex | Gemini |
|-------|-------|--------|
| 実装 | `~/.claude/.ccg/prompts/codex/architect.md` | `~/.claude/.ccg/prompts/gemini/frontend.md` |
| レビュー | `~/.claude/.ccg/prompts/codex/reviewer.md` | `~/.claude/.ccg/prompts/gemini/reviewer.md` |

**セッション再利用**: `/ccg:plan` が SESSION_ID を提供した場合、コンテキストを再利用するために `resume <SESSION_ID>` を使用する。

**バックグラウンドタスクの待機**（最大タイムアウト 600000ms = 10 分）:

```
TaskOutput({ task_id: "<task_id>", block: true, timeout: 600000 })
```

**重要**:
- `timeout: 600000` を指定する必要がある。そうしないとデフォルトの 30 秒で早期タイムアウトが発生する
- 10 分後もまだ完了していない場合は、`TaskOutput` でポーリングを継続し、**プロセスを絶対に強制終了しない**
- タイムアウトにより待機をスキップした場合、**`AskUserQuestion` を呼び出して待機を継続するかタスクを強制終了するかユーザーに確認しなければならない**

---

## 実行ワークフロー

**実行タスク**: $ARGUMENTS

### フェーズ 0: 計画の読み込み

`[モード: 準備]`

1. **入力タイプの特定**:
   - 計画ファイルパス（例: `.claude/plan/xxx.md`）
   - 直接のタスク説明

2. **計画内容の読み込み**:
   - 計画ファイルパスが提供された場合、読み込んで解析する
   - タスクタイプ、実装ステップ、キーファイル、SESSION_ID を抽出する

3. **実行前の確認**:
   - 入力が「直接のタスク説明」または計画に `SESSION_ID` / キーファイルが欠如している場合: まずユーザーに確認する
   - ユーザーが計画に「Y」と返信したことを確認できない場合: 進める前に再確認する必要がある

4. **タスクタイプのルーティング**:

   | タスクタイプ | 検出 | ルート |
   |-----------|-----------|-------|
   | **フロントエンド** | ページ、コンポーネント、UI、スタイル、レイアウト | Gemini |
   | **バックエンド** | API、インターフェース、データベース、ロジック、アルゴリズム | Codex |
   | **フルスタック** | フロントエンドとバックエンドの両方を含む | Codex ∥ Gemini 並列 |

---

### フェーズ 1: クイックコンテキスト取得

`[モード: 取得]`

**ace-tool MCP が利用可能な場合**、クイックコンテキスト取得に使用する:

計画の「キーファイル」リストに基づき、`mcp__ace-tool__search_context` を呼び出す:

```
mcp__ace-tool__search_context({
  query: "<semantic query based on plan content, including key files, modules, function names>",
  project_root_path: "$PWD"
})
```

**取得戦略**:
- 計画の「キーファイル」テーブルからターゲットパスを抽出する
- エントリーファイル、依存モジュール、関連型定義をカバーするセマンティッククエリを構築する
- 結果が不十分な場合、1〜2 回の再帰的取得を追加する

**ace-tool MCP が利用できない場合**、フォールバックとして Claude Code 組み込みツールを使用する:
1. **Glob**: 計画の「キーファイル」テーブルからターゲットファイルを見つける（例: `Glob("src/components/**/*.tsx")`）
2. **Grep**: コードベース全体でキーシンボル、関数名、型定義を検索する
3. **Read**: 発見されたファイルを読み込んで完全なコンテキストを収集する
4. **Task（探索エージェント）**: より広い探索のために、`subagent_type: "Explore"` で `Task` を使用する

**取得後**:
- 取得したコードスニペットを整理する
- 実装のための完全なコンテキストを確認する
- フェーズ 3 に進む

---

### フェーズ 3: プロトタイプ取得

`[モード: プロトタイプ]`

**タスクタイプに基づいたルーティング**:

#### ルート A: フロントエンド/UI/スタイル → Gemini

**制限**: コンテキスト < 32k トークン

1. Gemini を呼び出す（`~/.claude/.ccg/prompts/gemini/frontend.md` を使用）
2. 入力: 計画内容 + 取得したコンテキスト + ターゲットファイル
3. 出力: `Unified Diff Patch ONLY. Strictly prohibit any actual modifications.`
4. **Gemini はフロントエンドデザインの権威であり、その CSS/React/Vue プロトタイプが最終的なビジュアルベースラインとなる**
5. **警告**: Gemini のバックエンドロジックの提案は無視する
6. 計画に `GEMINI_SESSION` が含まれている場合: `resume <GEMINI_SESSION>` を優先する

#### ルート B: バックエンド/ロジック/アルゴリズム → Codex

1. Codex を呼び出す（`~/.claude/.ccg/prompts/codex/architect.md` を使用）
2. 入力: 計画内容 + 取得したコンテキスト + ターゲットファイル
3. 出力: `Unified Diff Patch ONLY. Strictly prohibit any actual modifications.`
4. **Codex はバックエンドロジックの権威であり、その論理的推論とデバッグ能力を活用する**
5. 計画に `CODEX_SESSION` が含まれている場合: `resume <CODEX_SESSION>` を優先する

#### ルート C: フルスタック → 並列呼び出し

1. **並列呼び出し**（`run_in_background: true`）:
   - Gemini: フロントエンド部分を処理する
   - Codex: バックエンド部分を処理する
2. `TaskOutput` で両モデルの完全な結果を待つ
3. それぞれが `resume` のために計画の対応する `SESSION_ID` を使用する（欠如している場合は新しいセッションを作成する）

**上記の `マルチモデル呼び出し仕様` の `重要` 指示に従うこと**

---

### フェーズ 4: コード実装

`[モード: 実装]`

**コード主権者としての Claude が以下のステップを実行する**:

1. **Diff の読み込み**: Codex/Gemini が返した Unified Diff Patch を解析する

2. **メンタルサンドボックス**:
   - Diff をターゲットファイルに適用することをシミュレートする
   - 論理的な一貫性を確認する
   - 潜在的な競合や副作用を特定する

3. **リファクタリングとクリーンアップ**:
   - 「ダーティプロトタイプ」を**高い可読性と保守性を持つエンタープライズグレードのコード**にリファクタリングする
   - 冗長なコードを削除する
   - プロジェクトの既存コード標準への準拠を確認する
   - **必要でない限りコメント/ドキュメントを生成しない**。コードは自己説明的であるべき

4. **最小スコープ**:
   - 変更は要件スコープのみに限定する
   - 副作用の**必須レビュー**
   - ターゲットを絞った修正を行う

5. **変更の適用**:
   - Edit/Write ツールを使用して実際の変更を実行する
   - **必要なコードのみ変更し**、ユーザーの他の既存機能には絶対に影響を与えない

6. **自己検証**（強く推奨）:
   - プロジェクトの既存の lint / typecheck / テストを実行する（最小限の関連スコープを優先する）
   - 失敗した場合: まずリグレッションを修正し、その後フェーズ 5 に進む

---

### フェーズ 5: 監査と納品

`[モード: 監査]`

#### 5.1 自動監査

**変更が有効になった後、直ちに** Codex と Gemini に並列でコードレビューを呼び出す必要がある:

1. **Codex レビュー**（`run_in_background: true`）:
   - ROLE_FILE: `~/.claude/.ccg/prompts/codex/reviewer.md`
   - 入力: 変更された Diff + ターゲットファイル
   - 焦点: セキュリティ、パフォーマンス、エラーハンドリング、ロジックの正確性

2. **Gemini レビュー**（`run_in_background: true`）:
   - ROLE_FILE: `~/.claude/.ccg/prompts/gemini/reviewer.md`
   - 入力: 変更された Diff + ターゲットファイル
   - 焦点: アクセシビリティ、デザインの一貫性、ユーザー体験

`TaskOutput` で両モデルの完全なレビュー結果を待つ。コンテキストの一貫性のために、フェーズ 3 のセッション（`resume <SESSION_ID>`）の再利用を優先する。

#### 5.2 統合と修正

1. Codex + Gemini のレビューフィードバックを統合する
2. 信頼ルールで重み付けする: バックエンドは Codex に従い、フロントエンドは Gemini に従う
3. 必要な修正を実行する
4. 必要に応じてフェーズ 5.1 を繰り返す（リスクが許容範囲になるまで）

#### 5.3 納品確認

監査が合格した後、ユーザーに報告する:

```markdown
## 実行完了

### 変更サマリー
| ファイル | 操作 | 説明 |
|------|-----------|-------------|
| path/to/file.ts | Modified | Description |

### 監査結果
- Codex: <合格/N 件の問題を検出>
- Gemini: <合格/N 件の問題を検出>

### 推奨事項
1. [ ] <推奨テストステップ>
2. [ ] <推奨検証ステップ>
```

---

## キールール

1. **コード主権** – 全てのファイル変更は Claude が行い、外部モデルは書き込みアクセスがゼロ
2. **ダーティプロトタイプのリファクタリング** – Codex/Gemini の出力はドラフトとして扱い、リファクタリングが必要
3. **信頼ルール** – バックエンドは Codex に従い、フロントエンドは Gemini に従う
4. **最小変更** – 必要なコードのみ変更し、副作用を生まない
5. **必須監査** – 変更後は必ずマルチモデルのコードレビューを実行する

---

## 使用方法

```bash
# Execute plan file
/ccg:execute .claude/plan/feature-name.md

# Execute task directly (for plans already discussed in context)
/ccg:execute implement user authentication based on previous plan
```

---

## /ccg:plan との関係

1. `/ccg:plan` が計画 + SESSION_ID を生成する
2. ユーザーが「Y」で確認する
3. `/ccg:execute` が計画を読み込み、SESSION_ID を再利用して実装を実行する
