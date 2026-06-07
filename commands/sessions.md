---
description: Claude Codeのセッション履歴、エイリアス、セッションメタデータを管理する。
---

# Sessions コマンド

Claude Code のセッション履歴を管理します。`~/.claude/session-data/` に保存されたセッションの一覧表示、ロード、エイリアス設定、編集、および `~/.claude/sessions/` からのレガシー読み取りに対応します。

## 使い方

`/sessions [list|load|alias|info|help] [options]`

## アクション

### セッションの一覧表示

メタデータ、フィルタリング、ページネーションを含むすべてのセッションを表示します。

スウォーム向けのオペレーターサーフェスコンテキスト（ブランチ、ワークツリーパス、セッションの新しさ）が必要な場合は `/sessions info` を使用してください。

```bash
/sessions                              # すべてのセッションを一覧表示（デフォルト）
/sessions list                         # 上と同じ
/sessions list --limit 10              # 10件のセッションを表示
/sessions list --date 2026-02-01       # 日付でフィルタリング
/sessions list --search abc            # セッションIDで検索
```

**スクリプト:**
```bash
node -e "
const _r = (()=>{var e=process.env.CLAUDE_PLUGIN_ROOT;if(e&&e.trim())return e.trim();var p=require('path'),f=require('fs'),h=require('os').homedir(),d=p.join(h,'.claude'),q=p.join('scripts','lib','utils.js');if(f.existsSync(p.join(d,q)))return d;for(var s of [['ecc'],['ecc@ecc'],['marketplaces','ecc'],['everything-claude-code'],['everything-claude-code@everything-claude-code'],['marketplaces','everything-claude-code']]){var l=p.join(d,'plugins',...s);if(f.existsSync(p.join(l,q)))return l}try{for(var g of ['ecc','everything-claude-code']){var b=p.join(d,'plugins','cache',g);for(var o of f.readdirSync(b,{withFileTypes:true})){if(!o.isDirectory())continue;for(var v of f.readdirSync(p.join(b,o.name),{withFileTypes:true})){if(!v.isDirectory())continue;var c=p.join(b,o.name,v.name);if(f.existsSync(p.join(c,q)))return c}}}}catch(x){}return d})();
const sm = require(_r + '/scripts/lib/session-manager');
const aa = require(_r + '/scripts/lib/session-aliases');
const path = require('path');

const result = sm.getAllSessions({ limit: 20 });
const aliases = aa.listAliases();
const aliasMap = {};
for (const a of aliases) aliasMap[a.sessionPath] = a.name;

console.log('Sessions (showing ' + result.sessions.length + ' of ' + result.total + '):');
console.log('');
console.log('ID        Date        Time     Branch       Worktree           Alias');
console.log('────────────────────────────────────────────────────────────────────');

for (const s of result.sessions) {
  const alias = aliasMap[s.filename] || '';
  const metadata = sm.parseSessionMetadata(sm.getSessionContent(s.sessionPath));
  const id = s.shortId === 'no-id' ? '(none)' : s.shortId.slice(0, 8);
  const time = s.modifiedTime.toTimeString().slice(0, 5);
  const branch = (metadata.branch || '-').slice(0, 12);
  const worktree = metadata.worktree ? path.basename(metadata.worktree).slice(0, 18) : '-';

  console.log(id.padEnd(8) + ' ' + s.date + '  ' + time + '   ' + branch.padEnd(12) + ' ' + worktree.padEnd(18) + ' ' + alias);
}
"
```

### セッションのロード

セッションの内容をIDまたはエイリアスで表示します。

```bash
/sessions load <id|alias>             # セッションをロード
/sessions load 2026-02-01             # 日付でロード（IDなしセッション向け）
/sessions load a1b2c3d4               # 短縮IDでロード
/sessions load my-alias               # エイリアス名でロード
```

**スクリプト:**
```bash
node -e "
const _r = (()=>{var e=process.env.CLAUDE_PLUGIN_ROOT;if(e&&e.trim())return e.trim();var p=require('path'),f=require('fs'),h=require('os').homedir(),d=p.join(h,'.claude'),q=p.join('scripts','lib','utils.js');if(f.existsSync(p.join(d,q)))return d;for(var s of [['ecc'],['ecc@ecc'],['marketplaces','ecc'],['everything-claude-code'],['everything-claude-code@everything-claude-code'],['marketplaces','everything-claude-code']]){var l=p.join(d,'plugins',...s);if(f.existsSync(p.join(l,q)))return l}try{for(var g of ['ecc','everything-claude-code']){var b=p.join(d,'plugins','cache',g);for(var o of f.readdirSync(b,{withFileTypes:true})){if(!o.isDirectory())continue;for(var v of f.readdirSync(p.join(b,o.name),{withFileTypes:true})){if(!v.isDirectory())continue;var c=p.join(b,o.name,v.name);if(f.existsSync(p.join(c,q)))return c}}}}catch(x){}return d})();
const sm = require(_r + '/scripts/lib/session-manager');
const aa = require(_r + '/scripts/lib/session-aliases');
const id = process.argv[1];

// まずエイリアスとして解決を試みる
const resolved = aa.resolveAlias(id);
const sessionId = resolved ? resolved.sessionPath : id;

const session = sm.getSessionById(sessionId, true);
if (!session) {
  console.log('Session not found: ' + id);
  process.exit(1);
}

const stats = sm.getSessionStats(session.sessionPath);
const size = sm.getSessionSize(session.sessionPath);
const aliases = aa.getAliasesForSession(session.filename);

console.log('Session: ' + session.filename);
console.log('Path: ' + session.sessionPath);
console.log('');
console.log('Statistics:');
console.log('  Lines: ' + stats.lineCount);
console.log('  Total items: ' + stats.totalItems);
console.log('  Completed: ' + stats.completedItems);
console.log('  In progress: ' + stats.inProgressItems);
console.log('  Size: ' + size);
console.log('');

if (aliases.length > 0) {
  console.log('Aliases: ' + aliases.map(a => a.name).join(', '));
  console.log('');
}

if (session.metadata.title) {
  console.log('Title: ' + session.metadata.title);
  console.log('');
}

if (session.metadata.started) {
  console.log('Started: ' + session.metadata.started);
}

if (session.metadata.lastUpdated) {
  console.log('Last Updated: ' + session.metadata.lastUpdated);
}

if (session.metadata.project) {
  console.log('Project: ' + session.metadata.project);
}

if (session.metadata.branch) {
  console.log('Branch: ' + session.metadata.branch);
}

if (session.metadata.worktree) {
  console.log('Worktree: ' + session.metadata.worktree);
}
" "$ARGUMENTS"
```

### エイリアスの作成

セッションに覚えやすいエイリアスを作成します。

```bash
/sessions alias <id> <name>           # エイリアスを作成
/sessions alias 2026-02-01 today-work # "today-work" という名前のエイリアスを作成
```

**スクリプト:**
```bash
node -e "
const _r = (()=>{var e=process.env.CLAUDE_PLUGIN_ROOT;if(e&&e.trim())return e.trim();var p=require('path'),f=require('fs'),h=require('os').homedir(),d=p.join(h,'.claude'),q=p.join('scripts','lib','utils.js');if(f.existsSync(p.join(d,q)))return d;for(var s of [['ecc'],['ecc@ecc'],['marketplaces','ecc'],['everything-claude-code'],['everything-claude-code@everything-claude-code'],['marketplaces','everything-claude-code']]){var l=p.join(d,'plugins',...s);if(f.existsSync(p.join(l,q)))return l}try{for(var g of ['ecc','everything-claude-code']){var b=p.join(d,'plugins','cache',g);for(var o of f.readdirSync(b,{withFileTypes:true})){if(!o.isDirectory())continue;for(var v of f.readdirSync(p.join(b,o.name),{withFileTypes:true})){if(!v.isDirectory())continue;var c=p.join(b,o.name,v.name);if(f.existsSync(p.join(c,q)))return c}}}}catch(x){}return d})();
const sm = require(_r + '/scripts/lib/session-manager');
const aa = require(_r + '/scripts/lib/session-aliases');

const sessionId = process.argv[1];
const aliasName = process.argv[2];

if (!sessionId || !aliasName) {
  console.log('Usage: /sessions alias <id> <name>');
  process.exit(1);
}

// セッションファイル名を取得
const session = sm.getSessionById(sessionId);
if (!session) {
  console.log('Session not found: ' + sessionId);
  process.exit(1);
}

const result = aa.setAlias(aliasName, session.filename);
if (result.success) {
  console.log('✓ Alias created: ' + aliasName + ' → ' + session.filename);
} else {
  console.log('✗ Error: ' + result.error);
  process.exit(1);
}
" "$ARGUMENTS"
```

### エイリアスの削除

既存のエイリアスを削除します。

```bash
/sessions alias --remove <name>        # エイリアスを削除
/sessions unalias <name>               # 上と同じ
```

**スクリプト:**
```bash
node -e "
const _r = (()=>{var e=process.env.CLAUDE_PLUGIN_ROOT;if(e&&e.trim())return e.trim();var p=require('path'),f=require('fs'),h=require('os').homedir(),d=p.join(h,'.claude'),q=p.join('scripts','lib','utils.js');if(f.existsSync(p.join(d,q)))return d;for(var s of [['ecc'],['ecc@ecc'],['marketplaces','ecc'],['everything-claude-code'],['everything-claude-code@everything-claude-code'],['marketplaces','everything-claude-code']]){var l=p.join(d,'plugins',...s);if(f.existsSync(p.join(l,q)))return l}try{for(var g of ['ecc','everything-claude-code']){var b=p.join(d,'plugins','cache',g);for(var o of f.readdirSync(b,{withFileTypes:true})){if(!o.isDirectory())continue;for(var v of f.readdirSync(p.join(b,o.name),{withFileTypes:true})){if(!v.isDirectory())continue;var c=p.join(b,o.name,v.name);if(f.existsSync(p.join(c,q)))return c}}}}catch(x){}return d})();
const aa = require(_r + '/scripts/lib/session-aliases');

const aliasName = process.argv[1];
if (!aliasName) {
  console.log('Usage: /sessions alias --remove <name>');
  process.exit(1);
}

const result = aa.deleteAlias(aliasName);
if (result.success) {
  console.log('✓ Alias removed: ' + aliasName);
} else {
  console.log('✗ Error: ' + result.error);
  process.exit(1);
}
" "$ARGUMENTS"
```

### セッション情報

セッションの詳細情報を表示します。

```bash
/sessions info <id|alias>              # セッションの詳細を表示
```

**スクリプト:**
```bash
node -e "
const _r = (()=>{var e=process.env.CLAUDE_PLUGIN_ROOT;if(e&&e.trim())return e.trim();var p=require('path'),f=require('fs'),h=require('os').homedir(),d=p.join(h,'.claude'),q=p.join('scripts','lib','utils.js');if(f.existsSync(p.join(d,q)))return d;for(var s of [['ecc'],['ecc@ecc'],['marketplaces','ecc'],['everything-claude-code'],['everything-claude-code@everything-claude-code'],['marketplaces','everything-claude-code']]){var l=p.join(d,'plugins',...s);if(f.existsSync(p.join(l,q)))return l}try{for(var g of ['ecc','everything-claude-code']){var b=p.join(d,'plugins','cache',g);for(var o of f.readdirSync(b,{withFileTypes:true})){if(!o.isDirectory())continue;for(var v of f.readdirSync(p.join(b,o.name),{withFileTypes:true})){if(!v.isDirectory())continue;var c=p.join(b,o.name,v.name);if(f.existsSync(p.join(c,q)))return c}}}}catch(x){}return d})();
const sm = require(_r + '/scripts/lib/session-manager');
const aa = require(_r + '/scripts/lib/session-aliases');

const id = process.argv[1];
const resolved = aa.resolveAlias(id);
const sessionId = resolved ? resolved.sessionPath : id;

const session = sm.getSessionById(sessionId, true);
if (!session) {
  console.log('Session not found: ' + id);
  process.exit(1);
}

const stats = sm.getSessionStats(session.sessionPath);
const size = sm.getSessionSize(session.sessionPath);
const aliases = aa.getAliasesForSession(session.filename);

console.log('Session Information');
console.log('════════════════════');
console.log('ID:          ' + (session.shortId === 'no-id' ? '(none)' : session.shortId));
console.log('Filename:    ' + session.filename);
console.log('Date:        ' + session.date);
console.log('Modified:    ' + session.modifiedTime.toISOString().slice(0, 19).replace('T', ' '));
console.log('Project:     ' + (session.metadata.project || '-'));
console.log('Branch:      ' + (session.metadata.branch || '-'));
console.log('Worktree:    ' + (session.metadata.worktree || '-'));
console.log('');
console.log('Content:');
console.log('  Lines:         ' + stats.lineCount);
console.log('  Total items:   ' + stats.totalItems);
console.log('  Completed:     ' + stats.completedItems);
console.log('  In progress:   ' + stats.inProgressItems);
console.log('  Size:          ' + size);
if (aliases.length > 0) {
  console.log('Aliases:     ' + aliases.map(a => a.name).join(', '));
}
" "$ARGUMENTS"
```

### エイリアス一覧

すべてのセッションエイリアスを表示します。

```bash
/sessions aliases                      # すべてのエイリアスを一覧表示
```

**スクリプト:**
```bash
node -e "
const _r = (()=>{var e=process.env.CLAUDE_PLUGIN_ROOT;if(e&&e.trim())return e.trim();var p=require('path'),f=require('fs'),h=require('os').homedir(),d=p.join(h,'.claude'),q=p.join('scripts','lib','utils.js');if(f.existsSync(p.join(d,q)))return d;for(var s of [['ecc'],['ecc@ecc'],['marketplaces','ecc'],['everything-claude-code'],['everything-claude-code@everything-claude-code'],['marketplaces','everything-claude-code']]){var l=p.join(d,'plugins',...s);if(f.existsSync(p.join(l,q)))return l}try{for(var g of ['ecc','everything-claude-code']){var b=p.join(d,'plugins','cache',g);for(var o of f.readdirSync(b,{withFileTypes:true})){if(!o.isDirectory())continue;for(var v of f.readdirSync(p.join(b,o.name),{withFileTypes:true})){if(!v.isDirectory())continue;var c=p.join(b,o.name,v.name);if(f.existsSync(p.join(c,q)))return c}}}}catch(x){}return d})();
const aa = require(_r + '/scripts/lib/session-aliases');

const aliases = aa.listAliases();
console.log('Session Aliases (' + aliases.length + '):');
console.log('');

if (aliases.length === 0) {
  console.log('No aliases found.');
} else {
  console.log('Name          Session File                    Title');
  console.log('─────────────────────────────────────────────────────────────');
  for (const a of aliases) {
    const name = a.name.padEnd(12);
    const file = (a.sessionPath.length > 30 ? a.sessionPath.slice(0, 27) + '...' : a.sessionPath).padEnd(30);
    const title = a.title || '';
    console.log(name + ' ' + file + ' ' + title);
  }
}
"
```

## オペレーター向けメモ

- セッションファイルは `Project`、`Branch`、`Worktree` をヘッダーに保持するため、`/sessions info` で並行する tmux/ワークツリー実行を識別できます。
- コマンドセンタースタイルの監視には、`/sessions info`、`git diff --stat`、および `scripts/hooks/cost-tracker.js` が出力するコストメトリクスを組み合わせてください。

## 引数

$ARGUMENTS:
- `list [options]` - セッションを一覧表示
  - `--limit <n>` - 表示するセッション数の上限（デフォルト: 50）
  - `--date <YYYY-MM-DD>` - 日付でフィルタリング
  - `--search <pattern>` - セッションIDで検索
- `load <id|alias>` - セッション内容をロード
- `alias <id> <name>` - セッションのエイリアスを作成
- `alias --remove <name>` - エイリアスを削除
- `unalias <name>` - `--remove` と同じ
- `info <id|alias>` - セッション統計を表示
- `aliases` - すべてのエイリアスを一覧表示
- `help` - このヘルプを表示

## 使用例

```bash
# すべてのセッションを一覧表示
/sessions list

# 今日のセッションにエイリアスを作成
/sessions alias 2026-02-01 today

# エイリアスでセッションをロード
/sessions load today

# セッション情報を表示
/sessions info today

# エイリアスを削除
/sessions alias --remove today

# すべてのエイリアスを一覧表示
/sessions aliases
```

## メモ

- セッションは `~/.claude/session-data/` にMarkdownファイルとして保存され、`~/.claude/sessions/` からのレガシー読み取りにも対応します
- エイリアスは `~/.claude/session-aliases.json` に保存されます
- セッションIDは短縮可能です（最初の4〜8文字で通常は一意）
- 頻繁に参照するセッションにはエイリアスを使用してください
