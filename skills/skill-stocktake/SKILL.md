---
name: skill-stocktake
description: "品質のためにClaudeスキルとコマンドを監査する際に使用する。変更されたスキルのみのクイックスキャンと、順次サブエージェントバッチ評価を使ったフルストックテイクモードをサポートする。"
origin: ECC
---

# skill-stocktake

品質チェックリスト＋AIによる総合的な判断を使って、すべてのClaudeスキルとコマンドを監査するスラッシュコマンド（`/skill-stocktake`）。最近変更されたスキルのクイックスキャンと、完全なレビューのフルストックテイクの2つのモードをサポートします。

## スコープ

このコマンドは**実行されたディレクトリを基準とした**以下のパスを対象とします:

| パス | 説明 |
|------|-------------|
| `~/.claude/skills/` | グローバルスキル（全プロジェクト） |
| `{cwd}/.claude/skills/` | プロジェクトレベルのスキル（ディレクトリが存在する場合） |

**フェーズ1の開始時に、コマンドは見つかってスキャンされたパスを明示的に一覧します。**

### 特定のプロジェクトを対象にする

プロジェクトレベルのスキルを含めるには、そのプロジェクトのルートディレクトリから実行します:

```bash
cd ~/path/to/my-project
/skill-stocktake
```

プロジェクトに `.claude/skills/` ディレクトリがない場合は、グローバルスキルとコマンドのみが評価されます。

## モード

| モード | トリガー | 所要時間 |
|------|---------|---------|
| クイックスキャン | `results.json` が存在する（デフォルト） | 5〜10分 |
| フルストックテイク | `results.json` がない、または `/skill-stocktake full` | 20〜30分 |

**結果キャッシュ:** `~/.claude/skills/skill-stocktake/results.json`

## クイックスキャンフロー

前回の実行以降に変更されたスキルのみを再評価します（5〜10分）。

1. `~/.claude/skills/skill-stocktake/results.json` を読む
2. 実行: `bash ~/.claude/skills/skill-stocktake/scripts/quick-diff.sh \
         ~/.claude/skills/skill-stocktake/results.json`
   （プロジェクトディレクトリは `$PWD/.claude/skills` から自動検出；必要な場合のみ明示的に渡す）
3. 出力が `[]` の場合: 「前回の実行以降変更なし」と報告して停止する
4. 変更されたファイルのみをフェーズ2と同じ基準で再評価する
5. 変更されていないスキルは前回の結果から引き継ぐ
6. 差分のみを出力する
7. 実行: `bash ~/.claude/skills/skill-stocktake/scripts/save-results.sh \
         ~/.claude/skills/skill-stocktake/results.json <<< "$EVAL_RESULTS"`

## フルストックテイクフロー

### フェーズ1 — インベントリ

実行: `bash ~/.claude/skills/skill-stocktake/scripts/scan.sh`

スクリプトはスキルファイルを列挙し、フロントマターを抽出し、UTCのmtimeを収集します。
プロジェクトディレクトリは `$PWD/.claude/skills` から自動検出；必要な場合のみ明示的に渡す。
スクリプト出力からスキャンサマリーとインベントリテーブルを提示します:

```
スキャン中:
  ✓ ~/.claude/skills/         (17 files)
  ✗ {cwd}/.claude/skills/    (not found — global skills only)
```

| スキル | 7日使用 | 30日使用 | 説明 |
|-------|--------|---------|-------------|

### フェーズ2 — 品質評価

完全なインベントリとチェックリストを使って、Agentツールサブエージェント（**汎用エージェント**）を起動します:

```text
Agent(
  subagent_type="general-purpose",
  prompt="
Evaluate the following skill inventory against the checklist.

[INVENTORY]

[CHECKLIST]

Return JSON for each skill:
{ \"verdict\": \"Keep\"|\"Improve\"|\"Update\"|\"Retire\"|\"Merge into [X]\", \"reason\": \"...\" }
"
)
```

サブエージェントは各スキルを読み、チェックリストを適用し、スキルごとのJSONを返します:

`{ "verdict": "Keep"|"Improve"|"Update"|"Retire"|"Merge into [X]", "reason": "..." }`

**チャンクガイダンス:** コンテキストを管理しやすくするために、サブエージェント呼び出しごとに約20スキルを処理します。各チャンク後に中間結果を `results.json`（`status: "in_progress"`）に保存します。

すべてのスキルが評価された後: `status: "completed"` を設定し、フェーズ3に進みます。

**再開検出:** 起動時に `status: "in_progress"` が見つかった場合、最初の未評価スキルから再開します。

各スキルはこのチェックリストに対して評価されます:

```
- [ ] 他のスキルとのコンテンツの重複を確認済み
- [ ] MEMORY.md / CLAUDE.md との重複を確認済み
- [ ] 技術リファレンスの新鮮さを検証済み（ツール名/CLIフラグ/APIが存在する場合はWebSearchを使用）
- [ ] 使用頻度を考慮済み
```

判定基準:

| 判定 | 意味 |
|---------|---------|
| Keep | 有用かつ最新 |
| Improve | 保持する価値があるが、特定の改善が必要 |
| Update | 参照している技術が古い（WebSearchで確認する） |
| Retire | 品質が低い、古い、またはコスト非対称 |
| Merge into [X] | 別のスキルと大幅に重複；マージ先を指定する |

評価は**AIによる総合的な判断**であり、数値的なルーブリックではありません。指導的な次元:
- **実用性**: すぐに行動できるコード例、コマンド、またはステップ
- **スコープの適合**: 名前、トリガー、コンテンツが一致している；広すぎず狭すぎない
- **独自性**: MEMORY.md / CLAUDE.md / 他のスキルでは代替できない価値
- **最新性**: 技術リファレンスが現在の環境で動作する

**理由の品質要件** — `reason` フィールドは自己完結していて意思決定を可能にするものでなければなりません:
- 「unchanged」だけを書かない — 常にコアとなる根拠を再述する
- **Retire** の場合: (1)発見された具体的な欠陥、(2)同じニーズをカバーするものを述べる
  - 悪い例: `"Superseded"`
  - 良い例: `"disable-model-invocation: true already set; superseded by continuous-learning-v2 which covers all the same patterns plus confidence scoring. No unique content remains."`
- **Merge** の場合: ターゲットを指定し、統合するコンテンツを説明する
  - 悪い例: `"Overlaps with X"`
  - 良い例: `"42-line thin content; Step 4 of chatlog-to-article already covers the same workflow. Integrate the 'article angle' tip as a note in that skill."`
- **Improve** の場合: 必要な具体的な変更を説明する（どのセクション、どのアクション、関連する場合はターゲットサイズ）
  - 悪い例: `"Too long"`
  - 良い例: `"276 lines; Section 'Framework Comparison' (L80–140) duplicates ai-era-architecture-principles; delete it to reach ~150 lines."`
- **Keep**（クイックスキャンでmtimeのみ変更された場合）: 「unchanged」と書かずに元の判定根拠を再述する
  - 悪い例: `"Unchanged"`
  - 良い例: `"mtime updated but content unchanged. Unique Python reference explicitly imported by rules/python/; no overlap found."`

### フェーズ3 — サマリーテーブル

| スキル | 7日使用 | 判定 | 理由 |
|-------|--------|---------|--------|

### フェーズ4 — 統合

1. **Retire / Merge**: ユーザーに確認する前にファイルごとの詳細な正当性を提示する:
   - 発見された具体的な問題（重複、陳腐化、壊れたリファレンス等）
   - 同じ機能をカバーする代替（Retireの場合: 既存のスキル/ルール；Mergeの場合: ターゲットファイルと統合するコンテンツ）
   - 削除の影響（依存するスキル、MEMORY.mdの参照、影響を受けるワークフロー）
2. **Improve**: 根拠付きの具体的な改善提案を提示する:
   - 何を変更するか、なぜか（例: 「セクションX/Yがpython-patternsと重複しているため430→200行に削減する」）
   - ユーザーが行動するかどうかを決める
3. **Update**: ソースを確認した上で更新されたコンテンツを提示する
4. MEMORY.mdの行数を確認；100行を超える場合は圧縮を提案する

## 結果ファイルのスキーマ

`~/.claude/skills/skill-stocktake/results.json`:

**`evaluated_at`**: 評価完了の実際のUTC時刻に設定すること。
Bashで取得: `date -u +%Y-%m-%dT%H:%M:%SZ`。`T00:00:00Z` のような日付のみの近似を使用しないこと。

```json
{
  "evaluated_at": "2026-02-21T10:00:00Z",
  "mode": "full",
  "batch_progress": {
    "total": 80,
    "evaluated": 80,
    "status": "completed"
  },
  "skills": {
    "skill-name": {
      "path": "~/.claude/skills/skill-name/SKILL.md",
      "verdict": "Keep",
      "reason": "Concrete, actionable, unique value for X workflow",
      "mtime": "2026-01-15T08:30:00Z"
    }
  }
}
```

## 注意事項

- 評価はブラインドです: 同じチェックリストがすべてのスキルに適用されます（出所がECC、自作、自動抽出かに関係なく）
- アーカイブ/削除操作は常にユーザーの明示的な確認が必要です
- スキルの出所によって判定を分岐させません
