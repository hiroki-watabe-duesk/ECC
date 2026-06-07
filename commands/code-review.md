---
description: コードレビュー — ローカルのコミットされていない変更または GitHub PR（PR モードの場合は PR 番号/URL を渡す）
argument-hint: [pr-number | pr-url | blank for local review]
---

# コードレビュー

> PR レビューモードは Wirasm の PRPs-agentic-eng から採用。PRP ワークフローシリーズの一部。

**入力**: $ARGUMENTS

---

## モード選択

`$ARGUMENTS` に PR 番号、PR URL、または `--pr` が含まれる場合:
→ 以下の **PR レビューモード** に進む。

それ以外の場合:
→ **ローカルレビューモード** を使用する。

---

## ローカルレビューモード

コミットされていない変更の包括的なセキュリティと品質レビュー。

### フェーズ 1 — 収集

```bash
git diff --name-only HEAD
```

変更されたファイルがなければ、「レビュー対象がありません」と停止する。

### フェーズ 2 — レビュー

変更された各ファイルを全体的に読む。以下を確認する:

**セキュリティ問題（重大）:**
- ハードコードされた認証情報、API キー、トークン
- SQL インジェクションの脆弱性
- XSS の脆弱性
- 入力バリデーションの欠如
- 安全でない依存関係
- パストラバーサルのリスク

**コード品質（高）:**
- 50 行を超える関数
- 800 行を超えるファイル
- 4 レベルを超えるネスト深度
- エラーハンドリングの欠如
- console.log 文
- TODO/FIXME コメント
- パブリック API に JSDoc がない

**ベストプラクティス（中）:**
- 変更パターン（代わりにイミュータブルを使用する）
- コード/コメントへの絵文字の使用
- 新しいコードに対するテストの欠如
- アクセシビリティの問題（a11y）

### フェーズ 3 — レポート

以下を含むレポートを生成する:
- 深刻度: CRITICAL（重大）、HIGH（高）、MEDIUM（中）、LOW（低）
- ファイルの場所と行番号
- 問題の説明
- 修正の提案

CRITICAL または HIGH の問題が見つかった場合はコミットをブロックする。
セキュリティの脆弱性があるコードは絶対に承認しない。

---

## PR レビューモード

包括的な GitHub PR レビュー — diff を取得し、全ファイルを読み、バリデーションを実行し、レビューを投稿する。

### フェーズ 1 — 取得

入力を解析して PR を特定する:

| 入力 | アクション |
|---|---|
| 番号（例: `42`） | PR 番号として使用 |
| URL（`github.com/.../pull/42`） | PR 番号を抽出 |
| ブランチ名 | `gh pr list --head <branch>` で PR を検索 |

```bash
gh pr view <NUMBER> --json number,title,body,author,baseRefName,headRefName,changedFiles,additions,deletions
gh pr diff <NUMBER>
```

PR が見つからない場合はエラーで停止する。後続フェーズのために PR メタデータを保存する。

### フェーズ 2 — コンテキスト

レビューコンテキストを構築する:

1. **プロジェクトルール** — `CLAUDE.md`、`.claude/docs/`、および貢献ガイドラインを読む
2. **計画成果物** — この PR に関連するコンテキストを `.claude/prds/`、`.claude/plans/`、`.claude/reviews/`、およびレガシーの `.claude/PRPs/{prds,plans,reports,reviews}/` で確認する
3. **PR の意図** — 目標、リンクされた Issue、テスト計画のために PR の説明を解析する
4. **変更ファイル** — すべての変更されたファイルを一覧し、タイプ（ソース、テスト、設定、ドキュメント）で分類する

### フェーズ 3 — レビュー

各変更ファイルを**全体的に**読む（diff ハンクだけでなく — 周囲のコンテキストが必要）。

PR レビューの場合、PR ヘッドリビジョンでファイルの全内容を取得する:
```bash
gh pr diff <NUMBER> --name-only | while IFS= read -r file; do
  gh api "repos/{owner}/{repo}/contents/$file?ref=<head-branch>" --jq '.content' | base64 -d
done
```

7 つのカテゴリーにわたってレビューチェックリストを適用する:

| カテゴリー | 確認事項 |
|---|---|
| **正確性** | ロジックエラー、オフバイワン、null 処理、エッジケース、競合状態 |
| **型安全性** | 型の不一致、安全でないキャスト、`any` の使用、ジェネリクスの欠如 |
| **パターン準拠** | プロジェクト規約に一致（命名、ファイル構造、エラーハンドリング、インポート） |
| **セキュリティ** | インジェクション、認証ギャップ、シークレット露出、SSRF、パストラバーサル、XSS |
| **パフォーマンス** | N+1 クエリ、インデックスの欠如、無制限ループ、メモリリーク、大きなペイロード |
| **完全性** | テストの欠如、エラーハンドリングの欠如、マイグレーションの不完全さ、ドキュメントの欠如 |
| **保守性** | デッドコード、マジックナンバー、深いネスト、不明確な命名、型の欠如 |

各発見に深刻度を割り当てる:

| 深刻度 | 意味 | アクション |
|---|---|---|
| **CRITICAL** | セキュリティの脆弱性またはデータ損失リスク | マージ前に必ず修正 |
| **HIGH** | バグまたは問題を引き起こす可能性の高いロジックエラー | マージ前に修正すべき |
| **MEDIUM** | コード品質の問題またはベストプラクティスの欠如 | 修正を推奨 |
| **LOW** | スタイルの指摘または軽微な提案 | 任意 |

### フェーズ 4 — バリデーション

利用可能なバリデーションコマンドを実行する:

設定ファイル（`package.json`、`Cargo.toml`、`go.mod`、`pyproject.toml` など）からプロジェクトタイプを検出し、適切なコマンドを実行する:

**Node.js / TypeScript**（`package.json` がある場合）:
```bash
npm run typecheck 2>/dev/null || npx tsc --noEmit 2>/dev/null  # 型チェック
npm run lint                                                    # リント
npm test                                                        # テスト
npm run build                                                   # ビルド
```

**Rust**（`Cargo.toml` がある場合）:
```bash
cargo clippy -- -D warnings  # リント
cargo test                   # テスト
cargo build                  # ビルド
```

**Go**（`go.mod` がある場合）:
```bash
go vet ./...    # リント
go test ./...   # テスト
go build ./...  # ビルド
```

**Python**（`pyproject.toml` / `setup.py` がある場合）:
```bash
pytest  # テスト
```

検出されたプロジェクトタイプに該当するコマンドのみを実行する。それぞれの合否を記録する。

### フェーズ 5 — 判定

発見に基づいて推奨を形成する:

| 条件 | 判定 |
|---|---|
| CRITICAL/HIGH の問題がゼロ、バリデーション合格 | **APPROVE（承認）** |
| MEDIUM/LOW の問題のみ、バリデーション合格 | コメント付きで **APPROVE（承認）** |
| HIGH の問題またはバリデーション失敗 | **REQUEST CHANGES（変更要求）** |
| CRITICAL の問題 | **BLOCK（ブロック）** — マージ前に必ず修正 |

特例:
- ドラフト PR → 常に **COMMENT（コメント）** を使用（承認/ブロックは使わない）
- ドキュメント/設定のみの変更 → 軽めのレビュー、正確性に注力
- 明示的な `--approve` または `--request-changes` フラグ → 判定を上書き（ただし発見はすべて報告する）

### フェーズ 6 — レポート

`.claude/reviews/pr-<NUMBER>-review.md` にレビュー成果物を作成する（このワークストリームでレガシーの `.claude/PRPs/reviews/` をすでに使用している場合はそちらに作成する）:

```markdown
# PR レビュー: #<NUMBER> — <TITLE>

**レビュー日**: <date>
**作成者**: <author>
**ブランチ**: <head> → <base>
**判定**: APPROVE | REQUEST CHANGES | BLOCK

## 概要
<全体的な評価を 1〜2 文で>

## 発見事項

### CRITICAL
<発見事項、または「なし」>

### HIGH
<発見事項、または「なし」>

### MEDIUM
<発見事項、または「なし」>

### LOW
<発見事項、または「なし」>

## バリデーション結果

| チェック | 結果 |
|---|---|
| 型チェック | 合格 / 不合格 / スキップ |
| リント | 合格 / 不合格 / スキップ |
| テスト | 合格 / 不合格 / スキップ |
| ビルド | 合格 / 不合格 / スキップ |

## レビューしたファイル
<変更タイプ付きのファイル一覧: 追加/変更/削除>
```

### フェーズ 7 — 公開

レビューを GitHub に投稿する:

```bash
# APPROVE の場合
gh pr review <NUMBER> --approve --body "<レビューの概要>"

# REQUEST CHANGES の場合
gh pr review <NUMBER> --request-changes --body "<必要な修正を含む概要>"

# COMMENT のみの場合（ドラフト PR または情報提供）
gh pr review <NUMBER> --comment --body "<概要>"
```

特定の行へのインラインコメントには、GitHub レビューコメント API を使用する:
```bash
gh api "repos/{owner}/{repo}/pulls/<NUMBER>/comments" \
  -f body="<コメント>" \
  -f path="<ファイル>" \
  -F line=<行番号> \
  -f side="RIGHT" \
  -f commit_id="$(gh pr view <NUMBER> --json headRefOid --jq .headRefOid)"
```

または、複数のインラインコメントを含む 1 つのレビューを一度に投稿する:
```bash
gh api "repos/{owner}/{repo}/pulls/<NUMBER>/reviews" \
  -f event="COMMENT" \
  -f body="<全体の概要>" \
  --input comments.json  # [{"path": "file", "line": N, "body": "comment"}, ...]
```

### フェーズ 8 — 出力

ユーザーに報告する:

```
PR #<NUMBER>: <TITLE>
判定: <APPROVE|REQUEST_CHANGES|BLOCK>

問題: <critical_count> 重大、<high_count> 高、<medium_count> 中、<low_count> 低
バリデーション: <pass_count>/<total_count> チェック合格

成果物:
  レビュー: .claude/reviews/pr-<NUMBER>-review.md
  GitHub: <PR URL>

次のステップ:
  - <判定に基づいたコンテキストに応じた提案>
```

---

## エッジケース

- **`gh` CLI がない**: ローカルのみのレビューにフォールバック（diff を読み、GitHub への公開はスキップ）。ユーザーに警告する。
- **ブランチが乖離している**: レビュー前に `git fetch origin && git rebase origin/<base>` を提案する。
- **大きな PR（ファイル数が 50 超）**: レビュー範囲について警告する。ソースの変更を優先し、次にテスト、最後に設定/ドキュメントを確認する。
