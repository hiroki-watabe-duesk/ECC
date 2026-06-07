---
name: search-first
description: コードを書く前に既存のツール・ライブラリ・パターンを調査するワークフロー。カスタムコードを書く前に既存解決策を探す。researcherエージェントを呼び出す。
origin: ECC
---

# /search-first — コードを書く前に調査する

「実装する前に既存の解決策を探す」ワークフローを体系化します。

## トリガー

次の場合にこのスキルを使用してください:
- 既存の解決策がありそうな新機能を開始するとき
- 依存関係やインテグレーションを追加するとき
- ユーザーが「X機能を追加して」と言い、コードを書こうとするとき
- 新しいユーティリティ・ヘルパー・抽象化を作成する前

## ワークフロー

```
┌─────────────────────────────────────────────┐
│  0. ツール利用可能性の事前確認               │
│     頼る前に検索チャネルを確認;             │
│     スキップしたチャネルは正直に報告する    │
├─────────────────────────────────────────────┤
│  1. 要件分析                                │
│     必要な機能を定義する                    │
│     言語/フレームワークの制約を確認する     │
├─────────────────────────────────────────────┤
│  2. 並列検索 (researcherエージェント)        │
│     ┌──────────┐ ┌──────────┐ ┌──────────┐  │
│     │  npm /   │ │  MCP /   │ │  GitHub / │  │
│     │  PyPI    │ │  Skills  │ │  Web      │  │
│     └──────────┘ └──────────┘ └──────────┘  │
├─────────────────────────────────────────────┤
│  3. 評価                                    │
│     候補をスコアリング(機能性、保守性、     │
│     コミュニティ、ドキュメント、ライセンス、│
│     依存関係)                               │
├─────────────────────────────────────────────┤
│  4. 決定                                    │
│     ┌─────────┐  ┌──────────┐  ┌─────────┐  │
│     │  採用   │  │ 拡張/   │  │ カスタム │  │
│     │ そのまま│  │ ラップ   │  │  構築   │  │
│     └─────────┘  └──────────┘  └─────────┘  │
├─────────────────────────────────────────────┤
│  5. 実装                                    │
│     パッケージのインストール / MCP設定 /    │
│     最小限のカスタムコードを書く           │
└─────────────────────────────────────────────┘
```

## 意思決定マトリクス

| シグナル | 行動 |
|--------|--------|
| 完全一致、適切に保守されている、MIT/Apache | **採用** — そのままインストールして使用 |
| 部分一致、良好な基盤 | **拡張** — インストール＋薄いラッパーを書く |
| 複数の弱い一致 | **合成** — 2〜3の小さなパッケージを組み合わせる |
| 適切なものが見つからない | **構築** — カスタムを書く（ただし調査結果を参考にする） |

## 使い方

### ステップ0: ツール利用可能性の事前確認

これはエージェント向けのガイダンスであり、実行可能なセットアップスクリプトではありません。目の前のタスクとプロジェクトに関連するチャネルのみ確認してください。

| チャネル | 確認方法 | 利用できない場合 |
|---------|-------|------------|
| リポジトリ検索 | `rg --files` および対象を絞った `rg` クエリ | 可視ファイルのみを調査したことを明記する |
| パッケージレジストリ | `npm --version`、`python -m pip --version`、またはプロジェクトのパッケージマネージャー | ウェブ/ドキュメント検索を使用し、レジストリを網羅したとは主張しない |
| GitHub CLI | `gh auth status` | 公開ウェブまたはローカルgit履歴のみ使用 |
| MCP/docsツール | 利用可能なツール一覧またはローカルMCP設定 | 公式ドキュメント/ウェブ検索にフォールバック |
| スキルディレクトリ | `ls ~/.claude/skills ~/.codex/skills`（該当する場合） | ローカルのスキルカタログは利用不可と伝える |

### クイックモード（インライン）

ユーティリティを書いたり機能を追加する前に、次の流れを頭の中で実行してください:

0. これはリポジトリ内に既に存在するか？ → まず関連するモジュール/テストを `rg` で検索する
1. よくある問題か？ → npm/PyPIで検索する
2. MCPがあるか？ → `~/.claude/settings.json` を確認して検索する
3. スキルがあるか？ → `~/.claude/skills/` を確認する
4. GitHubに実装例/テンプレートがあるか？ → 新規コードを書く前に、保守されているOSSのGitHubコード検索を実行する

### フルモード（エージェント）

非自明な機能については、researcherエージェントを起動してください:

```
Agent(subagent_type="general-purpose", prompt="
  Research existing tools for: [DESCRIPTION]
  Language/framework: [LANG]
  Constraints: [ANY]

  Search: npm/PyPI, MCP servers, Claude Code skills, GitHub
  Return: Structured comparison with recommendation
")
```

古いClaude Codeドキュメントでは `Task(...)` と呼んでいる場合があります。現在のハーネスが公開しているエージェント/サブエージェントのツール名を使用してください。

## カテゴリ別検索ショートカット

### 開発ツール
- リンティング → `eslint`、`ruff`、`textlint`、`markdownlint`
- フォーマット → `prettier`、`black`、`gofmt`
- テスト → `jest`、`pytest`、`go test`
- プリコミット → `husky`、`lint-staged`、`pre-commit`

### AI/LLMインテグレーション
- Claude SDK → 最新ドキュメントはContext7を参照
- プロンプト管理 → MCPサーバーを確認
- ドキュメント処理 → `unstructured`、`pdfplumber`、`mammoth`

### データ＆API
- HTTPクライアント → `httpx`（Python）、`ky`/`undici`（Node）
- バリデーション → `zod`（TS）、`pydantic`（Python）
- データベース → まずMCPサーバーを確認

### コンテンツ＆公開
- Markdown処理 → `remark`、`unified`、`markdown-it`
- 画像最適化 → `sharp`、`imagemin`

## インテグレーションポイント

### plannerエージェントとの連携
plannerはフェーズ1（アーキテクチャレビュー）の前にresearcherを呼び出すべきです:
- researcherが利用可能なツールを特定する
- plannerがそれらを実装計画に組み込む
- 計画での「車輪の再発明」を回避する

### architectエージェントとの連携
architectはresearcherに以下を相談すべきです:
- 技術スタックの選定
- インテグレーションパターンの探索
- 既存のリファレンスアーキテクチャ

### iterative-retrievalスキルとの連携
段階的な発見のために組み合わせます:
- サイクル1: 広範な検索（npm、PyPI、MCP）
- サイクル2: 上位候補を詳しく評価する
- サイクル3: プロジェクトの制約との互換性をテストする

## 例

### 例1: 「デッドリンクチェックを追加する」
```
Need: Check markdown files for broken links
Search: npm "markdown dead link checker"
Found: textlint-rule-no-dead-link (score: 9/10)
Action: ADOPT — npm install textlint-rule-no-dead-link
Result: Zero custom code, battle-tested solution
```

### 例2: 「HTTPクライアントラッパーを追加する」
```
Need: Resilient HTTP client with retries and timeout handling
Search: npm "http client retry", PyPI "httpx retry"
Found: got (Node) with retry plugin, httpx (Python) with built-in retry
Action: ADOPT — use got/httpx directly with retry config
Result: Zero custom code, production-proven libraries
```

### 例3: 「設定ファイルリンターを追加する」
```
Need: Validate project config files against a schema
Search: npm "config linter schema", "json schema validator cli"
Found: ajv-cli (score: 8/10)
Action: ADOPT + EXTEND — install ajv-cli, write project-specific schema
Result: 1 package + 1 schema file, no custom validation logic
```

## アンチパターン

- **コードに飛びつく**: 既存のものがあるか確認せずにユーティリティを書く
- **MCPを無視する**: MCPサーバーが既に機能を提供しているか確認しない
- **黙ってスキップする**: 検索チャネルが利用できなかったのに「何も見つからなかった」と報告する
- **過度なカスタマイズ**: ライブラリを過剰にラップしてそのメリットを失う
- **依存関係の肥大化**: 一つの小さな機能のために巨大なパッケージをインストールする
