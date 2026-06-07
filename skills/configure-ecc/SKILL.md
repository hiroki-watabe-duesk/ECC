---
name: configure-ecc
description: Everything Claude Codeのインタラクティブインストーラー — スキルとルールを選択してユーザーレベルまたはプロジェクトレベルのディレクトリにインストールし、パスを検証し、オプションでインストール済みファイルを最適化するガイドを提供します。
origin: ECC
---

# Everything Claude Code（ECC）の設定

Everything Claude Codeプロジェクトのインタラクティブなステップバイステップのインストールウィザードです。`AskUserQuestion` を使用して、スキルとルールの選択的インストールをガイドし、正確性を確認してから最適化を提案します。

## 起動タイミング

- ユーザーが「configure ecc」「install ecc」「setup everything claude code」などと言った場合
- ユーザーがこのプロジェクトからスキルやルールを選択的にインストールしたい場合
- ユーザーが既存のECCインストールを確認または修正したい場合
- ユーザーがインストール済みのスキルやルールをプロジェクト向けに最適化したい場合

## 前提条件

このスキルはClaude Codeからアクセス可能である必要があります。ブートストラップの方法は2つあります：
1. **プラグイン経由**: `/plugin install ecc@ecc` — プラグインがこのスキルを自動的に読み込む
2. **手動**: このスキルのみを `~/.claude/skills/configure-ecc/SKILL.md` にコピーして、「configure ecc」と言って起動する

---

## ステップ0: ECCリポジトリのクローン

インストールの前に、最新のECCソースを `/tmp` にクローンします：

```bash
rm -rf /tmp/everything-claude-code
git clone https://github.com/affaan-m/everything-claude-code.git /tmp/everything-claude-code
```

`ECC_ROOT=/tmp/everything-claude-code` を以降のすべてのコピー操作のソースとして設定します。

クローンが失敗した場合（ネットワーク問題等）、`AskUserQuestion` を使用して既存のECCクローンのローカルパスをユーザーに提供してもらいます。

---

## ステップ1: インストールレベルの選択

`AskUserQuestion` を使用してインストール場所を確認します：

```
Question: "ECCコンポーネントをどこにインストールしますか？"
Options:
  - "ユーザーレベル (~/.claude/)" — "すべてのClaude Codeプロジェクトに適用"
  - "プロジェクトレベル (.claude/)" — "現在のプロジェクトのみに適用"
  - "両方" — "共通/共有アイテムはユーザーレベル、プロジェクト固有のアイテムはプロジェクトレベル"
```

選択を `INSTALL_LEVEL` として保存します。ターゲットディレクトリを設定します：
- ユーザーレベル: `TARGET=~/.claude`
- プロジェクトレベル: `TARGET=.claude`（現在のプロジェクトルートからの相対パス）
- 両方: `TARGET_USER=~/.claude`、`TARGET_PROJECT=.claude`

ターゲットディレクトリが存在しない場合は作成します：
```bash
mkdir -p $TARGET/skills $TARGET/rules
```

---

## ステップ2: スキルの選択とインストール

### 2a: スコープの選択（コア vs ニッチ）

デフォルトは **コア（新規ユーザーへの推奨）** — `.agents/skills/*` と調査優先ワークフロー用の `skills/search-first/` をコピーします。このバンドルには、エンジニアリング、評価、検証、セキュリティ、戦略的コンパクション、フロントエンドデザイン、Anthropicのクロスファンクショナルスキル（記事執筆、コンテンツエンジン、市場調査、フロントエンドスライド）が含まれます。

`AskUserQuestion`（単一選択）を使用します：
```
Question: "コアスキルのみインストールしますか、それともニッチ/フレームワークパックも含めますか？"
Options:
  - "コアのみ（推奨）" — "tdd、e2e、evals、verification、research-first、security、frontend patterns、compacting、クロスファンクショナルAnthropicスキル"
  - "コア + 選択したニッチ" — "コアの後にフレームワーク/ドメイン固有のスキルを追加"
  - "ニッチのみ" — "コアをスキップし、特定のフレームワーク/ドメインスキルをインストール"
Default: コアのみ
```

ユーザーがニッチまたはコア+ニッチを選択した場合、以下のカテゴリ選択に進み、ユーザーが選択したニッチスキルのみを含めます。

### 2b: スキルカテゴリの選択

以下に選択可能な7つのカテゴリグループがあります。続く詳細な確認リストは8つのカテゴリにわたる45のスキルと1つのスタンドアロンテンプレートをカバーします。`multiSelect: true` で `AskUserQuestion` を使用します：

```
Question: "インストールするスキルカテゴリを選択してください"
Options:
  - "フレームワーク & 言語" — "Django、Laravel、Spring Boot、Quarkus、Go、Python、Java、Frontend、Backend patterns"
  - "データベース" — "PostgreSQL、ClickHouse、JPA/Hibernateパターン"
  - "ワークフロー & 品質" — "TDD、verification、learning、security review、compaction"
  - "リサーチ & APIs" — "Deep research、Exa search、Claude APIパターン"
  - "ソーシャル & コンテンツ配信" — "X/Twitter API、content-engineと組み合わせたクロスポスト"
  - "メディア生成" — "VideoDBと組み合わせたfal.aiの画像/動画/音声"
  - "オーケストレーション" — "dmuxマルチエージェントワークフロー"
  - "すべてのスキル" — "利用可能なすべてのスキルをインストール"
```

### 2c: 個別スキルの確認

選択した各カテゴリについて、以下のスキルの完全なリストを表示し、特定のスキルを確認または選択解除するよう求めます。リストが4アイテムを超える場合は、テキストでリストを表示し、「リストされたすべてをインストール」オプションと「その他」（ユーザーが特定の名前を貼り付ける）を使用した `AskUserQuestion` を使用します。

**カテゴリ: フレームワーク & 言語（25スキル）**

| スキル | 説明 |
|-------|-------------|
| `backend-patterns` | Node.js/Express/Next.js向けのバックエンドアーキテクチャ、API設計、サーバーサイドのベストプラクティス |
| `coding-standards` | TypeScript、JavaScript、React、Node.js向けのユニバーサルコーディング標準 |
| `django-patterns` | Djangoアーキテクチャ、DRFを使用したREST API、ORM、キャッシュ、シグナル、ミドルウェア |
| `django-security` | Djangoセキュリティ: 認証、CSRF、SQLインジェクション、XSS防止 |
| `django-tdd` | pytest-django、factory_boy、モッキング、カバレッジを使用したDjangoテスト |
| `django-verification` | Django検証ループ: マイグレーション、リンティング、テスト、セキュリティスキャン |
| `laravel-patterns` | Laravelアーキテクチャパターン: ルーティング、コントローラー、Eloquent、キュー、キャッシュ |
| `laravel-security` | Laravelセキュリティ: 認証、ポリシー、CSRF、マスアサインメント、レート制限 |
| `laravel-tdd` | PHPUnitとPestを使用したLaravelテスト、ファクトリー、フェイク、カバレッジ |
| `laravel-verification` | Laravel検証: リンティング、静的解析、テスト、セキュリティスキャン |
| `frontend-patterns` | React、Next.js、状態管理、パフォーマンス、UIパターン |
| `frontend-slides` | 依存関係ゼロのHTMLプレゼンテーション、スタイルプレビュー、PPTX→Web変換 |
| `golang-patterns` | ロバストなGoアプリケーション向けのイディオマティックGoパターンと規約 |
| `golang-testing` | Goテスト: テーブル駆動テスト、サブテスト、ベンチマーク、ファジング |
| `java-coding-standards` | Spring BootとQuarkus向けのJavaコーディング標準: 命名、不変性、Optional、ストリーム、CDI |
| `python-patterns` | Pythonicなイディオム、PEP 8、型ヒント、ベストプラクティス |
| `python-testing` | pytestを使用したPythonテスト、TDD、フィクスチャ、モッキング、パラメトリゼーション |
| `quarkus-patterns` | Quarkusアーキテクチャ、Camelメッセージング、CDIサービス、Panacheデータアクセス |
| `quarkus-security` | Quarkusセキュリティ: JWT/OIDC、RBAC、入力バリデーション、シークレット管理 |
| `quarkus-tdd` | JUnit 5、Mockito、REST Assured、Camelテストを使用したQuarkus TDD |
| `quarkus-verification` | Quarkus検証: ビルド、静的解析、テスト、ネイティブコンパイル |
| `springboot-patterns` | Spring Bootアーキテクチャ、REST API、レイヤードサービス、キャッシュ、非同期 |
| `springboot-security` | Spring Security: 認証/認可、バリデーション、CSRF、シークレット、レート制限 |
| `springboot-tdd` | JUnit 5、Mockito、MockMvc、TestcontainersによるSpring Boot TDD |
| `springboot-verification` | Spring Boot検証: ビルド、静的解析、テスト、セキュリティスキャン |

**カテゴリ: データベース（3スキル）**

| スキル | 説明 |
|-------|-------------|
| `clickhouse-io` | ClickHouseパターン、クエリ最適化、アナリティクス、データエンジニアリング |
| `jpa-patterns` | JPA/Hibernateエンティティ設計、リレーションシップ、クエリ最適化、トランザクション |
| `postgres-patterns` | PostgreSQLクエリ最適化、スキーマ設計、インデックス、セキュリティ |

**カテゴリ: ワークフロー & 品質（8スキル）**

| スキル | 説明 |
|-------|-------------|
| `continuous-learning` | レガシーv1 Stop-hookセッションパターン抽出; 新規インストールには `continuous-learning-v2` を推奨 |
| `continuous-learning-v2` | 信頼スコアリングを持つ本能ベースの学習、スキル、エージェント、オプションのレガシーコマンドシムに進化 |
| `eval-harness` | 評価駆動開発（EDD）のための正式な評価フレームワーク |
| `iterative-retrieval` | サブエージェントコンテキスト問題のためのプログレッシブコンテキスト精緻化 |
| `security-review` | セキュリティチェックリスト: 認証、入力、シークレット、API、決済機能 |
| `strategic-compact` | 論理的なインターバルで手動コンテキストコンパクションを提案 |
| `tdd-workflow` | 80%以上のカバレッジでTDDを強制: ユニット、インテグレーション、E2E |
| `verification-loop` | 検証と品質ループパターン |

**カテゴリ: ビジネス & コンテンツ（5スキル）**

| スキル | 説明 |
|-------|-------------|
| `article-writing` | ノート、例、またはソースドキュメントを使用した指定ボイスでの長文執筆 |
| `content-engine` | マルチプラットフォームのソーシャルコンテンツ、スクリプト、コンテンツ再利用ワークフロー |
| `market-research` | ソース引用付きの市場、競合他社、ファンド、テクノロジーリサーチ |
| `investor-materials` | ピッチデッキ、ワンページャー、投資家メモ、財務モデル |
| `investor-outreach` | パーソナライズされた投資家向けコールドメール、ウォームイントロ、フォローアップ |

**カテゴリ: リサーチ & APIs（2スキル）**

| スキル | 説明 |
|-------|-------------|
| `deep-research` | firecrawlとexa MCPを使用した引用付きレポートによるマルチソースの深いリサーチ |
| `exa-search` | WebやコードZe、企業、人物リサーチ向けのExa MCPによるニューラル検索 |

`claude-api` はAnthropicの公式スキルです。ECCにバンドルされたコピーの代わりに公式のClaude APIワークフローが必要な場合は [`anthropics/skills`](https://github.com/anthropics/skills) からインストールしてください。

**カテゴリ: ソーシャル & コンテンツ配信（2スキル）**

| スキル | 説明 |
|-------|-------------|
| `x-api` | X/Twitter APIの投稿、スレッド、検索、アナリティクス統合 |
| `crosspost` | プラットフォームネイティブな適応によるマルチプラットフォームコンテンツ配信 |

**カテゴリ: メディア生成（2スキル）**

| スキル | 説明 |
|-------|-------------|
| `fal-ai-media` | fal.ai MCP経由の統合AIメディア生成（画像、動画、音声） |
| `video-editing` | 実際の映像のカット、構成、拡張のためのAI支援動画編集 |

**カテゴリ: オーケストレーション（1スキル）**

| スキル | 説明 |
|-------|-------------|
| `dmux-workflows` | 並列エージェントセッションのためのdmuxを使用したマルチエージェントオーケストレーション |

**スタンドアロン**

| スキル | 説明 |
|-------|-------------|
| `docs/examples/project-guidelines-template.md` | プロジェクト固有スキル作成のためのテンプレート |

### 2d: インストールの実行

選択した各スキルについて、正しいソースルートからスキルディレクトリ全体をコピーします：

```bash
# コアスキルは .agents/skills/ 以下にある
cp -R "$ECC_ROOT/.agents/skills/<skill-name>" "$TARGET/skills/"

# ニッチスキルは skills/ 以下にある
cp -R "$ECC_ROOT/skills/<skill-name>" "$TARGET/skills/"
```

globされたソースディレクトリを反復処理する場合、末尾スラッシュのソースを直接 `cp` に渡さないでください。ディレクトリパスを宛先名として明示的に使用してください：

```bash
cp -R "${src%/}" "$TARGET/skills/$(basename "${src%/}")"
```

注意: `continuous-learning` と `continuous-learning-v2` には追加ファイル（config.json、フック、スクリプト）があります — SKILL.mdだけでなく、ディレクトリ全体がコピーされるようにしてください。

---

## ステップ3: ルールの選択とインストール

`multiSelect: true` で `AskUserQuestion` を使用します：

```
Question: "インストールするルールセットを選択してください"
Options:
  - "共通ルール（推奨）" — "言語に依存しない原則: コーディングスタイル、gitワークフロー、テスト、セキュリティ等（8ファイル）"
  - "TypeScript/JavaScript" — "TS/JSパターン、フック、Playwrightによるテスト（5ファイル）"
  - "Python" — "Pythonパターン、pytest、black/ruffフォーマット（5ファイル）"
  - "Go" — "Goパターン、テーブル駆動テスト、gofmt/staticcheck（5ファイル）"
```

インストールを実行します：
```bash
# 共通ルール
cp -r $ECC_ROOT/rules/common $TARGET/rules/common

# 言語固有ルール（言語ごとのディレクトリを維持）
cp -r $ECC_ROOT/rules/typescript $TARGET/rules/typescript   # 選択した場合
cp -r $ECC_ROOT/rules/python $TARGET/rules/python            # 選択した場合
cp -r $ECC_ROOT/rules/golang $TARGET/rules/golang            # 選択した場合
```

**重要**: ユーザーが言語固有のルールを選択したが共通ルールを選択しなかった場合、警告します：
> 「言語固有のルールは共通ルールを拡張します。共通ルールなしでインストールすると、カバレッジが不完全になる可能性があります。共通ルールもインストールしますか？」

---

## ステップ4: インストール後の検証

インストール後に以下の自動チェックを実行します：

### 4a: ファイルの存在確認

インストールされたすべてのファイルをリストし、ターゲット場所に存在することを確認します：
```bash
ls -la $TARGET/skills/
ls -la $TARGET/rules/
```

### 4b: パス参照のチェック

インストールされたすべての `.md` ファイルのパス参照をスキャンします：
```bash
grep -rn "~/.claude/" $TARGET/skills/ $TARGET/rules/
grep -rn "../common/" $TARGET/rules/
grep -rn "skills/" $TARGET/skills/
```

**プロジェクトレベルのインストールの場合**、`~/.claude/` パスへの参照にフラグを立てます：
- スキルが `~/.claude/settings.json` を参照している場合 — 通常は問題なし（設定は常にユーザーレベル）
- スキルが `~/.claude/skills/` または `~/.claude/rules/` を参照している場合 — プロジェクトレベルのみのインストールでは壊れる可能性あり
- スキルが別のスキルを名前で参照している場合 — 参照されたスキルもインストールされたかを確認

### 4c: スキル間のクロスリファレンスのチェック

一部のスキルは他のスキルを参照します。これらの依存関係を確認します：
- `django-tdd` は `django-patterns` を参照する可能性あり
- `laravel-tdd` は `laravel-patterns` を参照する可能性あり
- `quarkus-tdd` は `quarkus-patterns` を参照する可能性あり
- `springboot-tdd` は `springboot-patterns` を参照する可能性あり
- `continuous-learning-v2` は `~/.claude/homunculus/` ディレクトリを参照
- `python-testing` は `python-patterns` を参照する可能性あり
- `golang-testing` は `golang-patterns` を参照する可能性あり
- `crosspost` は `content-engine` と `x-api` を参照
- `deep-research` は `exa-search` を参照（補完的なMCPツール）
- `fal-ai-media` は `videodb` を参照（補完的なメディアスキル）
- `x-api` は `content-engine` と `crosspost` を参照
- 言語固有のルールは `common/` の対応するものを参照

### 4d: 問題の報告

見つかった各問題について報告します：
1. **ファイル**: 問題のある参照を含むファイル
2. **行**: 行番号
3. **問題**: 何が問題か（例：「~/.claude/skills/python-patternsを参照しているがpython-patternsはインストールされていない」）
4. **推奨される修正**: 何をすべきか（例：「python-patternsスキルをインストール」または「パスを.claude/skills/に更新」）

---

## ステップ5: インストール済みファイルの最適化（オプション）

`AskUserQuestion` を使用します：

```
Question: "インストールされたファイルをプロジェクト向けに最適化しますか？"
Options:
  - "スキルを最適化" — "無関係なセクションを削除し、パスを調整し、テックスタックに合わせてカスタマイズ"
  - "ルールを最適化" — "カバレッジターゲットを調整し、プロジェクト固有のパターンを追加し、ツール設定をカスタマイズ"
  - "両方を最適化" — "インストールされたすべてのファイルを完全に最適化"
  - "スキップ" — "すべてをそのまま保持"
```

### スキルを最適化する場合：
1. インストールされた各SKILL.mdを読み込む
2. ユーザーのプロジェクトのテックスタックを確認する（まだわかっていない場合）
3. 各スキルについて、無関係なセクションの削除を提案する
4. インストールターゲット（ソースリポジトリではない）でSKILL.mdファイルをその場で編集する
5. ステップ4で見つかったパスの問題を修正する

### ルールを最適化する場合：
1. インストールされた各ルール .mdファイルを読み込む
2. ユーザーの設定を確認する：
   - テストカバレッジターゲット（デフォルト80%）
   - 推奨フォーマットツール
   - Gitワークフロー規約
   - セキュリティ要件
3. インストールターゲットのルールファイルをその場で編集する

**重要**: インストールターゲット（`$TARGET/`）内のファイルのみを変更し、ソースECCリポジトリ（`$ECC_ROOT/`）のファイルを変更しないでください。

---

## ステップ6: インストールサマリー

クローンしたリポジトリを `/tmp` からクリーンアップします：

```bash
rm -rf /tmp/everything-claude-code
```

サマリーレポートを表示します：

```
## ECCインストール完了

### インストールターゲット
- レベル: [ユーザーレベル / プロジェクトレベル / 両方]
- パス: [ターゲットパス]

### インストールされたスキル（[数]）
- skill-1, skill-2, skill-3, ...

### インストールされたルール（[数]）
- common（8ファイル）
- typescript（5ファイル）
- ...

### 検証結果
- [数]件の問題が見つかり、[数]件が修正されました
- [残っている問題をリスト]

### 適用された最適化
- [加えた変更をリスト、または「なし」]
```

---

## トラブルシューティング

### 「スキルがClaude Codeに認識されない」
- スキルディレクトリに `SKILL.md` ファイルが含まれているか確認（.mdファイルがバラバラに置かれていないこと）
- ユーザーレベルの場合: `~/.claude/skills/<skill-name>/SKILL.md` が存在するか確認
- プロジェクトレベルの場合: `.claude/skills/<skill-name>/SKILL.md` が存在するか確認

### 「ルールが機能しない」
- ルールはサブディレクトリではなくフラットファイルです: `$TARGET/rules/coding-style.md`（正しい）vs `$TARGET/rules/common/coding-style.md`（フラットインストールでは不正しい）
- ルールをインストールした後はClaude Codeを再起動する

### 「プロジェクトレベルのインストール後のパス参照エラー」
- 一部のスキルは `~/.claude/` パスを想定しています。ステップ4の検証を実行してこれらを見つけて修正してください。
- `continuous-learning-v2` の場合、`~/.claude/homunculus/` ディレクトリは常にユーザーレベルです — これは想定された動作でエラーではありません。
