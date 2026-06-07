---
name: plankton-code-quality
description: "Planktonを使った書き込み時のコード品質強制 — ファイル編集のたびにフックを介して自動フォーマット・リンティング・Claudeによる修正を実行する。"
origin: community
---

# Plankton コード品質スキル

Plankton（クレジット: @alxfazio）のインテグレーションリファレンス。Claude Codeの書き込み時コード品質強制システムです。Planktonは PostToolUse フックを通じてすべてのファイル編集時にフォーマッターとリンターを実行し、エージェントが見逃した違反を修正するためにClaudeサブプロセスを起動します。

## 使用すべき時

- すべてのファイル編集で（コミット時だけでなく）自動フォーマットとリンティングが必要な場合
- エージェントが修正ではなくリンター設定を変更して通過させることへの防衛が必要な場合
- 修正のためのティアモデルルーティングが必要な場合（シンプルなスタイルはHaiku、ロジックはSonnet、型はOpus）
- 複数言語で作業する場合（Python、TypeScript、Shell、YAML、JSON、TOML、Markdown、Dockerfile）

## 動作原理

### 3フェーズアーキテクチャ

Claude Codeがファイルを編集または書き込むたびに、PlanktonのPostToolUseフック `multi_linter.sh` が実行されます:

```
フェーズ1: 自動フォーマット（無音）
├─ フォーマッターを実行（ruff format、biome、shfmt、taplo、markdownlint）
├─ 問題の40〜50%を無音で修正
└─ メインエージェントへの出力なし

フェーズ2: 違反の収集（JSON）
├─ リンターを実行して修正不可能な違反を収集
├─ 構造化JSONを返す: {line, column, code, message, linter}
└─ まだメインエージェントへの出力なし

フェーズ3: 委任＋検証
├─ 違反JSONと共にclaude -pサブプロセスを起動
├─ 違反の複雑さに基づいてモデルティアにルーティング:
│   ├─ Haiku: フォーマット、インポート、スタイル（E/W/Fコード）— タイムアウト120秒
│   ├─ Sonnet: 複雑さ、リファクタリング（C901、PLRコード）— タイムアウト300秒
│   └─ Opus: 型システム、深い推論（unresolved-attribute）— タイムアウト600秒
├─ フェーズ1+2を再実行して修正を検証
└─ クリーンならExit 0、違反が残ればExit 2（メインエージェントに報告）
```

### メインエージェントが見るもの

| シナリオ | エージェントが見るもの | フック終了 |
|----------|-----------|-----------|
| 違反なし | 何もない | 0 |
| サブプロセスですべて修正 | 何もない | 0 |
| サブプロセス後も違反が残る | `[hook] N violation(s) remain` | 2 |
| 勧告（重複、旧ツール） | `[hook:advisory] ...` | 0 |

メインエージェントはサブプロセスが修正できなかった問題のみを見ます。ほとんどの品質問題は透過的に解決されます。

### 設定保護（ルールゲーミングへの防衛）

LLMはコードを修正する代わりに `.ruff.toml` や `biome.json` を変更してルールを無効にしようとします。Planktonは3層でこれをブロックします:

1. **PreToolUseフック** — `protect_linter_configs.sh` が発生前にすべてのリンター設定への編集をブロック
2. **Stopフック** — `stop_config_guardian.sh` がセッション終了時に `git diff` で設定変更を検出
3. **保護ファイルリスト** — `.ruff.toml`、`biome.json`、`.shellcheckrc`、`.yamllint`、`.hadolint.yaml` 等

### パッケージマネージャー強制

Bashに対するPreToolUseフックがレガシーパッケージマネージャーをブロックします:
- `pip`、`pip3`、`poetry`、`pipenv` → ブロック（`uv` を使用すること）
- `npm`、`yarn`、`pnpm` → ブロック（`bun` を使用すること）
- 許可される例外: `npm audit`、`npm view`、`npm publish`

## セットアップ

### クイックスタート

> **注意:** Planktonはそのリポジトリから手動インストールが必要です。インストール前にコードをレビューしてください。

```bash
# コア依存関係のインストール
brew install jaq ruff uv

# Pythonリンターのインストール
uv sync --all-extras

# Claude Codeを起動 — フックが自動的に有効化される
claude
```

インストールコマンドなし、プラグイン設定なし。`.claude/settings.json` のフックはPlanktonディレクトリでClaude Codeを実行すると自動的に取得されます。

### プロジェクト別インテグレーション

Planktonフックを自分のプロジェクトで使用するには:

1. `.claude/hooks/` ディレクトリをプロジェクトにコピーする
2. `.claude/settings.json` のフック設定をコピーする
3. リンター設定ファイル（`.ruff.toml`、`biome.json` 等）をコピーする
4. 使用する言語のリンターをインストールする

### 言語別依存関係

| 言語 | 必須 | オプション |
|----------|----------|----------|
| Python | `ruff`、`uv` | `ty`（型）、`vulture`（デッドコード）、`bandit`（セキュリティ） |
| TypeScript/JS | `biome` | `oxlint`、`semgrep`、`knip`（デッドエクスポート） |
| Shell | `shellcheck`、`shfmt` | — |
| YAML | `yamllint` | — |
| Markdown | `markdownlint-cli2` | — |
| Dockerfile | `hadolint`（>= 2.12.0） | — |
| TOML | `taplo` | — |
| JSON | `jaq` | — |

## ECCとの組み合わせ

### 補完的、重複なし

| 関心事 | ECC | Plankton |
|---------|-----|----------|
| コード品質強制 | PostToolUseフック（Prettier、tsc） | PostToolUseフック（20以上のリンター＋サブプロセス修正） |
| セキュリティスキャン | AgentShield、security-reviewerエージェント | Bandit（Python）、Semgrep（TypeScript） |
| 設定保護 | — | PreToolUseブロック＋Stopフック検出 |
| パッケージマネージャー | 検出＋セットアップ | 強制（レガシーPMをブロック） |
| CIインテグレーション | — | git用プリコミットフック |
| モデルルーティング | 手動（`/model opus`） | 自動（違反の複雑さ → ティア） |

### 推奨する組み合わせ

1. ECCをプラグインとしてインストール（エージェント、スキル、コマンド、ルール）
2. 書き込み時の品質強制のためにPlanktonフックを追加
3. セキュリティ監査にAgentShieldを使用
4. PRの最終ゲートとしてECCのverification-loopを使用

### フックの競合を避ける

ECCとPlanktonフックの両方を実行する場合:
- ECCのPrettierフックとPlanktonのbiomeフォーマッターがJS/TSファイルで競合する可能性がある
- 解決策: PlanktonとともにECCのPretier PostToolUseフックを無効にする（Planktonのbiomeはより包括的）
- 異なるファイルタイプで両方を共存させることができる（ECCはPlanktonが対応していないものを処理する）

## 設定リファレンス

PlanktonのすべてのビヘイビアはPlanktonの `.claude/hooks/config.json` で制御されます:

```json
{
  "languages": {
    "python": true,
    "shell": true,
    "yaml": true,
    "json": true,
    "toml": true,
    "dockerfile": true,
    "markdown": true,
    "typescript": {
      "enabled": true,
      "js_runtime": "auto",
      "biome_nursery": "warn",
      "semgrep": true
    }
  },
  "phases": {
    "auto_format": true,
    "subprocess_delegation": true
  },
  "subprocess": {
    "tiers": {
      "haiku":  { "timeout": 120, "max_turns": 10 },
      "sonnet": { "timeout": 300, "max_turns": 10 },
      "opus":   { "timeout": 600, "max_turns": 15 }
    },
    "volume_threshold": 5
  }
}
```

**主な設定:**
- 使用しない言語を無効にしてフックを高速化する
- `volume_threshold` — 違反がこの数を超えると自動的に上位モデルティアにエスカレートする
- `subprocess_delegation: false` — フェーズ3を完全にスキップ（違反のみ報告）

## 環境変数による上書き

| 変数 | 目的 |
|----------|---------|
| `HOOK_SKIP_SUBPROCESS=1` | フェーズ3をスキップして違反を直接報告する |
| `HOOK_SUBPROCESS_TIMEOUT=N` | ティアタイムアウトを上書きする |
| `HOOK_DEBUG_MODEL=1` | モデル選択の決定をログに記録する |
| `HOOK_SKIP_PM=1` | パッケージマネージャー強制をバイパスする |

## リファレンス

- Plankton（クレジット: @alxfazio）
- Plankton REFERENCE.md — 完全なアーキテクチャドキュメント（クレジット: @alxfazio）
- Plankton SETUP.md — 詳細インストールガイド（クレジット: @alxfazio）

## ECC v1.8 追加機能

### コピー可能なフックプロファイル

厳格な品質動作を設定する:

```bash
export ECC_HOOK_PROFILE=strict
export ECC_QUALITY_GATE_FIX=true
export ECC_QUALITY_GATE_STRICT=true
```

### 言語ゲートテーブル

- TypeScript/JavaScript: Biome優先、Prettierフォールバック
- Python: Ruffフォーマット/チェック
- Go: gofmt

### 設定改ざんガード

品質強制中に、同じイテレーション内での設定ファイルへの変更をフラグ立てする:

- `biome.json`、`.eslintrc*`、`prettier.config*`、`tsconfig.json`、`pyproject.toml`

違反を抑制するために設定が変更された場合、マージ前に明示的なレビューを要求する。

### CIインテグレーションパターン

ローカルフックと同じコマンドをCIで使用する:

1. フォーマッターチェックを実行する
2. lint/型チェックを実行する
3. strictモードで早期失敗する
4. 修復サマリーを公開する

### ヘルスメトリクス

以下を追跡する:
- ゲートにフラグされた編集数
- 平均修復時間
- カテゴリ別の繰り返し違反
- ゲート失敗によるマージブロック数
