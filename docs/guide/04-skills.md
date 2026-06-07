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

## Claude Code におけるスキルとは（初心者向け）

> Claude Code の全体的な入門（ハーネスの仕組み・コンポーネント体系）は [03-components-overview.md](./03-components-overview.md) を先に読むことをお勧めします。この節はスキル固有の基礎に絞って説明します。

### スキルファイルの置き場所と自動ロード

Claude Code がスキルを認識する仕組みはシンプルです。`SKILL.md` ファイルを所定のディレクトリに置くと、Claude Code がそのファイルを読み込み候補として管理します。配置場所には大きく2つのパターンがあります。

- **プロジェクト内スキル**: リポジトリの `skills/<name>/SKILL.md`（ECCがこの形式を採用）
- **ユーザーローカルスキル**: `~/.claude/skills/<name>/SKILL.md`（自分専用、配布されない）

ECC プラグインをインストールすると、`skills/` 配下の curated スキルが Claude Code の参照可能な状態になります。

### description による Progressive Disclosure（段階的開示）

スキルが「いつ読み込まれるか」を決めるのが、フロントマターの `description` フィールドです。

```yaml
---
name: tdd-workflow
description: Use this skill when writing new features, fixing bugs, or refactoring code.
---
```

Claude Code はユーザーのタスクとスキルの `description` を照合し、関連性が高いと判断したときだけそのスキルの本文を読み込みます。これを **Progressive Disclosure（段階的開示）** と呼びます。すべてのスキルを常時読み込むのではなく、必要なときだけ必要なスキルを取り込むことで、コンテキストウィンドウを節約します。

言い換えると、`description` はスキルの「目次インデックス」であり、スキル本文は「必要時にだけ開く専門書」です。

### SKILL.md はモデルが読む知識ファイル

`SKILL.md` は Node.js スクリプトでも設定ファイルでもありません。**Claude（LLM）が読んで即座に活用するための、構造化された知識テキスト**です。コードのように「実行」されるのではなく、Claude のコンテキストに「注入」されます。

このため、SKILL.md を書くときは「Claude に何を伝えたいか」を起点に考えます。「どう実行させるか」ではなく「何を知っておいてほしいか」という視点です。

### スキル・コマンド・エージェントの違い（初学者向け早見表）

| コンポーネント | 起動方法 | 目的 | 実体 |
|---|---|---|---|
| **スキル** | 自動（description マッチ）または明示参照 | 知識・パターン・ガイドラインを Claude に注入する | Claude が読む Markdown ファイル |
| **コマンド** | ユーザーが `/コマンド名` と入力 | ユーザー起点の操作・ワークフロー実行 | Claude が実行する Markdown 手順書 |
| **エージェント** | Task ツールによる委譲 | 独立したコンテキストで専門タスクを遂行 | 独自コンテキストを持つサブエージェント |

スキルは「受動的な知識ソース」、コマンドは「能動的なアクション起点」、エージェントは「独立した実行単位」です。

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

## 4種別（Curated / Learned / Imported / Evolved）の役割と違い

前節の表は「どこに置かれ、出荷されるか」を示しました。この節では同じ4種別を「**誰が・何のために作り、どの知識から生まれ、どこへ向かうか**」という役割の観点から整理し、本質的な違いを明確にします。

### 多次元比較表

| 観点 | Curated | Learned | Imported | Evolved |
|---|---|---|---|---|
| **役割（一言で）** | 全員に出荷する「正典」スキル | 自分のセッション経験から抽出した学び | 外部から取り込んだ他者の成果 | 蓄積instinctを統合・蒸留した進化形 |
| **生成主体** | 人間（コントリビュータ） | 継続学習（`/learn`・`/learn-eval`・`evaluate-session` フック） | ユーザ（手動・外部由来） | `instinct-cli evolve`（自動） |
| **知識の出所** | 設計・キュレーション | 自分の実作業の振り返り | URL・ファイルコピー等の外部ソース | クラスタ化された複数のinstinct |
| **所属システム** | 通常のスキルロード | 継続学習（continuous-learning） | 通常のスキルロード（手動配置） | 継続学習 v2 / homunculus（別系統） |
| **配置** | `skills/`（リポジトリ） | `~/.claude/skills/learned/`（または プロジェクト `.claude/skills/learned/`） | `~/.claude/skills/imported/` | `~/.claude/homunculus/.../evolved/skills/` |
| **出荷・配布** | される（全員に） | されない（自分専用） | されない（自分専用） | されない（自分専用） |
| **provenance** | 不要（`origin` で帰属表示） | 必須（`.provenance.json`） | 必須（`.provenance.json`） | instinctから継承（個別ファイル不要） |
| **CI検証** | `validate-skills.js` の対象 | 対象外（存在時にランタイムでロード） | 対象外 | 対象外 |
| **主な利用シーン** | 安定した再利用パターンを全員へ展開 | 「今回学んだこと」を次回へ持ち越す | 外部の良い手順・プロンプトを試す | 繰り返し現れる癖を自動的に定着させる |

### 種別ごとの役割

**Curated（キュレーション済み）— 「出荷される正典」**
リポジトリ `skills/` に置かれ、インストールマニフェスト（`manifests/install-modules.json`）に載って**全ユーザへ配布される唯一の種別**です。人間が設計・レビューし、CI（`validate-skills.js`）で検証された安定的な再利用ワークフローで、チーム/コミュニティ共通の「source of truth」として機能します。出所は `.provenance.json` ではなくフロントマターの `origin`（`ECC` / `community`）で示します。

**Learned（学習済み）— 「自分の経験の記録」**
セッション中に非自明な解法・回避策・プロジェクト固有のパターンを解決したとき、継続学習システム（`/learn`・`/learn-eval` コマンドや `evaluate-session` フック）が新しいスキルとして書き出します。次に類似の問題が来たときにロードされ、同じ学びを繰り返さずに済みます。**自分のローカル環境専用**でリポジトリには入りません。`/learn-eval` は保存先を「Global（`~/.claude/skills/learned/`、複数プロジェクトで有用）」と「Project（`.claude/skills/learned/`、特定プロジェクト固有）」に振り分ける品質ゲートを備えます。出所証明として `.provenance.json` が必須です。

**Imported（インポート済み）— 「外部成果の取り込み」**
他リポジトリ・プロンプトパック・ブログ等、**外部ソース由来**のスキルをユーザが手動で持ち込んだものです（現時点で自動インポーターは無く、配置は規約ベース）。Learned が「自分の経験」なのに対し、Imported は「他者の成果」である点が本質的な違いです。これも自分専用で出荷されず、`.provenance.json`（`source` に取得元URL等）が必須です。

**Evolved（進化済み）— 「instinctの統合体」**
継続学習 v2（homunculus）という**別系統のシステム**が、蓄積された多数の instinct（行動の癖・好み）をクラスタリングし、`instinct-cli evolve` で1つのスキルへ**統合・蒸留**したものです。Learned が「単発の学び」なら、Evolved は「繰り返し現れたパターンの集約」です。provenance は元になった instinct から継承するため、個別の `.provenance.json` は不要です。グローバル版とプロジェクト別版があります。

### 知識のライフサイクル（種別間の関係）

4種別は無関係に並ぶのではなく、「**実験的・個人的な学び → 検証 → 安定した正典へ昇格**」という流れの中の段階として捉えると理解しやすくなります。

```
 ローカル専用（ユーザ home に置かれ、出荷されない）
 ┌──────────────────────────────────────────────────┐
 │  Imported          Learned            Evolved      │
 │ （外部由来）     （自分の経験）     （instinct統合）│
 │     │           /learn               instinct-cli  │
 │     │           /learn-eval             evolve      │
 │     │              │                      │         │
 │     │       continuous-learning    continuous-      │
 │     │            (v1)              learning v2       │
 │     └──────────────┴──────────────────────┘         │
 └────────────────────────┬───────────────────────────┘
                          │  価値が広く認められたパターンを
                          │  人手でキュレーション（PR で skills/ に追加）
                          ▼
                Curated（リポジトリ同梱・全員へ出荷）
                          │  install で各ハーネスへ配布
                          ▼
              Claude Code / Codex / Cursor / OpenCode ...
```

- **下流（Imported / Learned / Evolved）**: ユーザ home に置かれる**ローカル専用・実験的**な層。気軽に増やし、合わなければ捨てられます。
- **上流（Curated）**: 検証を経て**永続化・出荷**される層。ローカルで価値が確認されたパターンは、最終的に**人手でキュレーションして `skills/` に追加（PR）**することで curated 化されます。「learned → curated」を自動で行うスラッシュコマンドは存在せず、curated 化は通常のコントリビューションフローを通ります（[12-authoring-components.md](./12-authoring-components.md) 参照）。
- **`/promote` の位置づけ（混同注意）**: `/promote` は「learned スキルを curated に昇格させる」コマンド**ではありません**。継続学習 v2 において **instinct を「プロジェクトスコープ → グローバルスコープ」へ昇格**させるコマンドで、Evolved スキルを生む内部メカニズム側の操作です。

> **本質的な違いの要約**
> - **Curated** = 唯一の「出荷される」種別。人手で設計・検証された正典。
> - **Learned** = 「自分の経験」から抽出された個人的な学び（Global / Project スコープあり）。
> - **Imported** = 「他者の成果」を手動で取り込んだもの。
> - **Evolved** = 「繰り返す癖（instinct）」を統合・蒸留した進化形（別系統の v2 システム）。
> - 出荷されるのは **Curated だけ**。残り3つはユーザ home に置かれるローカル専用で、provenance（または instinct からの継承）で出所を管理します。

---

## Learned / Imported / Evolved の生成の仕組みと流れ

「この3種別は誰が・どうやって作るのか？ プログラムなのか専用エージェントなのか？」——答えは種別ごとにまったく異なります。結論を先に示すと、**Learned は Claude（モデル）自身が書き、Imported は人間が手で置き、Evolved は専用プログラム `instinct-cli.py` が機械的に生成します**（Evolved だけは前段に専用のバックグラウンド・エージェントが介在します）。

### 生成者の早見表

| 種別 | 実際に SKILL.md を書くのは誰か | きっかけ（トリガー） | 中核の実装 | 出力先 |
|---|---|---|---|---|
| **Learned** | **Claude（メインのモデル）** | Stopフックの合図、または `/learn`・`/learn-eval` | `scripts/hooks/evaluate-session.js`（合図のみ）/ `commands/learn*.md`（抽出手順） | `~/.claude/skills/learned/`（または プロジェクト `.claude/skills/learned/`） |
| **Imported** | **人間（手動）** | ユーザが外部から取得し配置 | 自動インポーター無し（規約ベースの手作業） | `~/.claude/skills/imported/` |
| **Evolved** | **プログラム（`instinct-cli.py`）** | `instinct-cli.py evolve --generate`（`/evolve`） | `skills/continuous-learning-v2/`（観測フック＋Haiku観測エージェント＋CLI） | `<homunculus>/evolved/skills/`（後述の注記参照） |

### Learned — Claude 自身が書く（フックは「合図」するだけ）

最も誤解されやすい点を先に強調します。**フックはスキルを書きません。** フックは Claude に「このセッションを学びとして抽出して保存して」と促す**合図（シグナル）を出すだけ**で、実際の `SKILL.md` を起草・保存するのは Claude（メインのモデル）です。専用の生成プログラムも専用エージェントも存在しません。

**自動フロー（セッション終了時）:**

```
セッション終了
  └─ Stopフック「stop:evaluate-session」（scripts/hooks/evaluate-session.js）
       ├─ stdin の JSON から transcript_path を読む
       ├─ user メッセージ数を数える（min_session_length=10 未満なら何もせず exit 0）
       └─ stderr に合図を出力:
          「N件のメッセージあり。抽出対象を評価し ~/.claude/skills/learned/ に保存せよ」
              │
              ▼
          この合図を受けた Claude が、セッションからパターンを抽出し
          SKILL.md を起草してユーザ確認のうえ保存する
```

**手動フロー（セッション途中）:**

- `/learn`（`commands/learn.md`）: セッションを振り返り、再利用価値のあるパターンを1つ選び、`~/.claude/skills/learned/<name>.md` を起草 → ユーザ確認 → 保存。
- `/learn-eval`（`commands/learn-eval.md`）: `/learn` に「品質ゲート」と「保存先の判定（Global `~/.claude/skills/learned/` か Project `.claude/skills/learned/` か）」を追加したもの。
- 設定は `skills/continuous-learning/config.json`（`min_session_length` / `learned_skills_path` / `patterns_to_detect` 等）。出所証明として `.provenance.json` を併せて書くのが規約です。

> つまり Learned の生成者は**メインの Claude 本体**であり、フックや `/learn` はその引き金にすぎません。フックの仕組み自体は [07-hooks-and-runtime.md](./07-hooks-and-runtime.md) を参照してください。

### Imported — 人間が手で置く（自動インポーターは無い）

スキル用の自動インポーターは**存在しません**（`docs/SKILL-PLACEMENT-POLICY.md` にも "No automated importer exists yet; placement is by convention." と明記）。生成者は**人間**です。

```
① 外部（他リポジトリ・URL・配布物）から SKILL.md を入手
② ~/.claude/skills/imported/<name>/ に置く
③ .provenance.json（source に取得元URL/パス 等）を手で作成する
```

> **混同注意**: `/instinct-import`（`instinct-cli.py import`）は、スキルではなく **instinct（原子的な学習単位）** を取り込むコマンドで、保存先は `instincts/inherited/` です。Imported「スキル」とはまったく別物なので混同しないでください。

### Evolved — 専用プログラムが機械生成（前段に専用エージェント）

Evolved は continuous-learning v2（homunculus）による**4段のパイプライン**で作られます。Learned/Imported と違い、**Claude が本文を書くのではなく、`instinct-cli.py` が蓄積した instinct を集約して機械的に `SKILL.md` を書き出します**。その前段、「instinct を作る」工程に専用のバックグラウンド・エージェント（Haiku）が介在します。

```
① 観測（フック＋プログラム）
   PreToolUse/PostToolUse フック「pre/post:observe:continuous-learning」
   → scripts/hooks/observe-runner.js → skills/continuous-learning-v2/hooks/observe.sh
   → ツール使用イベントを秘密情報を伏せて observations.jsonl に追記（プロジェクト別）
   → 観測が貯まると観測エージェントを遅延起動し、N件(既定20)ごとに SIGUSR1 で通知

② instinct 生成（専用バックグラウンド・エージェント＝Haiku）
   observer エージェント（skills/continuous-learning-v2/agents/observer.md, model: haiku）
   → observations.jsonl を解析（ユーザ訂正／エラー解決／反復ワークフロー／ツール選好）
   → 信頼度付きの instinct（小さな YAML+MD）を instincts/personal/ に作成・更新
     （信頼度: 観測頻度で 0.3→0.85、確認 +0.05／矛盾 -0.1／週次減衰 -0.02、scope=project/global）

③ 昇格（プログラム・任意）
   instinct-cli.py promote（/promote）
   → 同じ instinct が「2プロジェクト以上」かつ「信頼度≥0.8」なら project→global へ昇格

④ スキル化（プログラム）★ここで Evolved スキルが生まれる
   instinct-cli.py evolve（/evolve）
   → instinct をドメイン／トリガーでクラスタリング（既定は分析結果の表示のみ）
   → `--generate` 指定時、_generate_evolved() が上位クラスタから
      evolved/skills/<name>/SKILL.md を機械的に書き出す
      （同時に evolved/commands/*.md・evolved/agents/*.md も生成）
```

- **生成者**は専用プログラム `instinct-cli.py`（の `_generate_evolved()`）。その材料となる instinct を作るのが**専用の Haiku 観測エージェント**です。SKILL.md の中身はクラスタ内 instinct の Action を機械的に連結して組み立てられます（Claude が自由記述するわけではありません）。
- **provenance** は元の instinct から継承するため、個別の `.provenance.json` は不要です。
- エージェントの一般論は [05-agents.md](./05-agents.md)、フックの一般論は [07-hooks-and-runtime.md](./07-hooks-and-runtime.md) を参照。

> **注記（保存先パスの差異）**: 本章の配置ポリシー表と `docs/SKILL-PLACEMENT-POLICY.md` は Evolved の保存先を `~/.claude/homunculus/evolved/skills/` と記載していますが、continuous-learning **v2.1** の実装（`instinct-cli.py` の `_resolve_homunculus_dir()`）は `${XDG_DATA_HOME:-~/.local/share}/ecc-homunculus/evolved/skills/`（プロジェクト別は `.../projects/<hash>/evolved/skills/`）を使います。`CLV2_HOMUNCULUS_DIR` または `XDG_DATA_HOME` で変更可能です。概念は同じ（ユーザ home 配下のローカル専用領域）ですが、正確な物理パスは実装側が優先されます。

### まとめ: 「誰が生成者か」

- **Learned** → **Claude 本体**が書く（フック/コマンドは引き金。フック自体は書かない）。
- **Imported** → **人間**が手動で置く（自動化なし。`/instinct-import` は instinct 用で別物）。
- **Evolved** → **専用プログラム `instinct-cli.py`** が機械生成（前段で Haiku 観測エージェントが instinct を作る）。

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

## スキル設計の思想とベストプラクティス

### なぜ「コードでなく知識」なのか

スキルを「実行コード」ではなく「モデルが読む知識」として設計した理由には、根本的な思想があります。

コードとして実装すると、ランタイム・言語バージョン・依存パッケージに縛られます。一方、Markdown で書かれた知識は **ランタイムに依存せず、Claude Code・Codex・Cursor など異なるハーネスにそのまま移植できます**。スキルがハーネス非依存である理由はここにあります。

また、LLM は「コードを実行するエンジン」である前に「テキストを理解するモデル」です。Markdown で書かれた知識・パターン・例示を読むことで、Claude はコードを生成・判断する際にその知識を自然に活用します。コードを直接実行させるより、知識として注入するほうが汎用性・安全性ともに高くなります。

### description を活かした Progressive Disclosure の設計

`description` はスキルの活性化ゲートです。書き方次第で「正しいときに呼ばれるスキル」になるかどうかが決まります。

**良い description の3条件:**

1. **具体的なトリガーを含む** — 「Use when X, Y, Z」のように起動条件を明示する
2. **1行のインラインスカラー** — ブロックスカラー（`|`）は使わない（バリデーター違反）
3. **ドメインを絞り込む** — `react-hook-patterns` のように対象を限定し、誤爆を防ぐ

```yaml
# 良い例: 起動条件が具体的
description: Use this skill when writing new features, fixing bugs, or refactoring code. Enforces TDD with 80%+ coverage.

# 悪い例: 曖昧すぎて誤爆・不活性化が起きる
description: |
  This skill helps with coding.
```

Progressive Disclosure の本質は「必要な知識を、必要なときだけ、必要な量だけ」注入することです。description を絞り込むほど、Claude のコンテキストが無駄な情報で埋まらなくなります。

### 単一責務・適切なスコープ

1つのスキルが担う知識ドメインは1つに限定します。複数ドメインを詰め込むと、description が曖昧になり活性化の精度が落ちます。また、スキル同士を組み合わせるほうが、1つに詰め込むより柔軟性が高まります。

| スキル例 | スコープ | 判断理由 |
|---|---|---|
| `react-hook-patterns` | 適切 | React の Hook に特化している |
| `react` | 広すぎ | React 全般では description が曖昧になる |
| `coding` | 広すぎ | どんなタスクでも起動してしまい意味を失う |
| `nextjs-app-router-cache` | 適切 | Next.js の App Router キャッシュに特化 |

### コード例を含める理由（Show, Don't Tell）

Claude は抽象的な説明よりも、**コピーペーストできる具体的なコード例**から多くを学びます。「エラーは適切に処理してください」より、`try/catch` の実装例を載せるほうが、Claude が即座に正しいコードを生成できます。

コード例はスキルの「活用可能な知識密度」を高める最も効果的な手段です。Anti-Patterns セクションで「悪い例 → 良い例」の対比を示すと、誤用を防ぐ効果も得られます。

### 移植性を保つ設計（ハーネス非依存）

スキルが特定のハーネス実装に依存すると、Claude Code 以外での再利用が困難になります。以下の設計原則で移植性を確保します。

- **外部ツールへの依存を最小化する** — 特定の CLI コマンドや環境変数を前提にしない
- **ECC 内部で表現できることはスキルに書く** — MCP や外部 API を前提にしない
- **パス・APIキー・個人情報を含めない** — 環境固有の情報はスキルの外に置く
- **標準セクション構成（When to Activate / Core Principles / How It Works / Anti-Patterns / Best Practices）に従う** — 構造が統一されているほど他のハーネスでも読みやすい

### ECC がこれをどう体現しているか

ECC のスキル設計は上記の思想を具体的な制度として実装しています。

**Origin とティア分離**: スキルは `origin` フィールドで帰属を明示し（`ECC` / `community`）、配置ティア（Curated / Learned / Imported / Evolved）で「出荷されるもの」と「個人専用のもの」を厳密に分けます。全員が使う curated スキルは人手でレビューされ CI で検証されます。

**採用・改変ポリシー（skill-adaptation-policy）**: 外部スキルのアイデアを取り込む際は「薄いラッパーにせず ECC ネイティブとして作り直す」原則を持ちます。名前のリネームポリシーも明文化されており、ベンダー寄りの名前をワークフロー指向の名前に改めます。

**CI 検証（validate-skills.js）**: `name` の形式、`description` のスカラー型、`SKILL.md` の存在を自動検証します。品質ゲートを CI に組み込むことで、description が壊れた状態でマージされることを防ぎます。

**配置ポリシーの文書化**: `docs/SKILL-PLACEMENT-POLICY.md` に配置規則を明文化し、Learned / Imported スキルには `.provenance.json` を必須化することで、知識の出所を常に追跡可能にします。

これらの制度は、ECC 固有の仕組みであると同時に、他のプロジェクトでスキルを設計する際にも参考にできる普遍的なベストプラクティスです。

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
