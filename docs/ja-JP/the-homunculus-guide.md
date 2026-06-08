# ホムンクルス（Homunculus）の仕組みガイド

> Everything Claude Code（ECC）の自己学習システム「ホムンクルス」を、概念から内部構造・ライフサイクル・運用までまとめたガイドです。

---

## ホムンクルスとは

**ホムンクルス（Homunculus）** とは、Claude Code の「自己モデル領域」を指します。あなたの Claude Code セッションから抽出された学習（instinct = 本能）と、それらを束ねて生成された evolved コンポーネント（skill / command / agent）を格納する場所です。

この仕組みは **Continuous Learning v2（continuous-learning-v2 スキル）** として実装されています。名前の「ホムンクルス」は、原子的な観測・信頼度スコアリング・本能の進化パイプラインという設計に影響を与えたコミュニティプロジェクトに由来します。

一言でいうと、ホムンクルスは次のサイクルを回す自己改善システムです。

1. **観測（Observe）** — すべてのツール使用をフックで 100% 確実に記録する
2. **抽出（Extract）** — バックグラウンドの Observer エージェント（Haiku）が観測からパターンを取り出し、原子的な「本能（instinct）」を作る
3. **保存（Store）** — 本能をプロジェクト単位／グローバルにスコープ分けして保存する
4. **昇格（Promote）** — 複数プロジェクトで実証された普遍的なパターンをグローバルへ昇格させる
5. **進化（Evolve）** — 関連する本能をクラスタリングし、再利用可能な skill / command / agent に蒸留する
6. **注入（Inject）** — セッション開始時に有効な本能をコンテキストへ差し込み、その場の判断に効かせる

---

## 中核概念：本能（Instinct）

**本能（Instinct）** とは、セッションから抽出された小さな学習行動です。YAML フロントマター＋ `## Action` セクションで構成される 1 ファイルが 1 つの本能に対応します。

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

本能の性質：

- **原子的（Atomic）** — 1 トリガー・1 アクション
- **信頼度付き（Confidence-weighted）** — 0.3 = 暫定、0.9 = ほぼ確実
- **ドメインタグ付き** — `code-style` / `testing` / `git` / `debugging` / `workflow` など
- **根拠付き（Evidence-backed）** — どの観測から生まれたかを記録
- **スコープ対応** — `project`（既定）または `global`

---

## 全体像：4 ステージのライフサイクル

```
git リポジトリ内でのセッション活動
      |
      | フックがプロンプト＋ツール使用を捕捉（100% 確実）
      | ＋プロジェクト文脈を検出（git remote / リポジトリパス）
      v
+---------------------------------------------+
|  projects/<project-hash>/observations.jsonl  |
|   （プロンプト・ツール呼び出し・結果・プロジェクト）  |
+---------------------------------------------+
      |
      | Observer エージェントが読む（バックグラウンド・Haiku）
      v
+---------------------------------------------+
|          パターン検出                          |
|   * ユーザーの訂正 -> 本能                       |
|   * エラー解決 -> 本能                           |
|   * 繰り返しのワークフロー -> 本能                 |
|   * スコープ判断：project か global か？          |
+---------------------------------------------+
      |
      | 作成・更新
      v
+---------------------------------------------+
|  projects/<project-hash>/instincts/personal/ |
|   * prefer-functional.yaml (0.7) [project]   |
|   * use-react-hooks.yaml (0.9) [project]     |
+---------------------------------------------+
|  instincts/personal/  （グローバル）            |
|   * always-validate-input.yaml (0.85) [global]|
|   * grep-before-edit.yaml (0.6) [global]     |
+---------------------------------------------+
      |
      | /evolve でクラスタリング + /promote で昇格
      v
+---------------------------------------------+
|  projects/<hash>/evolved/ （プロジェクト単位）    |
|  evolved/ （グローバル）                         |
|   * commands/new-feature.md                  |
|   * skills/testing-workflow.md               |
|   * agents/refactor-specialist.md            |
+---------------------------------------------+
```

### ステージ 1：観測（フック）

`PreToolUse` / `PostToolUse` フックが、Claude がツール（Bash / Edit / Read など）を使うたびに発火し、イベントを `observations.jsonl` に追記します。

- スキルが「確率的」に発火する（Claude の判断で約 50〜80% しか起動しない）のに対し、**フックは 100% 確実に・決定論的に**発火します。これにより、すべてのツール呼び出しが観測され、パターンを取りこぼしません。
- プラグインとしてインストールしている場合、Claude Code v2.1+ がプラグインの `hooks/hooks.json` を自動ロードし、`observe.sh` が登録済みのため、`settings.json` への追記は不要です。
  - 過去に `observe.sh` を `~/.claude/settings.json` へコピーしていた場合は、その重複ブロックを削除してください。プラグインフックとの二重登録は二重実行や `${CLAUDE_PLUGIN_ROOT}` の解決エラーを招きます。

### ステージ 2：本能の生成（Observer エージェント）

バックグラウンドの Observer エージェント（Haiku）が `observations.jsonl` を読み、パターンを検出して本能を作成・更新します。検出対象は次のようなものです。

- **ユーザーの訂正** — 「いや、X を使って」 → 本能化
- **エラー解決** — エラー → 修正のパターン
- **繰り返されるワークフロー** — 同じツール列の反復
- **ツールの好み** — 「Edit の前に必ず Grep」

Observer の動作は `config.json` で制御します（既定では `enabled: false` のため、利用するには有効化が必要です）。

### ステージ 3：スコープ判断と昇格（Promote）

v2.1 の核心は **プロジェクト単位のスコープ** です。React のパターンは React プロジェクトに、Python の規約は Python プロジェクトに留め、普遍的なパターン（例：「常に入力を検証する」）だけをグローバルで共有します。これによりプロジェクト間の「汚染」を防ぎます。

| パターンの種類 | スコープ | 例 |
|---|---|---|
| 言語・フレームワークの規約 | **project** | 「React Hooks を使う」「Django REST のパターンに従う」 |
| ファイル構成の好み | **project** | 「テストは `__tests__/`」「コンポーネントは `src/components/`」 |
| コードスタイル | **project** | 「関数型スタイル」「dataclass を優先」 |
| エラー処理方針 | **project** | 「エラーには Result 型」 |
| セキュリティ実践 | **global** | 「ユーザー入力を検証」「SQL をサニタイズ」 |
| 一般的ベストプラクティス | **global** | 「テストを先に書く」「常にエラーを処理」 |
| ツールのワークフローの好み | **global** | 「Edit の前に Grep」「Write の前に Read」 |
| Git の実践 | **global** | 「Conventional Commits」「小さく焦点を絞ったコミット」 |

**昇格（Project → Global）** は、同じ本能が複数プロジェクトで高い信頼度を獲得したときに行います。

- 自動昇格の条件：**同一の本能 ID が 2 つ以上のプロジェクトに存在**し、**平均信頼度 >= 0.8**
- `/evolve` も昇格候補を提案します

```bash
# 特定の本能を昇格
python3 instinct-cli.py promote prefer-explicit-errors

# 条件を満たすものをすべて自動昇格
python3 instinct-cli.py promote

# 変更せずプレビュー
python3 instinct-cli.py promote --dry-run
```

### ステージ 4：進化（Evolve）

`/evolve` は、同じトリガーを持つ複数の本能をクラスタリングし、再利用可能な構造へ蒸留します。提案される種別は次の通りです。

- **Command** — ユーザーが呼び出すアクション（関連する本能が 2 つ以上）
- **Skill** — 自動的に発火する振る舞い
- **Agent** — 複雑な複数ステップのプロセス

生成物（evolved）は、プロジェクト単位なら `projects/<hash>/evolved/`、グローバルなら `evolved/` に書き込まれます。evolved は `learned`（`/learn` で生成）や `imported`（手動配置）とは別系統のシステムであり、出自（provenance）はクラスタ元の本能から継承されるため、個別の `.provenance.json` は不要です。

---

## データの保存場所とディレクトリ構成

continuous-learning-v2 は、Claude Code の機微パス（sensitive-path）ガードがバックグラウンドの本能書き込みをブロックしないよう、データを **`~/.claude` の外** に保存します。解決の優先順位は次の通りです。

1. `CLV2_HOMUNCULUS_DIR`（絶対パスで指定された場合）
2. `$XDG_DATA_HOME/ecc-homunculus`
3. `$HOME/.local/share/ecc-homunculus`（既定）

> 補足：v2.0 までは `~/.claude/homunculus/` に保存していました。既存ユーザーは `bash skills/continuous-learning-v2/scripts/migrate-homunculus.sh` で一度だけ移行できます。

```
${XDG_DATA_HOME:-~/.local/share}/ecc-homunculus/
├── identity.json           # ユーザープロファイル・技術レベル
├── projects.json           # レジストリ：プロジェクトハッシュ -> 名前/パス/remote
├── observations.jsonl      # グローバル観測（フォールバック）
├── instincts/
│   ├── personal/           # グローバルの自動学習本能
│   └── inherited/          # グローバルのインポート本能
├── evolved/
│   ├── agents/             # グローバル生成エージェント
│   ├── skills/             # グローバル生成スキル
│   └── commands/           # グローバル生成コマンド
└── projects/
    ├── a1b2c3d4e5f6/       # プロジェクトハッシュ（git remote URL 由来）
    │   ├── project.json    # プロジェクトメタデータの控え（id/name/root/remote）
    │   ├── observations.jsonl
    │   ├── observations.archive/
    │   ├── instincts/
    │   │   ├── personal/   # プロジェクト固有の自動学習
    │   │   └── inherited/  # プロジェクト固有のインポート
    │   └── evolved/
    │       ├── skills/
    │       ├── commands/
    │       └── agents/
    └── f6e5d4c3b2a1/       # 別のプロジェクト
        └── ...
```

---

## プロジェクトの自動検出

現在のプロジェクトは次の優先順位で判定されます。

1. **`CLAUDE_PROJECT_DIR` 環境変数**（最優先）
2. **`git remote get-url origin`** — ハッシュ化してポータブルなプロジェクト ID を生成（同じリポジトリなら別マシンでも同じ ID）
3. **`git rev-parse --show-toplevel`** — リポジトリパスによるフォールバック（マシン依存）
4. **グローバルフォールバック** — プロジェクトを検出できない場合は global スコープへ

各プロジェクトには 12 文字のハッシュ ID（例：`a1b2c3d4e5f6`）が割り当てられ、`projects.json` レジストリが ID と人間可読な名前を対応付けます。

---

## セッション開始時の注入

セッション開始フック（`scripts/hooks/session-start.js`）は、有効な本能をその場のコンテキストへ差し込みます。

- プロジェクト／グローバル両方の `instincts/personal` と `instincts/inherited` を読み込む
- **信頼度 0.7 未満は除外**（`INSTINCT_CONFIDENCE_THRESHOLD = 0.7`）
- ID で重複排除し、同一 ID ではプロジェクトスコープを優先
- 信頼度の降順で並べ、**最大 6 件**（`MAX_INJECTED_INSTINCTS = 6`）を注入

注入例：

```
Active instincts:
- [project 92%] Always validate user input
- [global 75%] Grep before editing
```

---

## 信頼度スコアリング

信頼度は時間とともに変化します。

| スコア | 意味 | 振る舞い |
|---|---|---|
| 0.3 | 暫定 | 提案するが強制しない |
| 0.5 | 中程度 | 関連する場面で適用 |
| 0.7 | 強い | 適用を自動承認 |
| 0.9 | ほぼ確実 | 中核的な振る舞い |

**上昇する**条件：パターンが繰り返し観測される／提案した振る舞いをユーザーが訂正しない／他ソースの類似本能が一致する。

**下降する**条件：ユーザーが明示的に訂正する／長期間そのパターンが観測されない／矛盾する根拠が現れる。

---

## コマンド一覧

| コマンド | 説明 |
|---|---|
| `/instinct-status` | すべての本能（プロジェクト＋グローバル）を信頼度付きで表示 |
| `/evolve` | 関連する本能をクラスタリングし skill/command を提案・昇格候補を提示 |
| `/instinct-export` | 本能をエクスポート（scope/domain でフィルタ可能） |
| `/instinct-import <file>` | スコープ指定付きで本能をインポート |
| `/promote [id]` | プロジェクトの本能をグローバルへ昇格 |
| `/projects` | 既知のプロジェクトと本能数の一覧 |

---

## 設定

### config.json（バックグラウンド Observer の制御）

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

| キー | 既定 | 説明 |
|---|---|---|
| `observer.enabled` | `false` | バックグラウンド Observer エージェントを有効化 |
| `observer.run_interval_minutes` | `5` | Observer が観測を分析する間隔 |
| `observer.min_observations_to_analyze` | `20` | 分析を実行する前に必要な最小観測数 |

その他の挙動（観測の捕捉、本能のしきい値、プロジェクトスコープ、昇格条件）は `instinct-cli.py` と `observe.sh` のコード既定値で決まります。

### 環境変数

| 変数 | 既定 | 用途 |
|---|---|---|
| `CLV2_HOMUNCULUS_DIR` | — | ホムンクルスディレクトリを上書き（絶対パス必須） |
| `XDG_DATA_HOME` | `~/.local/share` | XDG Base Directory 準拠の保存先 |
| `CLAUDE_PROJECT_DIR` | — | プロジェクト検出を上書き（最優先） |
| `CLV2_PYTHON_CMD` | `python3` / `python` | Python インタプリタを上書き |

---

## v1 → v2 → v2.1 の進化と互換性

| 観点 | v1 | v2 | v2.1 |
|---|---|---|---|
| 観測方法 | Stop フック（セッション終了時） | PreToolUse/PostToolUse（100% 確実） | 同左 |
| 分析 | メインコンテキスト | バックグラウンドエージェント（Haiku） | 同左 |
| 粒度 | フルスキル | 原子的な「本能」 | 同左 |
| 信頼度 | なし | 0.3〜0.9 で重み付け | 同左 |
| 進化 | 直接スキル化 | 本能 → クラスタ → skill/command/agent | 同左 |
| 保存 | — | グローバル（`~/.claude/homunculus/`） | プロジェクト単位（`ecc-homunculus/projects/<hash>/`） |
| スコープ | — | すべての本能がどこでも適用 | プロジェクト単位＋グローバル |
| 昇格 | — | なし | 2 つ以上のプロジェクトで観測されたら昇格 |

v2.1 は v2.0・v1 と完全互換です。

- 既存のグローバル本能は `scripts/migrate-homunculus.sh` で `~/.claude/homunculus/instincts/` から移行可能
- v1 の `~/.claude/skills/learned/` スキルは引き続き動作
- Stop フックも引き続き動作し、v2 へも供給される
- 段階的移行：両方を並行運用できる

---

## プライバシー

- 観測データはあなたのマシン上に **ローカル** に留まる
- プロジェクト単位の本能はプロジェクトごとに分離される
- エクスポートできるのは **本能（パターン）のみ** で、生の観測データは対象外
- 実際のコードや会話内容は共有されない
- 何をエクスポート・昇格するかはあなたが制御する

---

## 関連

- [continuous-learning-v2 SKILL.md](./skills/continuous-learning-v2/SKILL.md) — 実装の一次情報
- `/instinct-status`・`/evolve`・`/promote`・`/projects` — 本能ライフサイクルの操作コマンド
- 用語集（GLOSSARY） — 「インスティンクト（Instinct）」「ホムンクルス（Homunculus）」の定義

---

*本能ベースの学習：あなたのパターンを、プロジェクトごとに 1 つずつ Claude に教えていく。*
