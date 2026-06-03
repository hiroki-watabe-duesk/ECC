# 第11章 品質ゲート・CI・テスト

## この章で学ぶこと

- ECCのテスト方式（外部フレームワーク不使用のNode.js素実行）とディレクトリ構成
- カバレッジ計測（c8、80%しきい値）の設定と目的
- CI/CDパイプライン（ci.yml）の全ジョブ構成とマトリクス戦略
- `validate` ジョブで動作する11種の検証スクリプトとそれぞれの責務
- JSONスキーマ群（schemas/）とAJVによる構造保証
- カタログ件数同期（catalog.js）とコマンドレジストリ生成の仕組み
- Lint・整形・コミット規約（ESLint、markdownlint、Prettier、commitlint）
- 補助ワークフロー（supply-chain-watch、release、maintenance、monthly-metrics）
- 品質ゲートコマンドとフック（/quality-gate、quality-gate.js）
- PR前に満たすべき品質チェックリスト
- 自プロジェクトに同等の検証層を最小構成で敷く方法

---

## 11.1 テスト方式の設計思想

ECCのテストは**外部テストフレームワークに依存しない**。Jest・Mocha・Vitestなどは一切使用せず、Node.js標準の `child_process.spawnSync` でテストファイルを個別実行し、その終了コードと標準出力から集計する設計になっている。

この選択には明確な理由がある。ECCはNode.js >=18のCommonJS環境で動作し、インストール済みの依存が少ない状態でも正確に動く必要がある。テストランナー自体を外部フレームワークに委ねると、その起動コスト・設定・バージョン依存が生まれる。自己完結したNode.jsスクリプトとして実装することで、CI・ローカル・WSL・Windowsいずれの環境でも同じコマンドで動作する。

### tests/run-all.js の構造

```
tests/run-all.js
├── walkFiles(testsDir)     # 再帰的にファイルを収集
├── matchesTestGlob()       # glob "tests/**/*.test.js" にマッチするか判定
├── discoverTestFiles()     # ソートされたテストファイル一覧を返す
└── for (testFile) {
      spawnSync('node', [testPath])  # 各ファイルを独立プロセスで実行
      stdout + stderr を解析         # "Passed: N" / "Failed: N" を抽出
    }
```

テストグロブは定数 `TEST_GLOB = 'tests/**/*.test.js'` で管理されている。Node.js 21以降が提供する `path.matchesGlob` を使い、古いバージョンでは正規表現フォールバックを用いる。

実行結果は各テストファイルの標準出力から `Passed: N` / `Failed: N` のパターンを抽出して集計し、最終的に以下の形式でサマリを出力する。

```
╔══════════════════════════════════════════════════════════╗
║                     Final Results                        ║
╠══════════════════════════════════════════════════════════╣
║  Total Tests:  NNN                                       ║
║  Passed:       NNN  ✓                                    ║
║  Failed:         0                                       ║
╚══════════════════════════════════════════════════════════╝
```

失敗したテストファイルが1つでもあれば `process.exit(1)` でCIを止める。

---

## 11.2 テストディレクトリ構成

`tests/` は `scripts/` のミラー構造を採用している。新しいスクリプトを `scripts/lib/` に追加したら、対応するテストを `tests/lib/` に置くというルールが徹底されている。

| ディレクトリ | 役割 |
|---|---|
| `tests/lib/` | `scripts/lib/` 配下のライブラリユーティリティのユニットテスト |
| `tests/hooks/` | `scripts/hooks/` 配下の各フックのユニット・結合テスト |
| `tests/ci/` | `scripts/ci/` 配下のバリデータ・CIスクリプトのテスト |
| `tests/commands/` | `commands/` 配下のコマンドファイルに関する構造テスト |
| `tests/docs/` | ドキュメント整合性・サーフェス検査テスト |
| `tests/integration/` | フック間連携・エンドツーエンドの統合テスト |
| `tests/scripts/` | `scripts/` 直下（install、release、ecc等）のスクリプトテスト |

ファイル命名規則は `*.test.js`（CommonJS）。Pythonテスト（`tests/test_*.py`、`tests/conftest.py`）は別スタックであり、`run-all.js` の収集対象外となる（後述のPythonツールチェーン節を参照）。

---

## 11.3 カバレッジ計測（c8）

カバレッジは `c8` で計測する。`package.json` の `coverage` スクリプトを以下のように定義している。

```json
"coverage": "c8 --all --include=\"scripts/**/*.js\" --check-coverage --lines 80 --functions 80 --branches 80 --statements 80 --reporter=text --reporter=lcov node tests/run-all.js"
```

重要なポイントをまとめると以下の通りである。

| 項目 | 設定値 |
|---|---|
| 計測対象 | `scripts/**/*.js`（`--all` で未実行ファイルも含む） |
| 最低しきい値 | lines / functions / branches / statements すべて **80%** |
| レポート形式 | text（コンソール表示）+ lcov（CI成果物） |
| テスト実行 | `node tests/run-all.js` を内包 |

しきい値未達の場合、`c8` は非ゼロ終了コードを返し、CIの coverage ジョブが失敗する。lcovレポートはCI成果物（`coverage/` ディレクトリ）としてアップロードされ、`coverage-ubuntu-node20-npm` という名前で参照できる。

---

## 11.4 CIパイプライン（ci.yml）

`.github/workflows/ci.yml` は以下の5ジョブで構成されている。プッシュ（`main`、`release/**`、タグ）とプルリクエスト（`main`）をトリガーとする。

### 11.4.1 test ジョブ（マトリクス）

最も広範なジョブ。3軸のマトリクス戦略で多環境互換性を担保する。

| 軸 | 値 |
|---|---|
| OS | ubuntu-latest / windows-latest / macos-latest |
| Node | 18.x / 20.x / 22.x |
| パッケージマネージャ | npm / pnpm / yarn / bun |

ただし Windows + bun の組み合わせは除外（bun のWindows対応が限定的なため）。

各マトリクスセルの実行内容は以下の通り。

```yaml
- npm ci --ignore-scripts                 # npm
- pnpm install --ignore-scripts ...       # pnpm（Node 18はCorepack経由でpnpm@9）
- yarn install --mode=skip-build          # yarn（Corepack経由でstable）
- bun install --ignore-scripts            # bun
→ node tests/run-all.js
  env: CLAUDE_CODE_PACKAGE_MANAGER=${{ matrix.pm }}
```

`COREPACK_ENABLE_STRICT=0` を設定することで、`package.json` が `yarn@4.9.2` を `packageManager` に宣言していても、pnpmがインストールできるようにしている。

失敗した場合は `tests/` ディレクトリをアーティファクトとしてアップロードし、診断を容易にする。

タイムアウトは10分。`fail-fast: false` によりマトリクスの一部が失敗しても他のセルは継続実行される。

### 11.4.2 validate ジョブ

Ubuntu / Node 20 の単一環境で、11種の検証スクリプトを順次実行する（詳細は[11.5節](#115-validateジョブの検証スクリプト群)）。タイムアウトは5分。

### 11.4.3 security ジョブ

サプライチェーンセキュリティに特化した2段階チェック。

```bash
npm audit signatures          # npmレジストリのパッケージ署名検証
npm audit --audit-level=high  # 高・重大レベルの脆弱性で失敗
node scripts/ci/scan-supply-chain-iocs.js  # IOCスキャン
```

`npm audit signatures` はレジストリに公開されたパッケージが改ざんされていないことをSigstore署名で確認する。

### 11.4.4 coverage ジョブ

Ubuntu / Node 20 で `npm run coverage` を実行し、c8のしきい値を確認する。結果はlcovレポートとしてアーティファクトにアップロードされる。

### 11.4.5 lint ジョブ

```bash
npx eslint scripts/**/*.js tests/**/*.js
npx markdownlint "agents/**/*.md" "skills/**/*.md" "commands/**/*.md" "rules/**/*.md"
```

JavaScriptとMarkdownをそれぞれ専用ツールで検査する（詳細は[11.7節](#117-lint整形コミット規約)）。

### 11.4.6 再利用可能ワークフロー

`reusable-test.yml` と `reusable-validate.yml` はそれぞれ `workflow_call` トリガーで定義されており、外部からパラメータ（OS、Nodeバージョン、パッケージマネージャ）を渡して同一ロジックを呼び出せる。リリースワークフローや他のカスタムワークフローからの再利用を想定した設計である。

---

## 11.5 validateジョブの検証スクリプト群

`scripts/ci/` 配下の検証スクリプト群が、コンポーネントの構造整合性を担保する中核である。

| スクリプト | npm script / 呼び出し方 | 検証内容 |
|---|---|---|
| `validate-agents.js` | `node scripts/ci/validate-agents.js` | `agents/*.md` のYAMLフロントマター。必須フィールド `model`（haiku/sonnet/opus）・`tools` の存在と値、重複キーを検査 |
| `validate-hooks.js` | `node scripts/ci/validate-hooks.js` | `hooks/hooks.json` をAJVでスキーマ検証後、イベントタイプ・フックタイプ・フィールド型・インラインJSのシンタックスを検査 |
| `validate-commands.js` | `node scripts/ci/validate-commands.js` | `commands/*.md` の非空チェック、フロントマター構文、他コマンド・エージェント・スキルへの相互参照整合性 |
| `validate-skills.js` | `node scripts/ci/validate-skills.js` | `skills/*/SKILL.md` の存在・非空・フロントマター `name` フィールド。`description` にリテラルブロックスカラー（`\|`）が使われていないことを確認 |
| `validate-rules.js` | `node scripts/ci/validate-rules.js` | `rules/` 配下を再帰走査し、すべての `.md` が非空であることを確認 |
| `validate-install-manifests.js` | `node scripts/ci/validate-install-manifests.js` | `manifests/install-modules.json`・`install-profiles.json`・`install-components.json` をAJVでスキーマ検証、モジュールパスの実在確認 |
| `validate-workflow-security.js` | `node scripts/ci/validate-workflow-security.js` | `.github/workflows/*.yml` の安全パターン検証。`workflow_run`/`pull_request_target` での信頼できないref使用、`npm ci` への `--ignore-scripts` 漏れ等 |
| `catalog.js --text` | `npm run catalog:check` | README.md・AGENTS.md・README.zh-CN.md等に記載されたコンポーネント件数が実ディレクトリと一致するか検証 |
| `generate-command-registry.js --check` | `npm run command-registry:check` | `docs/COMMAND-REGISTRY.json` が現在のコマンド・エージェント・スキルの状態と一致するか検証 |
| `check-unicode-safety.js` | `node scripts/ci/check-unicode-safety.js` | テキストファイル中の危険なUnicode文字（ゼロ幅、ホモグリフ、不可視文字等）を検出 |
| `validate-no-personal-paths.js` | `node scripts/ci/validate-no-personal-paths.js` | `skills/`・`commands/`・`agents/`・`docs/` 等に `/Users/<name>` や `C:\Users\<name>` 形式の個人パスが含まれていないことを確認 |

### 検証スクリプトの設計原則

- **AJV使用箇所**: `validate-hooks.js`・`validate-install-manifests.js` がAJV v8を使用してJSONスキーマ検証を実施する。`allErrors: true` を指定することで、最初のエラーで止まらず全エラーをレポートする。
- **エラーは `process.exit(1)`、警告は `console.warn`**: スキルのフロントマター検査は `--strict` フラグか環境変数 `CI_STRICT_SKILLS=1` がない限り警告にとどまる（CI非ブロッキング）。
- **スクリプト不在でも正常終了**: 対象ディレクトリが存在しない場合は `process.exit(0)` でスキップし、単独導入でも壊れない。

---

## 11.6 スキーマ群（schemas/）

`schemas/` ディレクトリには10個のJSON Schemaが格納されている（draft-07）。AJVによる検証が前提。

| スキーマファイル | 対象データ | 主な制約 |
|---|---|---|
| `hooks.schema.json` | `hooks/hooks.json` | イベント種別（18種）・フックタイプ（command/http/prompt/agent）・必須フィールドを規定 |
| `plugin.schema.json` | `.claude-plugin/plugin.json` | プラグインのname・version（semver）・skills・commands・features等 |
| `install-components.schema.json` | `manifests/install-components.json` | コンポーネントIDのパターン（`baseline:*` / `lang:*` / `framework:*` 等）・family enum |
| `install-modules.schema.json` | `manifests/install-modules.json` | インストールモジュールの構造 |
| `install-profiles.schema.json` | `manifests/install-profiles.json` | インストールプロファイルの構造 |
| `install-state.schema.json` | `.ecc-install-state.json` | インストール状態スナップショット（schemaVersion定数・target・request・resolution・operations） |
| `ecc-install-config.schema.json` | `.ecc` 設定ファイル | ターゲットID enum（claude/cursor/codex/gemini/opencode等） |
| `package-manager.schema.json` | パッケージマネージャ設定 | npm/pnpm/yarn/bun の enum と setAt タイムスタンプ |
| `provenance.schema.json` | `~/.claude/skills/learned/*` 等 | スキル来歴情報（source・created_at・confidence・author） |
| `state-store.schema.json` | ECCステートストア | sessions・skillRuns・skillVersions・decisions・installState・governanceEvents・workItems の型定義 |

### スキーマ検証を実装する最小コード例

```javascript
const Ajv = require('ajv');
const schema = require('./schemas/hooks.schema.json');
const data = JSON.parse(fs.readFileSync('hooks/hooks.json', 'utf8'));

const ajv = new Ajv({ allErrors: true });
const validate = ajv.compile(schema);
if (!validate(data)) {
  validate.errors.forEach(err =>
    console.error(`ERROR: ${err.instancePath || '/'} ${err.message}`)
  );
  process.exit(1);
}
```

---

## 11.7 カタログ件数同期とコマンドレジストリ

### catalog.js：件数の単一真実源

`scripts/ci/catalog.js` はコンポーネント件数のドリフトを防ぐための同期ツールである。

**検査対象ドキュメント**:
- `README.md`（英語）- クイックスタートサマリ、プロジェクトツリー、比較表、パリティ表の4箇所
- `AGENTS.md` - サマリ行とプロジェクト構造
- `README.zh-CN.md` - 中国語版ルートREADME
- `docs/zh-CN/README.md` / `docs/zh-CN/AGENTS.md` - 中国語版ドキュメント
- `.claude-plugin/plugin.json` / `.claude-plugin/marketplace.json` - プラグインJSON

**動作モード**:

```bash
npm run catalog:check   # 不一致があれば exit 1
npm run catalog:sync    # --write で自動修正してから確認
node scripts/ci/catalog.js --json   # JSON出力
node scripts/ci/catalog.js --md     # Markdown表出力
```

`buildCatalog()` 関数は実ディレクトリを走査して件数を集計する。

```
agents: agents/*.md のファイル数
commands: commands/*.md のファイル数
skills: skills/*/SKILL.md が存在するディレクトリ数
```

### generate-command-registry.js：コマンドの静的インデックス

```bash
npm run command-registry:generate  # docs/COMMAND-REGISTRY.json を生成
npm run command-registry:write     # --write フラグ付きで書き込み
npm run command-registry:check     # 差分があれば exit 1
```

`docs/COMMAND-REGISTRY.json` は各コマンドファイルのフロントマターを解析し、コマンド名・エージェント参照・スキル参照を索引化した決定的なJSONファイルである。CIでは `--check` モードで現状との差分を検出する。

---

## 11.8 Lint・整形・コミット規約

### ESLint（eslint.config.js）

flat config 形式（ESLint v9）を採用。

```javascript
rules: {
  'no-unused-vars': ['error', {
    argsIgnorePattern: '^_',
    varsIgnorePattern: '^_',
    caughtErrorsIgnorePattern: '^_'
  }],
  'no-undef': 'error',
  'eqeqeq': 'warn'
}
```

対象は `scripts/**/*.js` と `tests/**/*.js`。`.mjs` ファイルは ESM として扱われる。`node_modules/`、`coverage/`、`.venv/` 等は無視。

### markdownlint（.markdownlint.json）

多くのルールを無効化し、実用的な運用を優先している。

| ルール | 設定 |
|---|---|
| MD013（行長制限） | 無効 |
| MD033（インラインHTML） | 無効 |
| MD041（ファイル先頭H1必須） | 無効 |
| MD040（コードブロック言語指定） | 無効 |
| MD009（末尾スペース） | `br_spaces: 2, strict: false` |
| MD024（重複見出し） | `siblings_only: true`（兄弟レベルのみ重複禁止） |

### Prettier（.prettierrc）

```json
{
  "singleQuote": true,
  "trailingComma": "none",
  "semi": true,
  "tabWidth": 2,
  "printWidth": 200,
  "arrowParens": "avoid"
}
```

`printWidth: 200` は通常の80より大幅に広く、長い文字列を折り返さないことを優先している。`trailingComma: "none"` はCommonJS環境での互換性を考慮した選択である。

### commitlint（commitlint.config.js）

Conventional Commits準拠。許可される `type` は以下の通り。

```
feat / fix / docs / style / refactor / perf / test / chore / ci / build / revert
```

`subject-case` はlower-case以外（センテンスケース、パスカルケース等）が禁止。`header-max-length` は100文字まで。

---

## 11.9 補助ワークフロー

### supply-chain-watch.yml（6時間ごと定期実行）

```
cron: '17 */6 * * *'
```

- `npm audit signatures` + `npm audit --audit-level=high`
- `tests/ci/scan-supply-chain-iocs.test.js` - IOCスキャナのフィクスチャ検証
- `tests/ci/supply-chain-advisory-sources.test.js` - アドバイザリソース検証
- `scripts/ci/scan-supply-chain-iocs.js --json` - IOCレポートをJSONで生成
- `scripts/ci/validate-workflow-security.js` - ワークフローハードニング確認
- 成果物 `supply-chain-ioc-report.json` を14日間保持

`scan-supply-chain-iocs.js` は既知の悪意あるパッケージバージョン（`MALICIOUS_PACKAGE_VERSIONS` として列挙）をロックファイルやAIツール設定ファイルと照合する。

### release.yml（vタグプッシュ時）

2ジョブ構成（`verify` → `publish`）。

**verifyジョブの主な検証**:
1. IOCスキャン（`npm run security:ioc-scan`）
2. バージョンタグ形式の正規表現チェック（`^v[0-9]+\.[0-9]+\.[0-9]+(-...)?$`）
3. `package.json` バージョンとタグの一致確認
4. プラグインマニフェスト整合性（`node tests/plugin-manifest.test.js`）
5. npmパック（`npm pack --json`）

**publishジョブ**:
- GitHub Release作成（`softprops/action-gh-release`）
- npm公開（`--provenance` フラグ付きでSigstore来歴情報を添付）
- `id-token: write` 権限でOIDCトークンを取得

### maintenance.yml（毎週月曜9時UTC）

- `npm outdated` で依存の陳腐化を確認
- `npm audit` で週次セキュリティ監査
- `actions/stale` でIssue/PRを30日後にstale、7日後にclose

### monthly-metrics.yml（毎月1日14時UTC）

`ecc-universal`・`ecc-agentshield` のnpmダウンロード数（週次・月次）、スター数、フォーク数、コントリビュータ数、リリース数、GitHub Traffic（14日間）を収集し、専用Issueの表に行を追記する。

---

## 11.10 Pythonツールチェーン

ECCにはNode.jsとは独立したPythonスタックが存在する（`pyproject.toml`）。

**対象**: `src/llm/` 配下のプロバイダ非依存LLM抽象化レイヤー

| ツール | 用途 | 設定 |
|---|---|---|
| pytest | テスト実行 | `asyncio_mode = "auto"` |
| ruff | Lint・フォーマット | Python 3.11ターゲット |
| mypy | 型チェック | `warn_return_any = true` |
| pytest-cov | カバレッジ | `source = ["src/llm"]` |

**ツールバージョン管理**: `.tool-versions` で以下を固定している。

```
nodejs 20.19.0
python 3.12.8
```

asdf・mise どちらの環境でも `asdf install` または `mise install` で再現できる。

Pythonテストは `node tests/run-all.js` の収集対象外であるため、Python側のCIを動かす場合は別途 `pytest` を実行する必要がある。

---

## 11.11 品質ゲートコマンドとフック

### /quality-gate コマンド

`commands/quality-gate.md` が定義するスラッシュコマンド。

```
/quality-gate [path|.] [--fix] [--strict]
```

パイプライン：
1. 対象ファイル/ディレクトリの言語・ツーリング検出
2. フォーマッタチェック（Prettier / Biome / gofmt / ruff）
3. Lint・型チェック（利用可能な場合）
4. 簡潔な修正リストの生成

`--fix` を渡すと自動フォーマットを許可。`--strict` は警告をエラーに昇格させる。

### scripts/hooks/quality-gate.js

PostToolUse フックとして動作するスクリプト。ファイル編集後に自動的に呼び出される。

```javascript
// 拡張子ごとの処理ロジック
'.js/.jsx/.ts/.tsx/.json/.md' → formatter検出 → biome/prettier 実行
'.go'                          → gofmt 実行
'.py'                          → ruff format 実行
```

重要な実装上の注意点：
- Biome環境で`.ts/.tsx/.js/.jsx` は `post-edit-format.js` が既に処理するため、このフックではスキップする（二重処理の防止）
- `ECC_QUALITY_GATE_FIX=true` 環境変数で `--fix`/`--write` を有効化
- `ECC_QUALITY_GATE_STRICT=true` で失敗をstderrへ出力
- すべての処理はparse失敗を含めて `exit 0`（ツールの実行をブロックしない設計）

フックはstdinからJSONを受け取り、`tool_input.file_path` を抽出して処理し、入力をそのままstdoutに返すpass-through設計である。

---

## 11.12 貢献者の品質チェックリスト

PRを提出する前に以下の項目をローカルで確認する。

### 必須チェック

```bash
# 1. テスト全件通過
node tests/run-all.js

# 2. バリデータ一括実行（npm testの一部として実行される）
npm test
# = check-unicode-safety → validate-agents → validate-commands → validate-rules
#   → validate-skills → validate-hooks → validate-install-manifests
#   → validate-no-personal-paths → catalog:check → command-registry:check
#   → tests/run-all.js

# 3. カバレッジ確認（80%以上）
npm run coverage

# 4. ESLint
npx eslint scripts/**/*.js tests/**/*.js

# 5. markdownlint
npx markdownlint "agents/**/*.md" "skills/**/*.md" "commands/**/*.md" "rules/**/*.md"
```

### コンポーネント追加時の追加チェック

```bash
# エージェント・スキル・コマンドを追加・削除した場合
npm run catalog:sync            # ドキュメント件数を自動修正
npm run command-registry:write  # コマンドレジストリを再生成

# 確認
npm run catalog:check
npm run command-registry:check
```

### コミット規約

```
feat: 新機能の追加
fix: バグ修正
docs: ドキュメントのみの変更
refactor: 動作変更なしのコード整理
test: テストの追加・修正
chore: ビルド・ツール・設定の変更
ci: CI/CDパイプラインの変更
```

ヘッダは100文字以内、サブジェクトはlower-case（センテンスケース禁止）。

### PR記入チェックリスト（.github/PULL_REQUEST_TEMPLATE.md）

- [ ] `node tests/run-all.js` がローカルで通過
- [ ] JSONファイルが正しく検証された
- [ ] シークレット・APIキーが含まれていない（`ghp_`、`sk-`、`AKIA`、`xoxb`、`xoxp` パターン）
- [ ] Conventional Commits形式に従っている
- [ ] 機密情報がログや出力に露出していない

---

## 11.13 自プロジェクトへの最小適用

ECCの検証層から、自プロジェクトに流用できる最小構成を提案する。

### ステップ1：テストランナーの自作（外部依存なし）

```javascript
// tests/run-all.js
const { spawnSync } = require('child_process');
const fs = require('fs');
const path = require('path');

function walkTestFiles(dir) {
  const results = [];
  for (const entry of fs.readdirSync(dir, { withFileTypes: true })) {
    const full = path.join(dir, entry.name);
    if (entry.isDirectory()) results.push(...walkTestFiles(full));
    else if (entry.name.endsWith('.test.js')) results.push(full);
  }
  return results.sort();
}

let passed = 0, failed = 0;
for (const file of walkTestFiles(path.join(__dirname, 'tests'))) {
  const r = spawnSync('node', [file], { encoding: 'utf8' });
  const ok = (r.stdout + r.stderr).match(/Passed:\s*(\d+)/)?.[1];
  const ng = (r.stdout + r.stderr).match(/Failed:\s*(\d+)/)?.[1];
  if (ok) passed += parseInt(ok);
  if (ng) failed += parseInt(ng);
  if (r.status !== 0) failed += ng ? 0 : 1;
}
process.exit(failed > 0 ? 1 : 0);
```

### ステップ2：スキーマ検証の追加

プロジェクト固有のJSON設定（hooks設定、マニフェスト等）にスキーマを定義し、AJV（`npm install ajv`）で検証するスクリプトを `scripts/ci/validate-*.js` として用意する。基本パターンはECCの `validate-hooks.js` を参考にできる。

### ステップ3：カタログ同期の実装

ドキュメントにコンポーネント件数を記載している場合、`catalog.js` のパターンを流用して「実際の件数」と「ドキュメントに記載された件数」の乖離を検出するスクリプトを作る。`--write` モードで自動修正できる設計にすることで、手作業による数値の維持コストをゼロにできる。

### ステップ4：GitHub Actions への組み込み

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@...
      - uses: actions/setup-node@...
        with: { node-version: '20.x' }
      - run: npm ci --ignore-scripts
      - run: node tests/run-all.js

  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@...
      - uses: actions/setup-node@...
        with: { node-version: '20.x' }
      - run: npm ci --ignore-scripts
      - run: node scripts/ci/validate-hooks.js
      - run: node scripts/ci/catalog.js --text
```

この最小構成でも「テスト通過」「スキーマ整合性」「ドキュメント件数の一致」という3つの品質ゲートが成立する。

---

## 関連章

- [第7章: フックとランタイム](07-hooks-and-runtime.md) — フック実装の詳細とPostToolUseフックの設計
- [第10章: クロスハーネスとインストール](10-cross-harness-and-install.md) — インストールマニフェスト・スキーマの全体像
- [第12章: コンポーネント作成ガイド](12-authoring-components.md) — エージェント・スキル・フックの正しい記述形式

## 参照ソース

| ソース | 役割 |
|---|---|
| `tests/run-all.js` | テストランナー本体 |
| `.github/workflows/ci.yml` | CIパイプライン定義 |
| `.github/workflows/reusable-test.yml` | 再利用可能テストワークフロー |
| `.github/workflows/reusable-validate.yml` | 再利用可能バリデーションワークフロー |
| `.github/workflows/supply-chain-watch.yml` | サプライチェーン監視 |
| `.github/workflows/release.yml` | リリースパイプライン |
| `.github/workflows/maintenance.yml` | 週次メンテナンス |
| `.github/workflows/monthly-metrics.yml` | 月次メトリクス |
| `scripts/ci/validate-agents.js` | エージェント検証 |
| `scripts/ci/validate-hooks.js` | フック検証（AJV） |
| `scripts/ci/validate-commands.js` | コマンド検証・相互参照チェック |
| `scripts/ci/validate-skills.js` | スキル検証 |
| `scripts/ci/validate-rules.js` | ルール検証 |
| `scripts/ci/validate-install-manifests.js` | インストールマニフェスト検証（AJV） |
| `scripts/ci/validate-workflow-security.js` | ワークフローセキュリティ検証 |
| `scripts/ci/catalog.js` | カタログ件数同期 |
| `scripts/ci/generate-command-registry.js` | コマンドレジストリ生成・検証 |
| `scripts/ci/check-unicode-safety.js` | Unicode安全性チェック |
| `scripts/ci/validate-no-personal-paths.js` | 個人パス検出 |
| `scripts/ci/scan-supply-chain-iocs.js` | IOCスキャン |
| `schemas/hooks.schema.json` | フック設定スキーマ |
| `schemas/plugin.schema.json` | プラグインスキーマ |
| `schemas/install-*.schema.json` | インストール関連スキーマ群 |
| `schemas/state-store.schema.json` | ステートストアスキーマ |
| `schemas/provenance.schema.json` | スキル来歴スキーマ |
| `commands/quality-gate.md` | 品質ゲートコマンド定義 |
| `scripts/hooks/quality-gate.js` | 品質ゲートフック実装 |
| `eslint.config.js` | ESLint flat config |
| `.markdownlint.json` | markdownlint設定 |
| `.prettierrc` | Prettier設定 |
| `commitlint.config.js` | コミット規約設定 |
| `package.json` | scripts・devDependencies |
| `pyproject.toml` | Pythonツールチェーン設定 |
| `.tool-versions` | Node.js/Pythonバージョン固定 |
| `.github/PULL_REQUEST_TEMPLATE.md` | PRテンプレート |
