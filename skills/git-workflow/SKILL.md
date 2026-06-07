---
name: git-workflow
description: ブランチ戦略、コミット規約、マージとリベース、コンフリクト解消、あらゆる規模のチームのための共同開発ベストプラクティスを含むGitワークフローパターン。
origin: ECC
---

# Git ワークフローパターン

Gitバージョン管理、ブランチ戦略、共同開発のためのベストプラクティス。

## 有効化するタイミング

- 新プロジェクトのGitワークフローを設定するとき
- ブランチ戦略（GitFlow、トランクベース、GitHubフロー）を決定するとき
- コミットメッセージとPR説明を書くとき
- マージコンフリクトを解消するとき
- リリースとバージョンタグを管理するとき
- 新チームメンバーにGitのプラクティスをオンボーディングするとき

## ブランチ戦略

### GitHubフロー（シンプル、ほとんどのケースで推奨）

継続的なデプロイと小〜中規模チームに最適です。

```
main (保護された、常にデプロイ可能)
  │
  ├── feature/user-auth      → PR → mainにマージ
  ├── feature/payment-flow   → PR → mainにマージ
  └── fix/login-bug          → PR → mainにマージ
```

**ルール:**
- `main` は常にデプロイ可能
- `main` からフィーチャーブランチを作成する
- レビュー準備ができたらプルリクエストをオープンする
- 承認とCIパス後、`main` にマージする
- マージ後すぐにデプロイする

### トランクベース開発（高速チーム向け）

強力なCI/CDとフィーチャーフラグを持つチームに最適です。

```
main (トランク)
  │
  ├── 短命フィーチャー（最大1〜2日）
  ├── 短命フィーチャー
  └── 短命フィーチャー
```

**ルール:**
- 全員が `main` または非常に短命なブランチにコミットする
- フィーチャーフラグが未完成の作業を隠す
- マージ前にCIがパスしなければならない
- 1日に複数回デプロイする

### GitFlow（複雑、リリースサイクル駆動）

スケジュールされたリリースとエンタープライズプロジェクトに最適です。

```
main (プロダクションリリース)
  │
  └── develop (統合ブランチ)
        │
        ├── feature/user-auth
        ├── feature/payment
        │
        ├── release/1.0.0    → mainとdevelopにマージ
        │
        └── hotfix/critical  → mainとdevelopにマージ
```

**ルール:**
- `main` はプロダクションレディなコードのみを含む
- `develop` は統合ブランチ
- フィーチャーブランチは `develop` から、`develop` にマージバック
- リリースブランチは `develop` から、`main` と `develop` にマージ
- ホットフィックスブランチは `main` から、`main` と `develop` 両方にマージ

### どれを使うか

| 戦略 | チームサイズ | リリース頻度 | 最適な用途 |
|----------|-----------|-----------------|----------|
| GitHubフロー | 任意 | 継続的 | SaaS、Webアプリ、スタートアップ |
| トランクベース | 5人以上（経験者） | 1日に複数回 | 高速チーム、フィーチャーフラグ |
| GitFlow | 10人以上 | スケジュール | エンタープライズ、規制産業 |

## コミットメッセージ

### Conventional Commits 形式

```
<type>(<scope>): <subject>

[optional body]

[optional footer(s)]
```

### タイプ

| タイプ | 用途 | 例 |
|------|---------|---------|
| `feat` | 新機能 | `feat(auth): add OAuth2 login` |
| `fix` | バグ修正 | `fix(api): handle null response in user endpoint` |
| `docs` | ドキュメント | `docs(readme): update installation instructions` |
| `style` | フォーマット、コード変更なし | `style: fix indentation in login component` |
| `refactor` | コードリファクタリング | `refactor(db): extract connection pool to module` |
| `test` | テストの追加/更新 | `test(auth): add unit tests for token validation` |
| `chore` | メンテナンスタスク | `chore(deps): update dependencies` |
| `perf` | パフォーマンス改善 | `perf(query): add index to users table` |
| `ci` | CI/CD変更 | `ci: add PostgreSQL service to test workflow` |
| `revert` | 以前のコミットを元に戻す | `revert: revert "feat(auth): add OAuth2 login"` |

### 良い例と悪い例

```
# 悪い例: 曖昧で文脈なし
git commit -m "fixed stuff"
git commit -m "updates"
git commit -m "WIP"

# 良い例: 明確、具体的、理由が説明されている
git commit -m "fix(api): retry requests on 503 Service Unavailable

The external API occasionally returns 503 errors during peak hours.
Added exponential backoff retry logic with max 3 attempts.

Closes #123"
```

### コミットメッセージテンプレート

リポジトリルートに `.gitmessage` を作成します:

```
# <type>(<scope>): <subject>
# # Types: feat, fix, docs, style, refactor, test, chore, perf, ci, revert
# Scope: api, ui, db, auth, etc.
# Subject: imperative mood, no period, max 50 chars
#
# [optional body] - explain why, not what
# [optional footer] - Breaking changes, closes #issue
```

有効化: `git config commit.template .gitmessage`

## マージとリベース

### マージ（履歴を保存）

```bash
# Creates a merge commit
git checkout main
git merge feature/user-auth

# Result:
# *   merge commit
# |\
# | * feature commits
# |/
# * main commits
```

**使用する場合:**
- フィーチャーブランチを `main` にマージするとき
- 正確な履歴を保存したいとき
- 複数の人がブランチで作業したとき
- ブランチがプッシュされ、他の人がそれをベースに作業している可能性があるとき

### リベース（線形履歴）

```bash
# Rewrites feature commits onto target branch
git checkout feature/user-auth
git rebase main

# Result:
# * feature commits (rewritten)
# * main commits
```

**使用する場合:**
- ローカルのフィーチャーブランチを最新の `main` で更新するとき
- 線形でクリーンな履歴を望むとき
- ブランチがローカルのみ（プッシュされていない）のとき
- ブランチで作業しているのが自分だけのとき

### リベースワークフロー

```bash
# Update feature branch with latest main (before PR)
git checkout feature/user-auth
git fetch origin
git rebase origin/main

# Fix any conflicts
# Tests should still pass

# Force push (only if you're the only contributor)
git push --force-with-lease origin feature/user-auth
```

### リベースをしてはいけない場合

```
# 以下のブランチは絶対にリベースしない:
- 共有リポジトリにプッシュされたブランチ
- 他の人がベースにして作業しているブランチ
- 保護されたブランチ（main、develop）
- すでにマージされたブランチ

# 理由: リベースは履歴を書き換え、他の人の作業を壊す
```

## プルリクエストワークフロー

### PRタイトル形式

```
<type>(<scope>): <description>

例:
feat(auth): add SSO support for enterprise users
fix(api): resolve race condition in order processing
docs(api): add OpenAPI specification for v2 endpoints
```

### PR説明テンプレート

```markdown
## What

このPRが何をするかの簡単な説明。

## Why

動機とコンテキストを説明する。

## How

強調する価値のある主要な実装の詳細。

## Testing

- [ ] ユニットテストを追加/更新した
- [ ] 統合テストを追加/更新した
- [ ] 手動テストを実施した

## Screenshots (if applicable)

UIの変更は変更前/後のスクリーンショット。

## Checklist

- [ ] コードはプロジェクトのスタイルガイドラインに従っている
- [ ] セルフレビューが完了した
- [ ] 複雑なロジックにコメントを追加した
- [ ] ドキュメントを更新した
- [ ] 新しい警告が導入されていない
- [ ] ローカルでテストがパスする
- [ ] 関連するイシューがリンクされている

Closes #123
```

### コードレビューチェックリスト

**レビュアー向け:**

- [ ] コードは記述された問題を解決しているか？
- [ ] 処理されていないエッジケースはあるか？
- [ ] コードは読みやすく保守しやすいか？
- [ ] 十分なテストがあるか？
- [ ] セキュリティの懸念はあるか？
- [ ] コミット履歴はクリーンか（必要に応じてスカッシュされているか）？

**作者向け:**

- [ ] レビューを依頼する前にセルフレビューが完了した
- [ ] CI がパスしている（テスト、リント、型チェック）
- [ ] PR のサイズは適切か（理想は500行未満）
- [ ] 単一のフィーチャー/修正に関連している
- [ ] 説明が変更を明確に説明している

## コンフリクト解消

### コンフリクトの識別

```bash
# Check for conflicts before merge
git checkout main
git merge feature/user-auth --no-commit --no-ff

# If conflicts, Git will show:
# CONFLICT (content): Merge conflict in src/auth/login.ts
# Automatic merge failed; fix conflicts and then commit the result.
```

### コンフリクトの解消

```bash
# See conflicted files
git status

# View conflict markers in file
# <<<<<<< HEAD
# content from main
# =======
# content from feature branch
# >>>>>>> feature/user-auth

# Option 1: Manual resolution
# Edit file, remove markers, keep correct content

# Option 2: Use merge tool
git mergetool

# Option 3: Accept one side
git checkout --ours src/auth/login.ts    # Keep main version
git checkout --theirs src/auth/login.ts  # Keep feature version

# After resolving, stage and commit
git add src/auth/login.ts
git commit
```

### コンフリクト防止戦略

```bash
# 1. フィーチャーブランチを小さく短命に保つ
# 2. mainに頻繁にリベースする
git checkout feature/user-auth
git fetch origin
git rebase origin/main

# 3. 共有ファイルを触ることをチームに伝える
# 4. 長命なブランチの代わりにフィーチャーフラグを使う
# 5. PRを速やかにレビューしてマージする
```

## ブランチ管理

### 命名規約

```
# フィーチャーブランチ
feature/user-authentication
feature/JIRA-123-payment-integration

# バグ修正
fix/login-redirect-loop
fix/456-null-pointer-exception

# ホットフィックス（プロダクション問題）
hotfix/critical-security-patch
hotfix/database-connection-leak

# リリース
release/1.2.0
release/2024-01-hotfix

# 実験/POC
experiment/new-caching-strategy
poc/graphql-migration
```

### ブランチのクリーンアップ

```bash
# Delete local branches that are merged
git branch --merged main | grep -v "^\*\|main" | xargs -n 1 git branch -d

# Delete remote-tracking references for deleted remote branches
git fetch -p

# Delete local branch
git branch -d feature/user-auth  # Safe delete (only if merged)
git branch -D feature/user-auth  # Force delete

# Delete remote branch
git push origin --delete feature/user-auth
```

### スタッシュワークフロー

```bash
# Save work in progress
git stash push -m "WIP: user authentication"

# List stashes
git stash list

# Apply most recent stash
git stash pop

# Apply specific stash
git stash apply stash@{2}

# Drop stash
git stash drop stash@{0}
```

## リリース管理

### セマンティックバージョニング

```
MAJOR.MINOR.PATCH

MAJOR: 破壊的変更
MINOR: 後方互換性のある新機能
PATCH: 後方互換性のあるバグ修正

例:
1.0.0 → 1.0.1 (patch: バグ修正)
1.0.1 → 1.1.0 (minor: 新機能)
1.1.0 → 2.0.0 (major: 破壊的変更)
```

### リリースの作成

```bash
# Create annotated tag
git tag -a v1.2.0 -m "Release v1.2.0

Features:
- Add user authentication
- Implement password reset

Fixes:
- Resolve login redirect issue

Breaking Changes:
- None"

# Push tag to remote
git push origin v1.2.0

# List tags
git tag -l

# Delete tag
git tag -d v1.2.0
git push origin --delete v1.2.0
```

### 変更ログの生成

```bash
# Generate changelog from commits
git log v1.1.0..v1.2.0 --oneline --no-merges

# Or use conventional-changelog
npx conventional-changelog -i CHANGELOG.md -s
```

## Git設定

### 必須設定

```bash
# User identity
git config --global user.name "Your Name"
git config --global user.email "your@email.com"

# Default branch name
git config --global init.defaultBranch main

# Pull behavior (rebase instead of merge)
git config --global pull.rebase true

# Push behavior (push current branch only)
git config --global push.default current

# Auto-correct typos
git config --global help.autocorrect 1

# Better diff algorithm
git config --global diff.algorithm histogram

# Color output
git config --global color.ui auto
```

### 便利なエイリアス

```bash
# Add to ~/.gitconfig
[alias]
    co = checkout
    br = branch
    ci = commit
    st = status
    unstage = reset HEAD --
    last = log -1 HEAD
    visual = log --oneline --graph --all
    amend = commit --amend --no-edit
    wip = commit -m "WIP"
    undo = reset --soft HEAD~1
    contributors = shortlog -sn
```

### Gitignoreパターン

```gitignore
# Dependencies
node_modules/
vendor/

# Build outputs
dist/
build/
*.o
*.exe

# Environment files
.env
.env.local
.env.*.local

# IDE
.idea/
.vscode/
*.swp
*.swo

# OS files
.DS_Store
Thumbs.db

# Logs
*.log
logs/

# Test coverage
coverage/

# Cache
.cache/
*.tsbuildinfo
```

## 一般的なワークフロー

### 新しいフィーチャーの開始

```bash
# 1. Update main branch
git checkout main
git pull origin main

# 2. Create feature branch
git checkout -b feature/user-auth

# 3. Make changes and commit
git add .
git commit -m "feat(auth): implement OAuth2 login"

# 4. Push to remote
git push -u origin feature/user-auth

# 5. Create Pull Request on GitHub/GitLab
```

### PRに新しい変更を追加

```bash
# 1. Make additional changes
git add .
git commit -m "feat(auth): add error handling"

# 2. Push updates
git push origin feature/user-auth
```

### フォークをアップストリームと同期

```bash
# 1. Add upstream remote (once)
git remote add upstream https://github.com/original/repo.git

# 2. Fetch upstream
git fetch upstream

# 3. Merge upstream/main into your main
git checkout main
git merge upstream/main

# 4. Push to your fork
git push origin main
```

### 間違いを元に戻す

```bash
# Undo last commit (keep changes)
git reset --soft HEAD~1

# Undo last commit (discard changes)
git reset --hard HEAD~1

# Undo last commit pushed to remote
git revert HEAD
git push origin main

# Undo specific file changes
git checkout HEAD -- path/to/file

# Fix last commit message
git commit --amend -m "New message"

# Add forgotten file to last commit
git add forgotten-file
git commit --amend --no-edit
```

## Gitフック

### Pre-Commitフック

```bash
#!/bin/bash
# .git/hooks/pre-commit

# Run linting
npm run lint || exit 1

# Run tests
npm test || exit 1

# Check for secrets
if git diff --cached | grep -E '(password|api_key|secret)'; then
    echo "Possible secret detected. Commit aborted."
    exit 1
fi
```

### Pre-Pushフック

```bash
#!/bin/bash
# .git/hooks/pre-push

# Run full test suite
npm run test:all || exit 1

# Check for console.log statements
if git diff origin/main | grep -E 'console\.log'; then
    echo "Remove console.log statements before pushing."
    exit 1
fi
```

## アンチパターン

```
# 悪い例: mainに直接コミットする
git checkout main
git commit -m "fix bug"

# 良い例: フィーチャーブランチとPRを使う

# 悪い例: シークレットをコミットする
git add .env  # Contains API keys

# 良い例: .gitignoreに追加し、環境変数を使う

# 悪い例: 巨大なPR（1000行以上）
# 良い例: 小さく集中したPRに分割する

# 悪い例: "Update" コミットメッセージ
git commit -m "update"
git commit -m "fix"

# 良い例: 説明的なメッセージ
git commit -m "fix(auth): resolve redirect loop after login"

# 悪い例: パブリック履歴を書き換える
git push --force origin main

# 良い例: パブリックブランチにはrevertを使う
git revert HEAD

# 悪い例: 長命なフィーチャーブランチ（数週間/数ヶ月）
# 良い例: ブランチを短命（数日）に保ち、頻繁にリベースする

# 悪い例: 生成されたファイルをコミットする
git add dist/
git add node_modules/

# 良い例: .gitignoreに追加する
```

## クイックリファレンス

| タスク | コマンド |
|------|---------|
| ブランチ作成 | `git checkout -b feature/name` |
| ブランチ切り替え | `git checkout branch-name` |
| ブランチ削除 | `git branch -d branch-name` |
| ブランチマージ | `git merge branch-name` |
| ブランチリベース | `git rebase main` |
| 履歴表示 | `git log --oneline --graph` |
| 変更確認 | `git diff` |
| 変更ステージ | `git add .` または `git add -p` |
| コミット | `git commit -m "message"` |
| プッシュ | `git push origin branch-name` |
| プル | `git pull origin branch-name` |
| スタッシュ | `git stash push -m "message"` |
| 直前のコミットを元に戻す | `git reset --soft HEAD~1` |
| コミットをリバート | `git revert HEAD` |
