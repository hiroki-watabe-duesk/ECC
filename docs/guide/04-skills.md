# 第04章 スキル

この章で学ぶこと: ECCのスキル（Skill）が「実行コード」ではなく「構造化知識」である設計思想、`SKILL.md` のフォーマット、配置ポリシー、provenance管理、採用・改変ポリシー、そして良いスキルを書くための実践的な指針を習得します。

---

## スキルとは何か

スキルは **Claude Codeがコンテキストに応じて参照するナレッジモジュール** です。エージェント（タスク実行）でもコマンド（ユーザ起動操作）でもなく、「知識のコンテナ」として機能します。

> **設計思想: スキルは「実行コード」ではなく「構造化知識」である**
>
> スキルは Node.js スクリプトでも関数でもありません。Markdownで書かれた、Claude が読んで理解し即座に活用できる知識の集合体です。コードのように「実行」されるのではなく、Claude のコンテキストに「注入」されます。この設計により、スキルは環境やランタイムに依存せず、プロジェクトをまたいで高い移植性を持ちます。

`docs/SKILL-DEVELOPMENT-GUIDE.md` より:

> Unlike **agents** (specialized subassistants) or **commands** (user-triggered actions), skills are passive knowledge that Claude Code references when relevant.

### スキルが活性化するタイミング

- ユーザのタスクがスキルのドメインにマッチするとき（自動）
- Claude Codeが関連コンテキストを検出したとき（自動）
- コマンドがスキルを参照するとき（明示）
- エージェントがドメイン知識を必要とするとき（エージェントプロンプト経由）

---

## SKILL.md フォーマット

各スキルは `skills/<skill-name>/` ディレクトリに配置され、`SKILL.md` がルートに必須です。

### ディレクトリ構造

```
skills/
└── your-skill-name/
    ├── SKILL.md           # 必須: スキル本体
    ├── examples/          # 任意: コード例
    │   ├── basic.ts
    │   └── advanced.ts
    └── references/        # 任意: 外部参照・補足資料
        └── links.md
```

`references/` サブディレクトリは、本文に含めると長くなりすぎる外部リンク集・仕様書の抜粋・補足説明などを格納します。スキル本体（SKILL.md）からリンクして参照します。

### YAMLフロントマター

```yaml
---
name: skill-name          # 必須: lowercase-with-hyphens 形式の識別子
description: 1行の説明    # 必須: 自動活性化の判断に使われる。ブロックスカラー(|)不可
origin: ECC               # 任意: ECC | community | プロジェクト名
tags: [tag1, tag2]        # 任意: カテゴリタグ
version: "1.0"            # 任意: バージョン管理
---
```

**フロントマターのルール（バリデーター準拠）:**

| フィールド | 必須 | 制約 |
|---|---|---|
| `name` | 必須 | 空不可。`lowercase-with-hyphens` 形式 |
| `description` | 必須 | インラインスカラーのみ。`\|`（リテラルブロックスカラー）は不可 |
| `origin` | 任意 | `ECC`、`community`、プロジェクト名 など |

`description` にブロックスカラー（`|`）を使うと、自動活性化の判断に使うフラットテーブルパーサが壊れます。1行の文字列で記述してください。

### 標準セクション構成

```markdown
---
name: your-skill-name
description: このスキルをいつ使うかを1行で説明
origin: ECC
---

# スキルタイトル

スキルの概要（1〜2文）。

## When to Activate

自動活性化・明示参照のトリガーとなるシナリオを列挙。
ここが具体的であるほど Claude の自動認識精度が上がる。

## Core Principles

主要パターン・設計原則。

## How It Works（または Code Examples）

実際に Claude が即座に使えるコード例・手順・決定木。

## Anti-Patterns

やってはいけないことを具体例で示す。

## Best Practices

アクションableなガイドライン（チェックリスト形式も有効）。

## Related Skills（または See Also）

関連スキルへのリンク。
```

### 実際のスキル例（tdd-workflow）

`skills/tdd-workflow/SKILL.md` のフロントマター:

```yaml
---
name: tdd-workflow
description: Use this skill when writing new features, fixing bugs, or refactoring code. Enforces test-driven development with 80%+ coverage including unit, integration, and E2E tests.
origin: ECC
---
```

- `description` が具体的な起動条件を含んでいる
- 1行のインラインスカラーで記述されている
- カバレッジ目標（80%+）まで含む高密度な説明

---

## 配置ポリシー

スキルは「出荷されるもの」と「ローカル生成・インポートされるもの」で配置場所が厳密に分離されています。

| 種別 | 配置パス | 出荷 | provenance 必須 |
|---|---|---|---|
| **Curated** | `skills/<name>/`（リポジトリ内） | **Yes** | 不要（フロントマターの `origin` で帰属表示） |
| **Learned** | `~/.claude/skills/learned/<name>/` | No | 必須（`.provenance.json`） |
| **Imported** | `~/.claude/skills/imported/<name>/` | No | 必須（`.provenance.json`） |
| **Evolved** | `~/.claude/homunculus/evolved/skills/`（グローバル）または `~/.claude/homunculus/projects/<hash>/evolved/skills/`（プロジェクト別） | No | ソースのinstinctから継承 |

**重要:** インストールマニフェスト（`manifests/install-modules.json`）が参照するのは curated パスのみです。Learned / Imported / Evolved スキルはリポジトリに含まれず、他者に配布・出荷されません。

### Curated スキル

- `skills/<name>/SKILL.md` に配置
- CI（`scripts/ci/validate-skills.js`）で自動検証
- インストール時に配布対象となる

### Learned スキル

- `/learn` コマンドや `evaluate-session` フックが作成
- `~/.claude/skills/learned/<name>/` に配置
- `.provenance.json` が必須（ないと無効）
- デフォルトパスは `skills/continuous-learning/config.json` の `learned_skills_path` で変更可能

### Imported スキル

- 外部ソース（URL・ファイルコピー等）からユーザが手動導入
- `~/.claude/skills/imported/<name>/` に配置
- `.provenance.json` が必須

### Evolved スキル（Continuous Learning v2）

- `instinct-cli evolve` がクラスタリングしたinstinctから生成
- Learned / Imported とは別システム
- provenanceはソースinstinctから継承（`.provenance.json` は不要）

---

## provenance.json の意味とスキーマ

Learned・Imported スキルには `.provenance.json` が必須です。このファイルがスキルの「出所証明」として機能します。

**スキーマ** (`schemas/provenance.schema.json` 準拠):

```json
{
  "source": "https://example.com/some-skill",
  "created_at": "2025-03-01T12:00:00Z",
  "confidence": 0.85,
  "author": "evaluate-session-hook"
}
```

| フィールド | 型 | 説明 |
|---|---|---|
| `source` | string（必須、空不可） | 由来（URL・パス・識別子） |
| `created_at` | string（必須、ISO 8601） | 生成タイムスタンプ |
| `confidence` | number（必須、0〜1） | 信頼度スコア |
| `author` | string（必須、空不可） | 生成者（フック名・ユーザ名等） |

Curated スキルには `.provenance.json` は不要です。代わりにフロントマターの `origin` フィールドで帰属を示します。

---

## 採用・改変ポリシー（skill-adaptation-policy）

外部リポジトリ・プロンプトパック・他ハーネスからスキルのアイデアを取り込む際のポリシーです（`docs/skill-adaptation-policy.md` 準拠）。

### 基本原則

> copy the underlying idea, workflow, or structure — adapt it to ECC's current install surfaces, validation flow, and repo conventions

外部からのアイデアはECCネイティブな表面として作り直します。薄いラッパーにしてはいけません。

### 元の名前を維持するケース

以下をすべて満たす場合のみ元の名前を維持します:

- 直接ポートに近い実装
- 名前が既に記述的かつ中立
- 表面が上流の概念と同様の動作をする
- ECCにより適した名前が存在しない

例: `nestjs-patterns`（フレームワーク名そのものが主題）

### リネームするケース

以下のいずれかで ECC がより広い・狭い・再パッケージ化を行った場合:

- ECCが実質的な新しい動作・構造・ガイダンスを追加した
- 元の名前がベンダー寄り・コミュニティブランド寄りで、ワークフロー指向でない
- 既存のECCサーフェスと重複しスコープ境界を明確化する必要がある

### 依存の優先順位

ECC は最も狭いネイティブサーフェスを優先します:

```
rules/          # 決定論的な制約 → 最優先
skills/         # オンデマンドのワークフロー
MCP             # 長期的なインタラクティブツール境界が正当化される場合
local scripts   # 決定論的なワンショット実行
direct APIs     # 呼び出しが狭くMCPが不要な場合
```

---

## スキルの検証（validate-skills.js）

CI は `scripts/ci/validate-skills.js` で curated スキルを検証します。

**検証ルール:**

| チェック | エラー種別 | 挙動 |
|---|---|---|
| `skills/` が存在しない | — | exit 0（スキルなしとして正常扱い） |
| サブディレクトリに `SKILL.md` がない | 構造エラー（fatal） | exit 1 |
| `SKILL.md` が空 | 構造エラー（fatal） | exit 1 |
| フロントマターに `name` フィールドがない | フロントマター警告 | デフォルトWARN（`--strict` でERROR） |
| `name` フィールドが空 | フロントマター警告 | デフォルトWARN（`--strict` でERROR） |
| `description` がリテラルブロックスカラー（`\|`） | フロントマター警告 | デフォルトWARN（`--strict` でERROR） |

`--strict` フラグまたは環境変数 `CI_STRICT_SKILLS=1` を設定するとフロントマター警告もエラーに昇格します。

**ローカル検証:**

```bash
# 通常検証（WARNは警告のみ）
node scripts/ci/validate-skills.js

# 厳格検証（WARNもエラーとして扱う）
node scripts/ci/validate-skills.js --strict
```

---

## 良いスキルの書き方

### 設計原則

| 良い例 | 悪い例 | 理由 |
|---|---|---|
| `react-hook-patterns` | `react` | スコープが広すぎる |
| `postgresql-indexing` | `databases` | 汎用すぎて活性化条件が曖昧 |
| `nextjs-app-router` | `nextjs` | サブ機能を明示することで精度向上 |

### コンテンツの質

**すぐに使えるコード例を含める（Show, Don't Tell）:**

```markdown
## Anti-Patterns

### 悪い例（曖昧な説明）
非同期関数では常に適切にエラーを処理してください。

### 良い例（具体的なコードで示す）
\`\`\`typescript
async function fetchData(url: string) {
  try {
    const response = await fetch(url);
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${response.statusText}`);
    }
    return await response.json();
  } catch (error) {
    console.error('Fetch failed:', error);
    throw new Error('Failed to fetch data');
  }
}
\`\`\`
```

### アンチパターン

以下はスキルとして避けるべきパターンです:

| アンチパターン | 問題 | 対処 |
|---|---|---|
| `description` にブロックスカラー（`\|`）使用 | バリデーター違反、自動活性化が壊れる | 1行インラインスカラーに修正 |
| `When to Activate` が曖昧 | 自動検出精度が下がる | 具体的なシナリオを列挙 |
| 複数ドメインを1スキルに詰め込む | スコープが広すぎて活性化が不安定 | スキルを分割する |
| コードなし・説明のみ | Claudeが即座に使いにくい | コピペ可能なコード例を追加 |
| 外部ツールへの依存前提 | 移植性低下 | ECC内部で表現できないか検討 |
| APIキー・パス等の秘密情報 | セキュリティリスク | 絶対に含めない |

### チェックリスト（スキル作成時）

```
- [ ] YAML フロントマターが valid（name 必須、description はインラインスカラー）
- [ ] name が lowercase-with-hyphens 形式
- [ ] When to Activate が具体的なシナリオを含む
- [ ] コード例がコピペ可能な実働するコード
- [ ] Anti-Patterns セクションがある
- [ ] Related Skills でリンクしている
- [ ] 秘密情報・個人パスが含まれていない
- [ ] 長さが 200〜800 行程度（過度に長くしない）
```

---

## スキル作成の実践手順

スキルを新たに作成する手順の概要です（詳細は [12-authoring-components.md](./12-authoring-components.md) を参照）。

```bash
# 1. ディレクトリ作成
mkdir -p skills/your-skill-name

# 2. SKILL.md を作成（フロントマター + 標準セクション）
# （テンプレートは docs/SKILL-DEVELOPMENT-GUIDE.md に掲載）

# 3. ローカル検証
node scripts/ci/validate-skills.js

# 4. テスト全体も実行してから PR を出す
node tests/run-all.js
```

---

## 関連章

- [03-components-overview.md](./03-components-overview.md) — 6コンポーネントの比較と選択基準
- [05-agents.md](./05-agents.md) — エージェントとスキルの連携
- [11-quality-ci-testing.md](./11-quality-ci-testing.md) — validate-skills.js の CI 統合
- [12-authoring-components.md](./12-authoring-components.md) — スキル作成の詳細手順
- [13-applying-to-your-project.md](./13-applying-to-your-project.md) — スキルを他プロジェクトへ移植する

## 参照ソース

- [`docs/SKILL-DEVELOPMENT-GUIDE.md`](../SKILL-DEVELOPMENT-GUIDE.md) — スキル開発ガイド
- [`docs/SKILL-PLACEMENT-POLICY.md`](../SKILL-PLACEMENT-POLICY.md) — 配置ポリシー
- [`docs/skill-adaptation-policy.md`](../skill-adaptation-policy.md) — 採用・改変ポリシー
- [`schemas/provenance.schema.json`](../../schemas/provenance.schema.json) — provenanceスキーマ
- [`scripts/ci/validate-skills.js`](../../scripts/ci/validate-skills.js) — バリデーター実装
- [`skills/tdd-workflow/SKILL.md`](../../skills/tdd-workflow/SKILL.md) — スキル実例（ワークフロー型）
- [`skills/api-design/SKILL.md`](../../skills/api-design/SKILL.md) — スキル実例（ドメイン知識型）
- [`skills/react-patterns/SKILL.md`](../../skills/react-patterns/SKILL.md) — スキル実例（フレームワーク型）
- [`skills/coding-standards/SKILL.md`](../../skills/coding-standards/SKILL.md) — スキル実例（標準規約型）
