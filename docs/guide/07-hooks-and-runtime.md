# 第07章 フックとランタイム機構

## この章で学ぶこと

- Claude Code が発火させるフックイベントの種別と発火タイミング
- `hooks/hooks.json` のスキーマ構造と各フィールドの意味
- `run-with-flags.js` を中心とした ECC 実行フレームワークのデータフロー
- `module.exports = { run }` パターンによる高速インプロセス実行
- プロファイル制御（`ECC_HOOK_PROFILE`）と個別無効化（`ECC_DISABLED_HOOKS`）
- `bash-hook-dispatcher.js` による連鎖実行パターン
- 現在登録されているフック群の一覧と役割
- フック入力 JSON の形と `additionalContext` の仕組み
- クロスプラットフォーム対応とパッケージマネージャ検出
- 安全なフックを書く 10 の原則

---

## Claude Code におけるフックとは（初心者向け）

Claude Code の **フック（hook）** は、「LLM が忘れても確実に実行される決定論的な自動化」です。スキルやプロンプトでの指示は Claude が文脈に応じて解釈しますが、フックはハーネスが直接スクリプトを呼び出すため、Claude の判断を介しません。

### なぜフックが必要なのか

Claude はコンテキストが長くなると初期の指示を忘れることがあります。また、特定の副作用（フォーマット・品質チェック・ログ）を「毎回必ず実行する」保証はプロンプトだけでは難しいです。フックはこの問題を解決します。

| 手段 | 保証の強さ | 向いている用途 |
|------|-----------|---------------|
| プロンプト・スキル | LLM が解釈する（忘れる可能性あり） | ワークフロー・知識の提供 |
| ルール | セッション全体で有効（常時コンテキスト） | コーディング規約・ガードレール |
| **フック** | ハーネスが決定論的に実行（LLM の判断を介さない） | フォーマット・品質ゲート・ブロック |

### ライフサイクルイベントの考え方

Claude Code はツール実行やセッションの節目に **ライフサイクルイベント** を発生させます。フックはこれらのイベントに「反応して実行されるスクリプト」です。

```
セッション開始
    │  ← SessionStart フック（コンテキスト注入・復元）
    │
    ├─ Claude がツール実行を決定
    │      │  ← PreToolUse フック（検査・ブロック）
    │      │
    │      ツール実行
    │      │
    │      │  ← PostToolUse フック（品質ゲート・ログ）
    │
    ├─ Claude が応答を出力
    │      │  ← Stop フック（フォーマット・セッション保存）
    │
セッション終了
    │  ← SessionEnd フック（ライフサイクルマーカー）
```

### PreToolUse でツール実行をブロックする（exit 2）

`PreToolUse` フックは特別で、スクリプトが **exit code 2** を返すとツールの実行を中断できます。これがECC のガードレール（設定ファイル保護・コミット品質チェックなど）が機能するコアメカニズムです。

```javascript
// フック内でブロックする場合（例: 危険なコマンドを検出）
return { exitCode: 2, stderr: 'このコマンドはブロックされました: 理由' };

// 通過させる場合
return { exitCode: 0 };
```

exit code 0 以外（1 を含む）は**通常エラー**として記録され、ツールは通過します。ブロックは exit code **2** のみです。

### フックは stdin で JSON を受け取る

Claude Code はフックスクリプトの標準入力（stdin）にイベント情報を JSON で渡します。フックスクリプトはこの JSON を読み込んで判断や処理を行います。

```json
// PreToolUse / PostToolUse の stdin 例
{
  "tool_name": "Bash",
  "tool_input": {
    "command": "git push --force"
  }
}
```

### フックの登録方法

フックは `settings.json`（または ECC プラグインの `hooks/hooks.json`）に登録します。

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "node scripts/hooks/my-check.js"
          }
        ]
      }
    ]
  }
}
```

`matcher` でどのツールに反応するかを指定し、`hooks` 配列に実行するスクリプトを列挙します。

> コンポーネント全体の関係については [03-components-overview.md](03-components-overview.md) を参照してください。

---

## フックイベント種別

Claude Code は以下のライフサイクルイベントを発火します。ECC はこのうち現時点で利用可能なイベントに対してフックを登録しています。

| イベント | 発火タイミング | ブロッキング | ECC での使用 |
|---|---|---|---|
| `SessionStart` | 新しいセッション開始時 | なし（情報注入） | セッション復元・パッケージマネージャ検出 |
| `PreToolUse` | ツール実行の直前 | exit code 2 でブロック可 | Bash 検査・設定保護・GateGuard |
| `PostToolUse` | ツール実行の直後（成功時） | なし | 品質ゲート・ログ・メトリクス |
| `PostToolUseFailure` | ツール実行の直後（失敗時） | なし | MCP ヘルスチェック失敗記録 |
| `PreCompact` | コンテキスト圧縮の直前 | なし | 状態保存 |
| `Stop` | Claude の各応答終了後 | 応答を通す or ブロック | フォーマット・型検査・セッション記録 |
| `SessionEnd` | セッション終了時 | なし | ライフサイクルマーカー |

> **注意:** スキーマ（`schemas/hooks.schema.json`）には `UserPromptSubmit`、`PermissionRequest`、`SubagentStart`、`Notification`、`TeammateIdle`、`TaskCompleted`、`ConfigChange`、`WorktreeCreate`、`WorktreeRemove` なども定義されていますが、ECC の `hooks.json` では現在未使用です。これらは将来のフック登録に備えて定義されたイベントです。

### PreToolUse のブロッキング動作

`PreToolUse` フックは exit code 2 を返すことでツール実行を中断できます。これが ECC のガードレールが機能するコアメカニズムです。exit code 0 は通過、それ以外（1 を含む）は通常エラーとして記録され、ツールは通過します。

---

## hooks.json のスキーマ

フックの登録は `hooks/hooks.json` で行います。トップレベルの構造は以下のとおりです。

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "hooks": {
    "PreToolUse": [ /* matcherEntry[] */ ],
    "PostToolUse": [ /* matcherEntry[] */ ],
    "Stop":        [ /* matcherEntry[] */ ],
    ...
  }
}
```

### matcherEntry の構造

```json
{
  "matcher": "Bash|Write|Edit|MultiEdit",
  "hooks": [ /* hookItem[] */ ],
  "description": "人間可読な説明",
  "id": "pre:bash:dispatcher"
}
```

`matcher` フィールドは以下の値を取れます。

| 値 | 意味 |
|---|---|
| `"Bash"` | Bash ツールにのみマッチ |
| `"Edit\|Write"` | Edit または Write ツールにマッチ |
| `"*"` | すべてのツールにマッチ |
| オブジェクト | 高度な条件オブジェクト（スキーマ定義あり） |

### hookItem の種類

スキーマが定義する hookItem の型は 3 種類です。

**command 型（最も多く使用）**

```json
{
  "type": "command",
  "command": "node scripts/hooks/...",
  "async": false,
  "timeout": 30
}
```

- `async: true` を指定すると非同期（バックグラウンド）実行になります。バックグラウンドフックは Claude の応答をブロックしません。
- `timeout` は秒単位です。

**http 型**

```json
{
  "type": "http",
  "url": "https://example.com/hook",
  "headers": { "Authorization": "Bearer TOKEN" },
  "allowedEnvVars": ["MY_SECRET"],
  "timeout": 10
}
```

**prompt/agent 型**

```json
{
  "type": "prompt",
  "prompt": "Analyze the tool output and...",
  "model": "claude-haiku-4-5",
  "timeout": 30
}
```

`agent` も `prompt` と同じフィールドを持ちます。

---

## 実行フレームワーク: run-with-flags.js

ECC のフック実行の中核は `scripts/hooks/run-with-flags.js` です。このスクリプトはすべてのフックへの統一ゲートウェイとして機能します。

### 引数シグネチャ

```bash
node scripts/hooks/run-with-flags.js <hookId> <scriptRelativePath> [profilesCsv]
```

例:

```bash
node run-with-flags.js pre:quality-gate scripts/hooks/quality-gate.js standard,strict
```

### データフロー

```
stdin (JSON)
    │
    ▼
[1] stdin 読み取り（最大 1 MB: MAX_STDIN = 1024 * 1024）
    │
    ▼
[2] isHookEnabled(hookId, { profiles: profilesCsv })
    ├── ECC_DISABLED_HOOKS に hookId が含まれる → exit 0（通過）
    └── ECC_HOOK_PROFILE が profiles に含まれない → exit 0（通過）
    │
    ▼
[3] パストラバーサル検査
    scriptPath がプラグインルート外に出ていないか確認
    （rejected の場合 stderr に警告して exit 0）
    │
    ▼
[4] スクリプトの存在確認（なければ exit 0）
    │
    ▼
[5] インプロセス実行を試みる
    ソースに `module.exports` と `run` が含まれるか検査
    ├── YES → require() して hookModule.run(raw, options) を呼ぶ
    │          （子プロセス不要: ~50-100ms 節約）
    └── NO  → spawnSync で子プロセス起動（レガシーパス）
    │
    ▼
[6] 結果の書き出し（emitHookResult）
    stdout に渡して exit
```

### run() 関数の返却値

`module.exports = { run }` を実装するフックは以下を返します。

```javascript
// 最もシンプルな形（通過）
return { exitCode: 0 };

// ブロック（PreToolUse のみ）
return { exitCode: 2, stderr: '理由メッセージ' };

// 可視コンテキスト注入（PreToolUse のみ）
return {
  exitCode: 0,
  additionalContext: 'Claude に見せる追加情報のテキスト',
};

// stdout を上書きする場合
return { exitCode: 0, stdout: '新しい JSON' };
```

`additionalContext` を返すと、`pretooluse-visible-output.js` が以下の JSON を構築して Claude のコンテキストに注入します。

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "additionalContext": "...テキスト..."
  }
}
```

このメカニズムにより、フックはツール実行をブロックせずに Claude に情報を渡すことができます（例: doc-file-warning.js）。

### レガシーパス（子プロセス）

`module.exports.run` が存在しない場合は `spawnSync` で子プロセスを起動します。この場合、環境変数として以下が設定されます。

| 変数 | 内容 |
|---|---|
| `CLAUDE_PLUGIN_ROOT` | プラグインルートパス |
| `ECC_PLUGIN_ROOT` | 同上（旧名） |
| `ECC_HOOK_ID` | フック ID 文字列 |
| `ECC_HOOK_INPUT_TRUNCATED` | 入力が切り捨てられた場合 `"1"` |
| `ECC_HOOK_INPUT_MAX_BYTES` | MAX_STDIN の値（文字列） |

---

## プロファイルと個別無効化

### ECC_HOOK_PROFILE

ECC のフックはプロファイルごとに有効・無効が切り替わります。

```bash
export ECC_HOOK_PROFILE=minimal    # 最低限のフックのみ
export ECC_HOOK_PROFILE=standard   # 既定値
export ECC_HOOK_PROFILE=strict     # 最大限のフック
```

有効なプロファイルは `minimal`、`standard`、`strict` の 3 つです（`hook-flags.js`）。無効な値を設定すると `standard` にフォールバックします。

各フック登録で `profilesCsv` 引数（例: `"standard,strict"`）を指定します。現在の `ECC_HOOK_PROFILE` がそのリストに含まれる場合のみフックが実行されます。

### ECC_DISABLED_HOOKS

特定のフックを個別に無効化するには、フック ID をカンマ区切りで指定します。

```bash
export ECC_DISABLED_HOOKS=pre:bash:commit-quality,pre:bash:tmux-reminder
```

`isHookEnabled()` はまず `ECC_DISABLED_HOOKS` を確認し、含まれていればプロファイルに関係なくスキップします。

---

## bash-hook-dispatcher による連鎖実行

`scripts/hooks/bash-hook-dispatcher.js` は、複数のフックロジックを単一プロセスで連鎖実行するためのディスパッチャです。

### 仕組み

```javascript
const PRE_BASH_HOOKS = [
  { id: 'pre:bash:block-no-verify',  profiles: 'minimal,standard,strict', run: ... },
  { id: 'pre:bash:auto-tmux-dev',                                          run: ... },
  { id: 'pre:bash:tmux-reminder',    profiles: 'strict',                   run: ... },
  { id: 'pre:bash:git-push-reminder',profiles: 'strict',                   run: ... },
  { id: 'pre:bash:commit-quality',   profiles: 'strict',                   run: ... },
  { id: 'pre:bash:gateguard-fact-force', profiles: 'standard,strict',      run: ... },
];
```

ディスパッチャは各フックを順に呼び出します。いずれかが exit code 0 以外（ブロック）を返した時点で連鎖を中断し、結果を返します。各フックが返す `additionalContext` は `combineAdditionalContext()` で結合されます。

この設計の利点は、Bash ツールの PreToolUse として登録するエントリが 1 つで済み、子プロセスの起動コストが最小化されることです。

同様に `POST_BASH_HOOKS`（post:bash:command-log-audit、post:bash:pr-created、post:bash:build-complete 等）も同一プロセス内で処理されます。

---

## 現在登録されているフック群

`hooks/hooks.json` に現在登録されているフックを整理します。

### PreToolUse

| ID | matcher | プロファイル | 役割 |
|---|---|---|---|
| `pre:bash:dispatcher` | Bash | 全て（ディスパッチャ内で制御） | Bash 実行前の複合チェック（block-no-verify / GateGuard 等を連鎖） |
| `pre:write:doc-file-warning` | Write | standard, strict | アドホックなドキュメントファイル名を警告（ブロックなし） |
| `pre:edit-write:suggest-compact` | Edit\|Write | standard, strict | 論理的な区切りで `/compact` を提案 |
| `pre:observe:continuous-learning` | `*` | standard, strict | ツール使用を継続学習用に記録（async） |
| `pre:governance-capture` | Bash\|Write\|Edit\|MultiEdit | standard, strict | 秘密情報・ポリシー違反イベントをキャプチャ |
| `pre:config-protection` | Write\|Edit\|MultiEdit | standard, strict | linter/formatter 設定ファイルへの変更をブロック |
| `pre:mcp-health-check` | `*` | standard, strict | MCP サーバーの健全性を確認し、不健全な場合はブロック |
| `pre:edit-write:gateguard-fact-force` | Edit\|Write\|MultiEdit | standard, strict | ファイルへの初回編集前に調査（インポーター等）を要求 |

### PreCompact

| ID | matcher | 役割 |
|---|---|---|
| `pre:compact` | `*` | 圧縮前に状態を保存 |

### SessionStart

| ID | matcher | 役割 |
|---|---|---|
| `session:start` | `*` | 前回セッション要約・パッケージマネージャ・プロジェクト型を読み込み |

### PostToolUse

| ID | matcher | 非同期 | 役割 |
|---|---|---|---|
| `post:bash:dispatcher` | Bash | async (30s) | Bash 実行後のログ・PR 通知・ビルド完了通知 |
| `post:quality-gate` | Edit\|Write\|MultiEdit | async (30s) | ファイル編集後の品質チェック |
| `post:edit:design-quality-check` | Edit\|Write\|MultiEdit | – (10s) | フロントエンド UI の品質チェック |
| `post:edit:accumulator` | Edit\|Write\|MultiEdit | – | 編集した JS/TS ファイルを Stop 時の一括処理用に記録 |
| `post:edit:console-warn` | Edit | – | console.log の混入を警告 |
| `post:governance-capture` | Bash\|Write\|Edit\|MultiEdit | – (10s) | ツール出力からガバナンスイベントをキャプチャ |
| `post:session-activity-tracker` | `*` | – (10s) | セッション内のツール使用を集計 |
| `post:observe:continuous-learning` | `*` | async (10s) | ツール使用結果を継続学習用に記録 |
| `post:ecc-metrics-bridge` | `*` | – (10s) | セッションメトリクスを集計（ステータスライン用） |
| `post:ecc-context-monitor` | `*` | – (10s) | コンテキスト枯渇・高コスト・ループを警告 |

### PostToolUseFailure

| ID | matcher | 役割 |
|---|---|---|
| `post:mcp-health-check` | `*` | MCP ツール失敗を記録し、再接続を試みる |

### Stop

| ID | timeout | 非同期 | 役割 |
|---|---|---|---|
| `stop:format-typecheck` | 300s | – | 編集済み JS/TS を一括フォーマット＆型検査 |
| `stop:check-console-log` | 30s | – | 変更ファイル内の console.log をチェック |
| `stop:session-end` | 10s | async | セッション状態をファイルに永続化 |
| `stop:evaluate-session` | 10s | async | パターン抽出のためセッションを評価 |
| `stop:cost-tracker` | 10s | async | セッションのトークン・コストを記録 |
| `stop:desktop-notify` | 10s | async | タスク完了をデスクトップ通知（macOS/WSL） |

### SessionEnd

| ID | 役割 |
|---|---|
| `session:end:marker` | セッション終了ライフサイクルマーカー（非ブロッキング） |

---

## フック入力 JSON の形

Claude Code は各フックの stdin に JSON を送ります。イベント種別によって形が異なります。

### PreToolUse / PostToolUse 共通部分

```json
{
  "tool_name": "Edit",
  "tool_input": {
    "file_path": "/path/to/file.ts",
    "old_string": "...",
    "new_string": "..."
  }
}
```

Bash の場合は `tool_input.command` にコマンド文字列が入ります。

### PostToolUse 追加フィールド

```json
{
  "tool_output": "コマンドの標準出力や結果",
  "tool_name": "Bash",
  "tool_input": { "command": "npm test" }
}
```

### Stop / SessionEnd

```json
{
  "transcript_path": "/path/to/session-uuid.jsonl"
}
```

`session-end.js` はこの `transcript_path` を読み込み、ユーザーメッセージ・使用ツール・変更ファイルを抽出してセッションファイルを更新します。

---

## パッケージマネージャ検出

`scripts/lib/package-manager.js` は以下の優先順で検出します。

| 優先度 | ソース | 例 |
|---|---|---|
| 1 | 環境変数 `CLAUDE_PACKAGE_MANAGER` | `export CLAUDE_PACKAGE_MANAGER=pnpm` |
| 2 | プロジェクト設定 `.claude/package-manager.json` | `{"packageManager": "pnpm"}` |
| 3 | `package.json` の `packageManager` フィールド | `"packageManager": "pnpm@8.6.0"` |
| 4 | ロックファイル | `pnpm-lock.yaml`、`bun.lockb`、`yarn.lock`、`package-lock.json` |
| 5 | グローバル設定 `~/.claude/package-manager.json` | – |
| 6 | デフォルト: npm | – |

ロックファイルの検出優先順は `pnpm > bun > yarn > npm` です（`DETECTION_PRIORITY` 定数）。

> **注意:** `getAvailablePackageManagers()` は `where.exe`（Windows）や `which`（Unix）を子プロセスで起動するため、セッション開始フックのホットパスでは呼び出しません。これは Windows の Bun 環境でスポーンリミットを超えてフリーズする問題（#162）への対処です。代わりに `detectFromLockFile()` や `detectFromPackageJson()` を使います。

---

## 安全なフックを書く 10 の原則

1. **exit 0 で非致命にする**: パースエラーや予期しない入力の場合は `process.exit(0)` で通過させます。フックがツール実行を誤ってブロックすることを防ぎます。

2. **ブロッキングフックは高速に保つ**: `PreToolUse` のブロッキングフックは 200ms 以内に完了させます。ネットワーク呼び出し、大規模なファイル読み込みは避けます。

3. **非ブロッキング処理には `async: true` を使う**: ログ、メトリクス、通知などは `"async": true` で非同期実行し、Claude の応答速度に影響しないようにします。タイムアウトは 30 秒以内に設定します。

4. **`module.exports = { run }` を実装する**: インプロセス実行パスが有効になり、子プロセス起動（~50-100ms）を節約できます。`run(rawInput, options)` を実装し、副作用をモジュールスコープに置かないようにします。

5. **パストラバーサルを検証する**: ファイルパスを扱う場合は、プラグインルートの外に出ていないか確認します。`run-with-flags.js` が自動的に行いますが、フック内でも二重確認を推奨します。

6. **プロファイルを尊重する**: `profilesCsv` を run-with-flags.js に渡すことで、`ECC_HOOK_PROFILE` の設定に応じてフックが自動的に有効・無効になります。

7. **`ECC_DISABLED_HOOKS` に対応する**: `isHookEnabled()` を通じて自動対応されます。フック ID に `pre:` / `post:` プレフィックスと意味のある名前を使います（例: `pre:bash:my-check`）。

8. **stderr にはプレフィックスを付ける**: `[HookName] エラーメッセージ` 形式でログを出力します。デバッグ時にどのフックからの出力かを識別しやすくなります。

9. **stdin を切り詰める**: 大きな入力に備えて `MAX_STDIN = 1024 * 1024` のような上限を設定し、メモリを使い果たさないようにします。

10. **PreToolUse の `additionalContext` を活用する**: ブロックせずに情報を提供したい場合は `additionalContext` フィールドを返します。Claude はこれを次のステップで参照します（警告・ヒント・チェック結果など）。

---

## 実践: フック作成の流れ

1. `scripts/hooks/` に `my-hook.js` を作成
2. `module.exports = { run }` を実装
3. `run(rawInput, options)` でJSONパース → 処理 → 結果返却
4. `hooks/hooks.json` にエントリを追加（`run-with-flags.js` 経由）
5. `node tests/run-all.js` でテスト実行
6. `node scripts/ci/validate-hooks.js` でスキーマ検証

詳細な実装手順は [12-authoring-components.md](./12-authoring-components.md) を参照してください。

---

## フック設計の思想・ベストプラクティス

### Claude Code がフックに込めた「決定論的 enforcement」の思想

Claude Code がフックシステムを持つ理由は一言で言えば「**LLM に任せてはいけない保証をハーネスが担う**」ためです。コーディング規約の遵守・品質チェック・セキュリティゲートを LLM への指示だけで実現しようとすると、文脈が長くなったとき・モデルが変わったとき・予期しない会話の流れのときに失敗します。フックはこれをハーネス側で決定論的に処理することで、LLM の状態に依存しない信頼性を提供します。

これは「LLM を信頼しない」という意味ではありません。LLM が得意なこと（理解・生成・判断）はプロンプトとスキルに任せ、LLM が苦手なこと（副作用の保証・外部状態の確認）はフックに任せる、という**責務分離**の思想です。

### ブロッキングフックは高速に保つ（< 200ms）

`PreToolUse` のブロッキングフック（exit code 2 でツールを止めるフック）は、Claude の応答の**ホットパス**に乗ります。ユーザーがツール実行を待っている間にフックが走るため、遅いフックは体験を著しく損ないます。

- ネットワーク呼び出しは避ける（DNS 解決だけで 50ms 以上かかることがある）
- ファイルの全読み込みは避ける（数 MB のファイルを読むと数百 ms になる）
- 複雑な処理は `async: true` で非同期フックに移す

ECC では `bash-hook-dispatcher.js` を使って複数の Bash 前チェックを単一プロセスで連鎖させることで、子プロセス起動コスト（50〜100ms/回）を最小化しています。

### 非致命的エラーは exit 0 でフェイルオープンにする

フックスクリプトが予期しない入力（JSON パースエラー・ファイル不在など）で例外を投げたとき、**exit code 0 で終了する**のがベストプラクティスです。フックのバグがツール実行全体をブロックしてしまう事態を防ぎます。

```javascript
// ベストプラクティス: エラーは stderr に記録して exit 0
try {
  const input = JSON.parse(rawInput);
  // ... 処理 ...
} catch (err) {
  process.stderr.write(`[MyHook] パースエラー: ${err.message}\n`);
  process.exit(0); // フェイルオープン
}
```

これは「エラーを無視する」ではなく「フックの失敗でユーザーの作業を止めない」という設計判断です。エラーは stderr に `[HookName]` プレフィックスを付けて記録することで、デバッグ時に発見できます。

### matcher は具体的に指定する

`matcher: "*"` はすべてのツール呼び出しでフックを起動します。フックが軽量でも、呼び出し回数が多いほどオーバーヘッドが積み重なります。必要なツールにだけマッチさせることが重要です。

```json
// 悪い例: すべてのツールに反応（重い処理を書くと全体が遅くなる）
{ "matcher": "*", "hooks": [...] }

// 良い例: Bash と Write/Edit にのみ反応
{ "matcher": "Bash", "hooks": [...] }
{ "matcher": "Edit|Write", "hooks": [...] }
```

ECC の `pre:config-protection` フックは `Write|Edit|MultiEdit` にのみマッチさせることで、読み取り系ツール（Bash での cat など）では不要なチェックをスキップしています。

### `run()` ラッパーとプロファイルで構成可能にする

ECC の `run-with-flags.js` ラッパーが提供する設計パターンは、フックシステムを**環境に応じて調整可能（configurable）**にします。

```bash
# フックの粒度をプロファイルで制御
export ECC_HOOK_PROFILE=minimal   # 最小限
export ECC_HOOK_PROFILE=standard  # 標準（デフォルト）
export ECC_HOOK_PROFILE=strict    # 最大限

# 特定フックだけを無効化
export ECC_DISABLED_HOOKS=pre:bash:commit-quality,pre:bash:tmux-reminder
```

フックスクリプトに `module.exports = { run }` を実装すると、`run-with-flags.js` がインプロセスで呼び出せるようになり、子プロセス起動を省略できます。これはパフォーマンスと同時に、テスト容易性も向上させます（Node.js の `require()` でフックをユニットテストできる）。

他のプロジェクトでも通用する設計指針としては、「フックの on/off を環境変数で制御できるようにしておく」ことが重要です。CI 環境と開発環境ではフックの必要性が異なることが多く、柔軟な制御手段がないとフックが邪魔になります。

### 可観測性: フックの動きを見えるようにする

フックは自動実行されるため、「いつ何が起きたか」を確認する手段が必要です。

**ECC が実装している可観測性の手段**:

- `stderr` への `[HookName]` プレフィックス付きログ — 問題の原因フックを特定しやすい
- `post:session-activity-tracker` — セッション内のツール使用を集計
- `stop:cost-tracker` — トークン・コストを記録
- `post:ecc-context-monitor` — コンテキスト枯渇・ループを警告
- `stop:evaluate-session` — セッションパターンを抽出して学習に活かす

他のプロジェクトでフックを書く場合も、最低限「フックが起動された」「ブロックした」「エラーが起きた」を stderr に記録することを推奨します。

### まとめ: フック設計のチェックリスト

| チェック項目 | 理由 |
|------------|------|
| ブロッキングフックは 200ms 以内か | 体験を損なわないため |
| 非致命的エラーで exit 0 しているか | フックのバグが作業を止めないため |
| matcher は最小限のツールに絞っているか | 不要なオーバーヘッドを避けるため |
| `module.exports = { run }` を実装しているか | インプロセス実行・テスト容易性のため |
| プロファイル（profilesCsv）を指定しているか | 環境に応じた on/off 制御のため |
| stderr に `[HookName]` プレフィックスを付けているか | 可観測性・デバッグのため |
| 非同期でよい処理に `async: true` を使っているか | Claude の応答速度を守るため |

---

## 関連章

- [03-components-overview.md](./03-components-overview.md) — フックを含むコンポーネント全体像
- [08-sessions-context-tokens.md](./08-sessions-context-tokens.md) — SessionStart/Stop フックによるセッション永続化
- [11-quality-ci-testing.md](./11-quality-ci-testing.md) — フックのテストと validate-hooks.js
- [12-authoring-components.md](./12-authoring-components.md) — フックの作成手順詳細

## 参照ソース

- `hooks/hooks.json` — 登録されているすべてのフック定義
- `schemas/hooks.schema.json` — フック設定の JSON スキーマ
- `scripts/hooks/run-with-flags.js` — フック実行フレームワーク
- `scripts/lib/hook-flags.js` — プロファイル・無効化制御
- `scripts/hooks/bash-hook-dispatcher.js` — Bash フック連鎖ディスパッチャ
- `scripts/hooks/pretooluse-visible-output.js` — additionalContext ビルダー
- `scripts/hooks/doc-file-warning.js` — run() パターンの実装例
- `scripts/hooks/session-start.js` — SessionStart フックの実装
- `scripts/lib/package-manager.js` — パッケージマネージャ検出
- `rules/common/hooks.md` — フック利用ガイドライン
- `scripts/ci/validate-hooks.js` — フック設定バリデーター
