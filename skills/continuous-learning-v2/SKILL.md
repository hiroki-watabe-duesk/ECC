---
name: continuous-learning-v2
description: フック経由でセッションを観察し、信頼度スコア付きの原子的インスティンクトを生成・進化させるインスティンクトベースの学習システム。v2.1ではプロジェクトスコープのインスティンクトを追加してプロジェクト間の汚染を防止する。
origin: ECC
version: 2.1.0
---

# 継続学習 v2.1 - インスティンクトベースアーキテクチャ

Claude Codeのセッションを、信頼度スコア付きの小さな学習済み行動「インスティンクト」を通じて再利用可能な知識へと変換する高度な学習システムです。

**v2.1** では **プロジェクトスコープのインスティンクト** が追加されました。ReactのパターンはあなたのReactプロジェクトにとどまりPythonの規約はPythonプロジェクトにとどまり、「常に入力を検証する」といった汎用パターンのみがグローバルに共有されます。

## いつ起動するか

- Claude Codeセッションからの自動学習を設定するとき
- フック経由のインスティンクトベース行動抽出を設定するとき
- 学習済み行動の信頼度閾値を調整するとき
- インスティンクトライブラリのレビュー・エクスポート・インポートをするとき
- インスティンクトをスキル、コマンド、またはエージェントへ進化させるとき
- プロジェクトスコープのインスティンクトとグローバルインスティンクトを管理するとき
- インスティンクトをプロジェクトからグローバルスコープへプロモートするとき

## v2.1 の新機能

| 機能 | v2.0 | v2.1 |
|---------|------|------|
| ストレージ | グローバル（`~/.claude/homunculus/`） | プロジェクトスコープ（`${XDG_DATA_HOME:-~/.local/share}/ecc-homunculus/projects/<hash>/`） |
| スコープ | 全インスティンクトがどこでも適用 | プロジェクトスコープ＋グローバル |
| 検出 | なし | git リモートURL / リポジトリパス |
| プロモート | N/A | 2プロジェクト以上で見られた場合、プロジェクト→グローバル |
| コマンド | 4つ（status/evolve/export/import） | 6つ（+promote/projects） |
| プロジェクト間 | 汚染リスクあり | デフォルトで分離 |

## v2 の新機能（v1 との比較）

| 機能 | v1 | v2 |
|---------|----|----|
| 観察 | Stopフック（セッション終了） | PreToolUse/PostToolUse（100%信頼性） |
| 分析 | メインコンテキスト | バックグラウンドエージェント（Haiku） |
| 粒度 | フルスキル | 原子的「インスティンクト」 |
| 信頼度 | なし | 0.3〜0.9の重み付け |
| 進化 | スキルへ直接 | インスティンクト → クラスター → スキル/コマンド/エージェント |
| 共有 | なし | インスティンクトのエクスポート/インポート |

## インスティンクトモデル

インスティンクトは小さな学習済み行動です：

```yaml
---
id: prefer-functional-style
trigger: "when writing new functions"
confidence: 0.7
domain: "code-style"
source: "session-observation"
scope: project
project_id: "a1b2c3d4e5f6"
project_name: "my-react-app"
---

# Prefer Functional Style

## Action
Use functional patterns over classes when appropriate.

## Evidence
- Observed 5 instances of functional pattern preference
- User corrected class-based approach to functional on 2025-01-15
```

**プロパティ：**
- **原子的** — トリガーひとつ、アクションひとつ
- **信頼度重み付き** — 0.3＝暫定的、0.9＝ほぼ確実
- **ドメインタグ付き** — code-style、testing、git、debugging、workflow など
- **エビデンスに基づく** — どの観察が生成したかを追跡
- **スコープ対応** — `project`（デフォルト）または `global`

## 動作の仕組み

```
セッションアクティビティ（gitリポジトリ内）
      |
      | フックがプロンプト＋ツール使用をキャプチャ（100%信頼性）
      | ＋プロジェクトコンテキストを検出（git リモート / リポジトリパス）
      v
+---------------------------------------------+
|  projects/<project-hash>/observations.jsonl  |
|   (プロンプト、ツールコール、結果、プロジェクト)  |
+---------------------------------------------+
      |
      | オブザーバーエージェントが読み取る（バックグラウンド、Haiku）
      v
+---------------------------------------------+
|          パターン検出                         |
|   * ユーザーの修正 → インスティンクト          |
|   * エラー解決 → インスティンクト              |
|   * 繰り返しワークフロー → インスティンクト     |
|   * スコープ決定：project か global か？      |
+---------------------------------------------+
      |
      | 作成/更新
      v
+---------------------------------------------+
|  projects/<project-hash>/instincts/personal/ |
|   * prefer-functional.yaml (0.7) [project]   |
|   * use-react-hooks.yaml (0.9) [project]     |
+---------------------------------------------+
|  instincts/personal/  (グローバル)            |
|   * always-validate-input.yaml (0.85) [global]|
|   * grep-before-edit.yaml (0.6) [global]     |
+---------------------------------------------+
      |
      | /evolve クラスター＋/promote
      v
+---------------------------------------------+
|  projects/<hash>/evolved/ (プロジェクトスコープ)|
|  evolved/ (グローバル)                        |
|   * commands/new-feature.md                  |
|   * skills/testing-workflow.md               |
|   * agents/refactor-specialist.md            |
+---------------------------------------------+
```

## プロジェクト検出

システムは現在のプロジェクトを自動的に検出します：

1. **`CLAUDE_PROJECT_DIR` 環境変数**（最高優先度）
2. **`git remote get-url origin`** — ハッシュ化してポータブルなプロジェクトIDを生成（異なるマシン上の同一リポジトリは同じIDになります）
3. **`git rev-parse --show-toplevel`** — リポジトリパスを使用したフォールバック（マシン固有）
4. **グローバルフォールバック** — プロジェクトが検出されない場合、インスティンクトはグローバルスコープへ

各プロジェクトには12文字のハッシュID（例：`a1b2c3d4e5f6`）が付与されます。`${XDG_DATA_HOME:-~/.local/share}/ecc-homunculus/projects.json` のレジストリファイルがIDと人間が読みやすい名前をマッピングします。

### データディレクトリ

continuous-learning-v2は、Claude Codeの機密パスガードがバックグラウンドのインスティンクト書き込みをブロックしないよう、`~/.claude`の外にオブザーバーデータを保存します：

1. `CLV2_HOMUNCULUS_DIR` が絶対パスに設定されている場合
2. `$XDG_DATA_HOME/ecc-homunculus`
3. `$HOME/.local/share/ecc-homunculus`

`~/.claude/homunculus`にデータを持つ既存ユーザーは一度だけ移行できます：

```bash
bash skills/continuous-learning-v2/scripts/migrate-homunculus.sh
```

## クイックスタート

### 1. 観察フックを有効化する

**プラグインとしてインストールした場合**（推奨）：

追加の `settings.json` フックブロックは不要です。Claude Code v2.1+はプラグインの `hooks/hooks.json` を自動ロードし、`observe.sh` はすでにそこに登録されています。

以前に `observe.sh` を `~/.claude/settings.json` にコピーした場合は、その重複した `PreToolUse` / `PostToolUse` ブロックを削除してください。プラグインフックを複製すると二重実行が発生し、`${CLAUDE_PLUGIN_ROOT}` の解決エラーが起きます（この変数はプラグイン管理の `hooks/hooks.json` エントリ内でのみ利用可能です）。

**手動で** `~/.claude/skills` にインストールした場合は、`~/.claude/settings.json` に以下を追加してください：

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "*",
      "hooks": [{
        "type": "command",
        "command": "~/.claude/skills/continuous-learning-v2/hooks/observe.sh"
      }]
    }],
    "PostToolUse": [{
      "matcher": "*",
      "hooks": [{
        "type": "command",
        "command": "~/.claude/skills/continuous-learning-v2/hooks/observe.sh"
      }]
    }]
  }
}
```

### 2. ディレクトリ構造を初期化する

システムは初回使用時に自動的にディレクトリを作成しますが、手動で作成することもできます：

```bash
# グローバルディレクトリ
mkdir -p "${XDG_DATA_HOME:-$HOME/.local/share}/ecc-homunculus"/{instincts/{personal,inherited},evolved/{agents,skills,commands},projects}

# プロジェクトディレクトリはgitリポジトリでフックが最初に実行されたときに自動作成される
```

### 3. インスティンクトコマンドを使用する

```bash
/instinct-status     # 学習済みインスティンクト（プロジェクト＋グローバル）を表示
/evolve              # 関連するインスティンクトをスキル/コマンドへクラスター化
/instinct-export     # インスティンクトをファイルへエクスポート
/instinct-import     # 他者からインスティンクトをインポート
/promote             # プロジェクトインスティンクトをグローバルスコープへプロモート
/projects            # 全既知プロジェクトとインスティンクト数を一覧表示
```

## コマンド

| コマンド | 説明 |
|---------|-------------|
| `/instinct-status` | 全インスティンクト（プロジェクトスコープ＋グローバル）を信頼度とともに表示 |
| `/evolve` | 関連するインスティンクトをスキル/コマンドへクラスター化し、プロモート候補を提案 |
| `/instinct-export` | インスティンクトをエクスポート（スコープ/ドメインでフィルタリング可） |
| `/instinct-import <file>` | スコープ制御付きでインスティンクトをインポート |
| `/promote [id]` | プロジェクトインスティンクトをグローバルスコープへプロモート |
| `/projects` | 全既知プロジェクトとインスティンクト数を一覧表示 |

## 設定

`config.json` を編集してバックグラウンドオブザーバーを制御します：

```json
{
  "version": "2.1",
  "observer": {
    "enabled": false,
    "run_interval_minutes": 5,
    "min_observations_to_analyze": 20
  }
}
```

| キー | デフォルト | 説明 |
|-----|---------|-------------|
| `observer.enabled` | `false` | バックグラウンドオブザーバーエージェントを有効化 |
| `observer.run_interval_minutes` | `5` | オブザーバーが観察を分析する間隔 |
| `observer.min_observations_to_analyze` | `20` | 分析を実行するための最低観察数 |

その他の動作（観察キャプチャ、インスティンクト閾値、プロジェクトスコープ、プロモート基準）は `instinct-cli.py` と `observe.sh` のコードデフォルトで設定されます。

## ファイル構造

```
${XDG_DATA_HOME:-~/.local/share}/ecc-homunculus/
+-- identity.json           # あなたのプロフィール、技術レベル
+-- projects.json           # レジストリ：プロジェクトハッシュ → 名前/パス/リモート
+-- observations.jsonl      # グローバル観察（フォールバック）
+-- instincts/
|   +-- personal/           # グローバル自動学習インスティンクト
|   +-- inherited/          # グローバルインポートインスティンクト
+-- evolved/
|   +-- agents/             # グローバル生成エージェント
|   +-- skills/             # グローバル生成スキル
|   +-- commands/           # グローバル生成コマンド
+-- projects/
    +-- a1b2c3d4e5f6/       # プロジェクトハッシュ（git リモートURLから）
    |   +-- project.json    # プロジェクトごとのメタデータミラー（id/name/root/remote）
    |   +-- observations.jsonl
    |   +-- observations.archive/
    |   +-- instincts/
    |   |   +-- personal/   # プロジェクト固有の自動学習
    |   |   +-- inherited/  # プロジェクト固有のインポート
    |   +-- evolved/
    |       +-- skills/
    |       +-- commands/
    |       +-- agents/
    +-- f6e5d4c3b2a1/       # 別のプロジェクト
        +-- ...
```

## スコープ決定ガイド

| パターンタイプ | スコープ | 例 |
|-------------|-------|---------|
| 言語/フレームワーク規約 | **project** | 「Reactフックを使う」「Django RESTパターンに従う」 |
| ファイル構造の好み | **project** | 「`__tests__`/にテストを置く」「src/components/にコンポーネントを置く」 |
| コードスタイル | **project** | 「関数型スタイルを使う」「データクラスを優先する」 |
| エラー処理戦略 | **project** | 「エラーにResult型を使う」 |
| セキュリティプラクティス | **global** | 「ユーザー入力を検証する」「SQLをサニタイズする」 |
| 一般的なベストプラクティス | **global** | 「テストを先に書く」「常にエラーを処理する」 |
| ツールワークフローの好み | **global** | 「編集前にGrepする」「書く前に読む」 |
| Gitプラクティス | **global** | 「コンベンショナルコミット」「小さく集中したコミット」 |

## インスティンクトプロモート（プロジェクト → グローバル）

同じインスティンクトが複数のプロジェクトで高信頼度で現れた場合、グローバルスコープへのプロモート候補になります。

**自動プロモート基準：**
- 2つ以上のプロジェクトで同じインスティンクトID
- 平均信頼度が0.8以上

**プロモート方法：**

```bash
# 特定のインスティンクトをプロモート
python3 instinct-cli.py promote prefer-explicit-errors

# 条件を満たす全インスティンクトを自動プロモート
python3 instinct-cli.py promote

# 変更なしでプレビュー
python3 instinct-cli.py promote --dry-run
```

`/evolve` コマンドもプロモート候補を提案します。

## 信頼度スコアリング

信頼度は時間とともに変化します：

| スコア | 意味 | 動作 |
|-------|---------|----------|
| 0.3 | 暫定的 | 提案されるが強制はされない |
| 0.5 | 中程度 | 関連する場合に適用 |
| 0.7 | 強い | 適用のため自動承認 |
| 0.9 | ほぼ確実 | コア動作 |

**信頼度が上がる条件：**
- パターンが繰り返し観察される
- ユーザーが提案された行動を修正しない
- 他のソースからの類似インスティンクトが一致する

**信頼度が下がる条件：**
- ユーザーが明示的に動作を修正する
- 長期間パターンが観察されない
- 矛盾するエビデンスが現れる

## 観察にフックを使う理由（スキルでなく）

> 「v1はスキルに観察を依存していました。スキルは確率的で、Claudeの判断に基づいて50〜80%程度の確率で起動します。」

フックは**100%確実**に、決定論的に起動します。つまり：
- 全ツールコールが観察される
- パターンが見逃されない
- 学習が網羅的になる

## 後方互換性

v2.1はv2.0およびv1と完全に互換です：
- 既存のグローバルインスティンクトは `scripts/migrate-homunculus.sh` で `~/.claude/homunculus/instincts/` から移行可能
- v1から `~/.claude/skills/learned/` にある既存スキルは引き続き動作
- Stopフックは引き続き実行される（ただしv2へもフィード）
- 段階的な移行：両方を並行して実行可能

## プライバシー

- 観察はあなたのマシン上で**ローカル**に保持される
- プロジェクトスコープのインスティンクトはプロジェクトごとに分離
- エクスポートできるのは**インスティンクト**（パターン）のみ — 生の観察ではない
- 実際のコードや会話内容は共有されない
- エクスポートおよびプロモートする内容はあなたが制御

## 関連

- [ECC-Tools GitHub App](https://github.com/apps/ecc-tools) - リポジトリ履歴からインスティンクトを生成
- Homunculus - v2インスティンクトベースアーキテクチャ（原子的観察、信頼度スコアリング、インスティンクト進化パイプライン）にインスピレーションを与えたコミュニティプロジェクト
- [The Longform Guide](https://x.com/affaanmustafa/status/2014040193557471352) - 継続学習セクション

---

*インスティンクトベース学習：一度に一プロジェクトずつ、Claudeにあなたのパターンを教える。*
