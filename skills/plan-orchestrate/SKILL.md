---
name: plan-orchestrate
description: 計画ドキュメントを読み込み、ステップに分解し、ECCカタログからステップごとのエージェントチェーンを設計し、すぐに貼り付け可能な /orchestrate カスタムプロンプトを出力する。生成のみ — /orchestrate 自体は実行しない。複数ステップの計画を手動でチェーン構成せずに orchestrate で進めたいときに使用する。
origin: ECC
---

# Plan Orchestrate

計画ドキュメントを`/orchestrate custom`に橋渡しし、ステップごとに貼り付け可能な呼び出しを1つ出力します。スキルは生成のみ — `/orchestrate`を実行しません。ユーザーは準備ができたら各行を貼り付けます。

## アクティブにするタイミング

- ユーザーが複数ステップの計画ドキュメント（PRD、RFC、実装計画）を持ち、`/orchestrate`で進めたいとき。
- ユーザーが「この計画をオーケストレートして」「各ステップのorchestrateプロンプトをください」「この計画のチェーンを構成して」と言うとき。
- ステップバイステップの計画は存在するが、ユーザーがステップごとにエージェントを手動で選択したくないとき。

スキップするとき:
- 作業が1つのアドホックなステップの場合 → `/orchestrate custom`を直接呼び出す。
- 計画が読めないまたは空の場合。明示的な番号付けがないだけではスキップ条件にならない — 以下の「ステップが明確でない」エッジケースを参照。

## 入力

```
<plan-doc-path> [--lang=python|typescript|go|rust|cpp|java|kotlin|flutter|auto] [--scope=all|step:<n>|range:<a>-<b>] [--dry-run]
```

- `<plan-doc-path>` — 必須。相対または絶対パス（`@docs/...`も可）。
- `--lang` — レビュアーの言語バリアント。デフォルトは`auto`（プロジェクトから検出）。
- `--scope` — 出力するステップを限定。デフォルトは`all`。
- `--dry-run` — 分解とチェーン根拠のみ出力。最終プロンプトは出力しない。

## 権威ある`/orchestrate`の形式（逸脱禁止）

```
{ORCH_CMD} custom "<agent1>,<agent2>,...,<agentN>" "<task description>"
```

`{ORCH_CMD}`はフェーズ0で決定される（以下参照）。出力で生成されるコマンド文字列は**常に1つの具体的な形式を使用する** — 両方、またはプレースホルダーは使わない。

- `custom`は順次チェーン。各エージェントのHANDOFFが次に渡される。
- カンマ区切りのエージェントリスト。スペースなしが望ましい。スペース1つは許容。
- `--mode` / `--gate` / `--agents=...`フラグは存在しない — 決して発明しない。
- エージェント名はこのスキルのカタログから取る。タスク説明内の埋め込みダブルクォートは`\"`としてエスケープする。

## ECCインストール形式と名前空間

2つのインストール形式が、スラッシュコマンドと全エージェント名の両方のプレフィックスを決定する。2つは必ず同期する — 出力は1つの形式のみ、決して混在しない:

`<claude-home>`はClaude Codeのホームディレクトリを示す: macOS/Linuxでは`~/.claude`、Windowsでは`%USERPROFILE%\.claude`。ホームディレクトリをホストプラットフォームの方法で解決する（`~`をハードコードしない）。

| 形式 | 検出方法 | `{ORCH_CMD}` | エージェント名の形式 |
|---|---|---|---|
| プラグインインストール（1.9.0+） | `<claude-home>/plugins/marketplaces/everything-claude-code/`が存在する | `/everything-claude-code:orchestrate` | `everything-claude-code:<name>` |
| レガシーベアインストール | 上記が存在しない。`<claude-home>/agents/`にエージェントファイルがある | `/orchestrate` | `<name>` |

なぜ重要か: プラグインインストールでは、エージェントは`everything-claude-code:tdd-guide`として登録される。ベア名は並列呼び出しで断続的に失敗するファジーマッチングを強制する。レガシーでは、プレフィックス付き形式は登録されておらず完全に失敗する。

## 利用可能なエージェントカタログ（これらから選ぶこと）

汎用:
- `planner` — 要件の再記述、リスク分解、ステップ計画
- `architect` — アーキテクチャ、システム設計、リファクタリング提案
- `tdd-guide` — テスト作成 → 実装 → カバレッジ80%以上
- `code-reviewer` — 汎用コードレビュー
- `security-reviewer` — セキュリティ監査、OWASP、シークレット漏洩
- `refactor-cleaner` — デッドコード、重複、knipクラスのクリーンアップ
- `doc-updater` — ドキュメント、コードマップ、README
- `docs-lookup` — サードパーティライブラリAPIのルックアップ（Context7）
- `e2e-runner` — エンドツーエンドテストのオーケストレーション
- `database-reviewer` — PostgreSQLスキーマ、マイグレーション、パフォーマンス
- `harness-optimizer` — ローカルエージェントハーネス設定
- `loop-operator` — 長時間実行の自律ループ
- `chief-of-staff` — マルチチャンネルトリアージ（計画ステップにはほとんど適さない）

ビルドエラー解決者:
- `build-error-resolver`（汎用）/ `cpp-build-resolver` / `go-build-resolver` / `java-build-resolver` / `kotlin-build-resolver` / `rust-build-resolver` / `pytorch-build-resolver`

コードレビュアー:
- `python-reviewer` / `typescript-reviewer` / `go-reviewer` / `rust-reviewer` / `cpp-reviewer` / `java-reviewer` / `kotlin-reviewer` / `flutter-reviewer`

エージェント名のスペルミスは`/orchestrate`を失敗させる。出力前にこのリストと照合すること。

## 仕組み

### フェーズ0 — ECCモード + 言語の検出

1. `<plan-doc-path>`を読み込む。見つからないまたは空の場合、報告して停止。
2. ECCインストール形式を一度検出し`ECC_MODE`に固定する。アルゴリズム（順番に実行し、最初に一致したところで停止）:
   1. `<claude-home>/plugins/marketplaces/everything-claude-code/`が存在する → `ECC_MODE=plugin`。
   2. そうでなければ`<claude-home>/agents/`が存在し、少なくとも1つのECCエージェントファイル（例: `tdd-guide.md`、`code-reviewer.md`）を含む → `ECC_MODE=legacy`。
   3. そうでなければ → `ECC_MODE=legacy`をデフォルトとし、出力の先頭に1行の警告を出力: `> Warning: could not detect ECC install; defaulting to legacy form. If you use the plugin install, edit the prefixes manually.`
   4. 両方のマーカーが存在する（混在インストール）場合、`plugin`が優先 — プラグイン名前空間のみがファジーマッチングなしでエージェント名を解決する唯一の方法。

   この時点から、出力する全行は対応するプレフィックスを**スラッシュコマンドと全エージェント名の両方に**使用する。**同じ出力内で両方の形式を出力しない。**
3. `--lang`を解決する。`auto`の場合、多言語対応の検出を実行:
   - プローブマーカー: `pyproject.toml` / `uv.lock` / `requirements.txt` → python; `package.json` → typescript; `go.mod` → go; `Cargo.toml` → rust; `CMakeLists.txt`またはトップレベルの`*.cpp` → cpp; `pom.xml` / `build.gradle`（Java）→ java; `build.gradle.kts`またはトップレベルのKotlin → kotlin; `pubspec.yaml` → flutter。
   - **多言語タイブレーク**: 複数のマーカーが一致する場合、ソースファイル数が他を上回る言語を選ぶ（`git ls-files`でカウント、`vendor/`、`node_modules/`、`dist/`、`build/`、`.venv/`、生成ファイル、明らかなテストフィクスチャを除く）。同数または単一言語がソースファイルの60%を超えない場合、`lang=unknown`を設定。
   - マーカーが一致しない → `lang=unknown`を設定。
   - `lang=unknown`はセンチネル — エージェント名ではない。フェーズ2のルール4と5がチェーン構成時に`code-reviewer` / `build-error-resolver`に変換する。
4. **PyTorchサブプロファイルを検出**: `lang=python`かつ`pyproject.toml` / `requirements.txt` / `uv.lock`のいずれかが`torch`への依存関係を宣言している場合、`pytorch=true`を設定。これは`build`チェーンの選択にのみ影響する（フェーズ2のルール参照）。レビュアーは`python-reviewer`のまま。
5. **計画内で宣言されたエージェント名を正規化する**: 計画テキストがプラグインプレフィックス形式でエージェントを参照している場合（例: `everything-claude-code:tdd-guide`）、プレフィックスを取り除いてベアカタログ名を取得してからバリデーションまたはチェーン構成を行う。プレフィックスの再付与はECC_MODEに従って出力時（フェーズ4）にのみ行う。プレフィックス付きの名前がチェーン構成に流れ込まないようにする — プラグインモードでダブルプレフィックスになってしまう。

### フェーズ1 — ステップの分解

優先順位順に「ステップ単位」を特定:

1. 明示的な番号付け: `## Step N` / `### Phase N` / `## N. ...` / トップレベルの順序付きリスト。
2. テーブルの「Step」列。
3. 動詞主導の見出しを持つ`---`区切りのブロック。
4. それ以外は各H2を1つのステップとして扱う。

各ステップから`id`（1ベース）、`title`（80文字以下）、`intent`（1〜3文）、`tags`を抽出する。

### フェーズ2 — タグ付けとチェーン選択

意図によるタグ付け（複数タグ可。チェーンはプライマリ + スタックされたセカンダリから構築）:

以下のトリガーワードは大文字小文字を区別せずマッチングする。多言語の計画は、意味が列挙された英語のトリガーワードと一致する限り、任意の言語の語幹にマッチングすることでサポートされる。

| タグ | トリガーワード | デフォルトチェーン |
|---|---|---|
| `design` | architecture, design, choose, evaluate, RFC | `planner,architect` |
| `plan` | plan, breakdown, milestone | `planner` |
| `impl` | implement, build, add, create, port | `tdd-guide,<lang>-reviewer` |
| `test` | test, coverage, e2e, integration | `tdd-guide,e2e-runner` |
| `refactor` | refactor, cleanup, dedupe, split | `architect,refactor-cleaner,<lang>-reviewer` |
| `migration` | migrate, upgrade, rewrite, port | `architect,tdd-guide,<lang>-reviewer` |
| `db` | schema, migration, index, SQL, Postgres, alembic, sqlmodel | `database-reviewer,<lang>-reviewer` |
| `security` | encrypt, auth, secret, OWASP, PII | `security-reviewer,<lang>-reviewer` |
| `build` | build, compile, lint failure, CI | `<lang>-build-resolver`（`build-error-resolver`にフォールバック） |
| `docs` | docs, readme, codemap, changelog | `doc-updater` |
| `lookup` | lookup, reference, API usage | `docs-lookup` |
| `review` | review, audit, verify | `<lang>-reviewer,code-reviewer` |
| `loop` | loop, autonomous, watchdog | `loop-operator` |

チェーン構成ルール:
1. **プライマリタグの選択**: ステップが複数のタグにマッチする場合、**テーブル順で最初のもの**（テーブルの上 = 最高優先度）がプライマリ。残りはセカンダリ。以下の構成ルール2と3は特定の複数タグの組み合わせを明示的に処理する。それ以外は、セカンダリチェーンをタグテーブルの順序で追加する。
2. `impl` + `security` → `tdd-guide,<lang>-reviewer,security-reviewer`。
3. `impl` + `db` → `tdd-guide,database-reviewer,<lang>-reviewer`。
4. 結果のチェーンを**重複排除**する（最初の出現を保持）。例: `review` + `lang=unknown`はルール5の後`code-reviewer,code-reviewer`になるが、重複排除で`code-reviewer`に折りたたまれる。
5. `<lang>-reviewer`は`lang=unknown`の場合`code-reviewer`に解決される。
6. `<lang>-build-resolver`は`lang=unknown`の場合`build-error-resolver`に解決される。**特殊ケース**: フェーズ0で`pytorch=true`が設定されている場合、`<lang>`に関係なく`build`チェーンには`pytorch-build-resolver`を使用する。`python-build-resolver`は存在しない。`pytorch=true`なしの`--lang=python`は`build-error-resolver`に解決される。
7. **ゼロタグのステップ**: トリガーワードがマッチしない場合、チェーンを`code-reviewer`に設定し「Chain rationale」に`no tag matched; default review-only chain`と記述する。
8. 重複排除後のチェーン長は4以下。超過した場合、最も弱いタグを削除する（`lookup`と`docs`を最初に）。
9. `impl`チェーンに`planner`と`architect`をペアにしない（トークンの無駄）。`design`ステップでのみペアにする。
10. `impl`、`refactor`、または`migration`タグのステップは**レビュアークラス**エージェントで終わる — `<lang>-reviewer`、`code-reviewer`、`security-reviewer`、または`database-reviewer`のいずれか。最もドメイン固有のレビュアーがテール位置を獲得する（例: ルール2の`impl+security`は`security-reviewer`で終わる。ルール3の`impl+db`は、`database-reviewer`がすでにチェーンの早い段階でマイグレーションをゲートしているため`<lang>-reviewer`で終わる）。`test`と`build`のステップは独自のバリデーター（`e2e-runner`とビルドリゾルバー）でゲートされており、追加レビュアーは不要。

### フェーズ3 — タスク説明の圧縮

各出力の`<task description>`は以下の条件を満たす必要がある:
- 自己完結型（最初のエージェントが計画ドキュメントを開く必要がない）。
- `[Plan: <path>#step-<id>]`で始まる。
- 1〜3つの検証可能な受け入れ基準を含む。
- スコープガード（`Out of scope: ...`）を含める — **計画がそのステップに対してスコープ外を宣言している場合のみ**。そのまま継承する。計画にスコープ外の記述がない場合、節を完全に省略する — 発明しない。
- 200〜600文字。1行。埋め込みの`"`は`\"`としてエスケープ。リテラルの改行なし。

### フェーズ4 — 出力

`ECC_MODE`で決定した**形式を使用して**Markdownを出力する。出力は1つの形式を全体で使用 — 全ての`{ORCH_CMD}`と全てのエージェント名は、フェーズ0で対応するプレフィックスでレンダリングされる。**両方の形式を出力しない。「これはプラグイン形式です」/「プレフィックスを削除してください」という指示をレンダリングされた出力に含めない。**

具体的なレンダリングルール:

- `{ORCH_CMD}` = `plugin`では`/everything-claude-code:orchestrate`、`legacy`では`/orchestrate`。
- `{AGENT(name)}` = `plugin`では`everything-claude-code:<name>`、`legacy`では`<name>`。
- 概要テーブルの「Chain」列は同じ`{AGENT(name)}`レンダリングを使用する。
- ステップごとのbashブロックには実行可能なコマンドのみを含む。**`# plugin form`や`# legacy form`のコメントなし** — 形式は出力全体で暗黙的かつ統一されている。

出力構造:

````markdown
# Plan-Orchestrate Result

**Plan**: `<path>`
**Lang**: `<detected-or-given>`
**ECC mode**: `<plugin | legacy>`
**Steps**: <N>
**Scope**: <all | step:n | range:a-b>

## Steps overview

| # | Title | Tags | Chain |
|---|---|---|---|
| 1 | ... | impl, db | `{AGENT(tdd-guide)},{AGENT(database-reviewer)},{AGENT(python-reviewer)}` |
| ... | | | |

---

## Step 1 — <title>

**Intent**: <1〜3文>
**Tags**: <a, b>
**Chain rationale**: <このチェーンを選んだ理由。どのエージェントがループを閉じるか>

```bash
{ORCH_CMD} custom "{AGENT(tdd-guide)},{AGENT(database-reviewer)},{AGENT(python-reviewer)}" "[Plan: docs/foo.md#step-1] <圧縮されたタスク説明>; Acceptance: <1〜3項目>; Out of scope: <…>"
```
````

> 上記の`{ORCH_CMD}`と`{AGENT(...)}`表記は、このスキルが実行時に行う置換を説明する。実際に出力されるMarkdownには解決済みの文字列が含まれ、プレースホルダーは含まれない。

全ステップのコマンドを順番にまとめた最終的な「Batch execution」ブロックを追加し、ユーザーが一度に全て貼り付けられるようにする。**概要専用モードではBatchブロックをスキップする**（「大規模な計画」エッジケース参照）: 概要テーブルのみを出力するモードでは、ステップごとのコマンドがないため集約するものがない。

### フェーズ5 — 自己チェック（出力前に実行）

- [ ] 全チェーンの全エージェントがカタログに存在する（計画に現れた`everything-claude-code:`プレフィックスを取り除いた後。フェーズ0ステップ5参照）。
- [ ] 解決済みの`{ORCH_CMD}`と全ての解決済み`{AGENT(...)}`が**同じ**形式（`plugin`または`legacy`）を使用している — 1つの出力内で混在しない。
- [ ] `# plugin form` / `# legacy form`のアノテーションや「プレフィックスを削除してください」という指示がレンダリングされた出力に残っていない。
- [ ] 発明した`--mode` / `--gate` / `--agents=...`フィールドがない。
- [ ] 各タスク説明が1行で、ダブルクォートで囲まれ、埋め込みの`"`がエスケープされている。
- [ ] 各タスク説明が`[Plan: <path>#step-<id>]`で始まり、受け入れ基準（1〜3項目）を含む。`Out of scope:`節は計画から継承した場合のみ存在する。
- [ ] 各チェーンにフェーズ2の重複排除後の重複エージェントがない。
- [ ] チェーン長が4以下。
- [ ] `impl`/`refactor`/`migration`タグのステップがレビュアークラスエージェント（`<lang>-reviewer`、`code-reviewer`、`security-reviewer`、または`database-reviewer`）で終わっている。`test`と`build`は免除 — フェーズ2ルール10参照。
- [ ] ゼロタグのステップが`code-reviewer`を出力し、根拠に`no tag matched; default review-only chain`と記述されている。
- [ ] 概要テーブルが`--scope`に関係なく計画の全ステップを列挙している。
- [ ] ステップごとの詳細ブロック数が解決済みの`--scope`と一致している（`--scope=all`の場合は計画全体。`step:n`の場合は1ブロック。範囲の場合は範囲サイズ）。概要専用モードでは、ステップごとのブロックもBatchブロックも出力しない。

## エッジケース

- **ステップが明確でない**: H2/H3の分割を優先する。それでも曖昧な場合、「no structured steps detected」と報告してドキュメントのアウトラインを表示し、アウトラインに沿って実行することをユーザーに確認する。
- **大規模な計画（1500行超）**: **概要専用モード**に入る — 概要テーブルのみを出力し、詳細を再実行する前に`--scope`で絞り込むよう求める。このモードでは、ステップごとの詳細ブロックとBatch executionブロックをスキップする。
- **ステップが広すぎる**（例: 「全バックエンド作業を完成させる」）: 無理に1つのチェーンにしない。N.aとN.bに分割することを提案し、分割案を提示する。
- **計画がエージェントを宣言している**（まれ）: まず**`everything-claude-code:`プレフィックスを取り除いて**ベアカタログ名を取得し（フェーズ0ステップ5）、カタログに対してバリデーションする。無効なエージェントを置き換え、「Chain rationale」で説明する。ベア名は出力時に`ECC_MODE`に従って再プレフィックスされる。
- **`--lang=auto`が勝者を決定できない多言語プロジェクト**: `lang=unknown`を設定。レビュアーは`code-reviewer`に、ビルドリゾルバーは`build-error-resolver`にフォールバックする。「Chain rationale」でフォールバックについて言及する。

## 例

### 例1 — プラグインモード、Pythonの計画

入力:
```
plan-orchestrate @docs/plan/example-feature.md --lang=python
```

期待される出力の抜粋:
````markdown
## Step 2 — Encrypt sensitive UserProfile fields

**Intent**: `EncryptedString` SQLAlchemy型を導入し、`birth_datetime` / `location`をAES-GCMで暗号化してから永続化する。キーは環境変数からロードする。
**Tags**: impl, security, db
**Chain rationale**: セキュリティに敏感な書き込みパスのため、`security-reviewer`がチェーンを閉じる。`database-reviewer`がalembicマイグレーションを検証し、`python-reviewer`が型付けとPEP 8をカバーする。

```bash
/everything-claude-code:orchestrate custom "everything-claude-code:tdd-guide,everything-claude-code:database-reviewer,everything-claude-code:python-reviewer,everything-claude-code:security-reviewer" "[Plan: docs/plan/example-feature.md#step-2] Implement EncryptedString SQLAlchemy type and migrate UserProfile.birth_datetime/location columns; key from ENV APP_DB_KEY; Acceptance: encrypt/decrypt roundtrip tests pass; alembic upgrade/downgrade clean on empty DB; no plaintext in DB after migrate; Out of scope: cross-tenant profile sharing logic"
```
````

### 例2 — レガシーモード、同じステップ

`ECC_MODE=legacy`が検出された場合、同じステップは統一されたコマンドとして出力される（出力のどこにもプラグインプレフィックス形式なし）:

```bash
/orchestrate custom "tdd-guide,database-reviewer,python-reviewer,security-reviewer" "[Plan: docs/plan/example-feature.md#step-2] ..."
```

上記の2つの例は**2つの異なる環境に対する2つの可能な出力**を示している。1回のスキル呼び出しはそのうちの1つのみを、エンドツーエンドで生成する。

## 注意事項

- 生成のみ。このスキルの内部から`/orchestrate`を呼び出さない。
- タスク説明には計画ドキュメントの言語に合わせる（エージェント名は常に英語のまま）。
- ユーザーが明示的に求めない限り、出力に「Co-Authored-By」行や絵文字を挿入しない。
