---
name: ui-demo
description: Playwrightを使用して洗練されたUIデモ動画を録画します。ユーザーがWebアプリケーションのデモ、ウォークスルー、画面録画、チュートリアル動画の作成を依頼したときに使用します。可視カーソル、自然なペーシング、プロフェッショナルな仕上がりのWebM動画を生成します。
origin: ECC
---

# UIデモ動画レコーダー

Playwrightの動画録画機能と、注入されたカーソルオーバーレイ、自然なペーシング、ストーリーテリングフローを使用して、Webアプリケーションの洗練されたデモ動画を録画します。

## 使用するタイミング

- ユーザーが「デモ動画」「画面録画」「ウォークスルー」「チュートリアル」を求めているとき
- ユーザーが機能やワークフローを視覚的に紹介したいとき
- ユーザーがドキュメント、オンボーディング、ステークホルダープレゼンテーション用の動画を必要としているとき

## 3フェーズプロセス

すべてのデモは**発見 -> リハーサル -> 録画**の3フェーズを経ます。録画に直接進んではいけません。

---

## フェーズ1: 発見

スクリプトを書く前に、対象ページを探索して実際に何があるかを理解します。

### なぜ必要か

見ていないものをスクリプト化することはできません。フィールドが `<textarea>` ではなく `<input>` であったり、ドロップダウンが `<select>` ではなくカスタムコンポーネントであったり、コメントボックスが `@メンション` や `#タグ` をサポートしていたりする場合があります。思い込みは録画を無音で壊します。

### 方法

フロー内の各ページに移動し、インタラクティブ要素をダンプします:

```javascript
// Run this for each page in the flow BEFORE writing the demo script
const fields = await page.evaluate(() => {
  const els = [];
  document.querySelectorAll('input, select, textarea, button, [contenteditable]').forEach(el => {
    if (el.offsetParent !== null) {
      els.push({
        tag: el.tagName,
        type: el.type || '',
        name: el.name || '',
        placeholder: el.placeholder || '',
        text: el.textContent?.trim().substring(0, 40) || '',
        contentEditable: el.contentEditable === 'true',
        role: el.getAttribute('role') || '',
      });
    }
  });
  return els;
});
console.log(JSON.stringify(fields, null, 2));
```

### 確認すべき点

- **フォームフィールド**: `<select>`、`<input>`、カスタムドロップダウン、コンボボックスのどれか？
- **選択オプション**: オプションの値とテキストの両方をダンプします。プレースホルダーは `value="0"` や `value=""` を持つことがあり、空でないように見えます。`Array.from(el.options).map(o => ({ value: o.value, text: o.text }))` を使用します。テキストに「Select」が含まれるか値が `"0"` のオプションはスキップします。
- **リッチテキスト**: コメントボックスは `@メンション`、`#タグ`、マークダウン、絵文字をサポートしているか？プレースホルダーテキストを確認します。
- **必須フィールド**: どのフィールドがフォーム送信をブロックするか？ `required`、ラベルの `*`、空で送信してバリデーションエラーを確認します。
- **動的コンテンツ**: 他のフィールドが入力された後にフィールドが表示されるか？
- **ボタンラベル**: `"Submit"`、`"Submit Request"`、`"Send"` などの正確なテキスト。
- **テーブル列ヘッダー**: テーブル駆動のモーダルでは、すべての数値入力が同じ意味だと仮定するのではなく、各 `input[type="number"]` をその列ヘッダーにマッピングします。

### 出力

スクリプトで正しいセレクターを書くために使用する、各ページのフィールドマップ。例:

```text
/purchase-requests/new:
  - Budget Code: <select> (first select on page, 4 options)
  - Desired Delivery: <input type="date">
  - Context: <textarea> (not input)
  - BOM table: inline-editable cells with span.cursor-pointer -> input pattern
  - Submit: <button> text="Submit"

/purchase-requests/N (detail):
  - Comment: <input placeholder="Type a message..."> supports @user and #PR tags
  - Send: <button> text="Send" (disabled until input has content)
```

---

## フェーズ2: リハーサル

録画せずにすべてのステップを通して実行します。すべてのセレクターが解決されることを確認します。

### なぜ必要か

サイレントなセレクター失敗がデモ録画が壊れる主な理由です。リハーサルは録画を無駄にする前にそれらを捕捉します。

### 方法

大きな声でログを出力して失敗する `ensureVisible` ラッパーを使用します:

```javascript
async function ensureVisible(page, locator, label) {
  const el = typeof locator === 'string' ? page.locator(locator).first() : locator;
  const visible = await el.isVisible().catch(() => false);
  if (!visible) {
    const msg = `REHEARSAL FAIL: "${label}" not found - selector: ${typeof locator === 'string' ? locator : '(locator object)'}`;
    console.error(msg);
    const found = await page.evaluate(() => {
      return Array.from(document.querySelectorAll('button, input, select, textarea, a'))
        .filter(el => el.offsetParent !== null)
        .map(el => `${el.tagName}[${el.type || ''}] "${el.textContent?.trim().substring(0, 30)}"`)
        .join('\n  ');
    });
    console.error('  Visible elements:\n  ' + found);
    return false;
  }
  console.log(`REHEARSAL OK: "${label}"`);
  return true;
}
```

### リハーサルスクリプトの構造

```javascript
const steps = [
  { label: 'Login email field', selector: '#email' },
  { label: 'Login submit', selector: 'button[type="submit"]' },
  { label: 'New Request button', selector: 'button:has-text("New Request")' },
  { label: 'Budget Code select', selector: 'select' },
  { label: 'Delivery date', selector: 'input[type="date"]:visible' },
  { label: 'Description field', selector: 'textarea:visible' },
  { label: 'Add Item button', selector: 'button:has-text("Add Item")' },
  { label: 'Submit button', selector: 'button:has-text("Submit")' },
];

let allOk = true;
for (const step of steps) {
  if (!await ensureVisible(page, step.selector, step.label)) {
    allOk = false;
  }
}
if (!allOk) {
  console.error('REHEARSAL FAILED - fix selectors before recording');
  process.exit(1);
}
console.log('REHEARSAL PASSED - all selectors verified');
```

### リハーサルが失敗した場合

1. 可視要素のダンプを読む。
2. 正しいセレクターを見つける。
3. スクリプトを更新する。
4. リハーサルを再実行する。
5. すべてのセレクターがパスした場合のみ先に進む。

---

## フェーズ3: 録画

発見とリハーサルがパスした後にのみ録画を作成します。

### 録画の原則

#### 1. ストーリーテリングフロー

動画をストーリーとして計画します。ユーザー指定の順序に従うか、次のデフォルトを使用します:

- **エントリー**: ログインするか開始点に移動する
- **コンテキスト**: 視聴者が方向を把握できるよう周囲をパンする
- **アクション**: メインワークフローのステップを実行する
- **バリエーション**: 設定、テーマ、ローカライゼーションなどの二次機能を示す
- **結果**: 結果、確認、または新しい状態を表示する

#### 2. ペーシング

- ログイン後: `4秒`
- ナビゲーション後: `3秒`
- ボタンクリック後: `2秒`
- 主要ステップ間: `1.5〜2秒`
- 最終アクション後: `3秒`
- タイピング遅延: 1文字あたり `25〜40ミリ秒`

#### 3. カーソルオーバーレイ

マウスの動きに追従するSVGの矢印カーソルを注入します:

```javascript
async function injectCursor(page) {
  await page.evaluate(() => {
    if (document.getElementById('demo-cursor')) return;
    const cursor = document.createElement('div');
    cursor.id = 'demo-cursor';
    cursor.innerHTML = `<svg width="24" height="24" viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg">
      <path d="M5 3L19 12L12 13L9 20L5 3Z" fill="white" stroke="black" stroke-width="1.5" stroke-linejoin="round"/>
    </svg>`;
    cursor.style.cssText = `
      position: fixed; z-index: 999999; pointer-events: none;
      width: 24px; height: 24px;
      transition: left 0.1s, top 0.1s;
      filter: drop-shadow(1px 1px 2px rgba(0,0,0,0.3));
    `;
    cursor.style.left = '0px';
    cursor.style.top = '0px';
    document.body.appendChild(cursor);
    document.addEventListener('mousemove', (e) => {
      cursor.style.left = e.clientX + 'px';
      cursor.style.top = e.clientY + 'px';
    });
  });
}
```

ナビゲーションごとにオーバーレイが破棄されるため、ページナビゲーション後に `injectCursor(page)` を呼び出してください。

#### 4. マウスの動き

カーソルをテレポートさせないでください。クリックする前にターゲットに移動します:

```javascript
async function moveAndClick(page, locator, label, opts = {}) {
  const { postClickDelay = 800, ...clickOpts } = opts;
  const el = typeof locator === 'string' ? page.locator(locator).first() : locator;
  const visible = await el.isVisible().catch(() => false);
  if (!visible) {
    console.error(`WARNING: moveAndClick skipped - "${label}" not visible`);
    return false;
  }
  try {
    await el.scrollIntoViewIfNeeded();
    await page.waitForTimeout(300);
    const box = await el.boundingBox();
    if (box) {
      await page.mouse.move(box.x + box.width / 2, box.y + box.height / 2, { steps: 10 });
      await page.waitForTimeout(400);
    }
    await el.click(clickOpts);
  } catch (e) {
    console.error(`WARNING: moveAndClick failed on "${label}": ${e.message}`);
    return false;
  }
  await page.waitForTimeout(postClickDelay);
  return true;
}
```

すべての呼び出しにデバッグ用の説明的な `label` を含めてください。

#### 5. タイピング

即時入力ではなく、視覚的に入力します:

```javascript
async function typeSlowly(page, locator, text, label, charDelay = 35) {
  const el = typeof locator === 'string' ? page.locator(locator).first() : locator;
  const visible = await el.isVisible().catch(() => false);
  if (!visible) {
    console.error(`WARNING: typeSlowly skipped - "${label}" not visible`);
    return false;
  }
  await moveAndClick(page, el, label);
  await el.fill('');
  await el.pressSequentially(text, { delay: charDelay });
  await page.waitForTimeout(500);
  return true;
}
```

#### 6. スクロール

ジャンプではなくスムーズスクロールを使用します:

```javascript
await page.evaluate(() => window.scrollTo({ top: 400, behavior: 'smooth' }));
await page.waitForTimeout(1500);
```

#### 7. ダッシュボードパンニング

ダッシュボードや概要ページを表示するとき、主要な要素の上でカーソルを動かします:

```javascript
async function panElements(page, selector, maxCount = 6) {
  const elements = await page.locator(selector).all();
  for (let i = 0; i < Math.min(elements.length, maxCount); i++) {
    try {
      const box = await elements[i].boundingBox();
      if (box && box.y < 700) {
        await page.mouse.move(box.x + box.width / 2, box.y + box.height / 2, { steps: 8 });
        await page.waitForTimeout(600);
      }
    } catch (e) {
      console.warn(`WARNING: panElements skipped element ${i} (selector: "${selector}"): ${e.message}`);
    }
  }
}
```

#### 8. 字幕

ビューポートの下部に字幕バーを注入します:

```javascript
async function injectSubtitleBar(page) {
  await page.evaluate(() => {
    if (document.getElementById('demo-subtitle')) return;
    const bar = document.createElement('div');
    bar.id = 'demo-subtitle';
    bar.style.cssText = `
      position: fixed; bottom: 0; left: 0; right: 0; z-index: 999998;
      text-align: center; padding: 12px 24px;
      background: rgba(0, 0, 0, 0.75);
      color: white; font-family: -apple-system, "Segoe UI", sans-serif;
      font-size: 16px; font-weight: 500; letter-spacing: 0.3px;
      transition: opacity 0.3s;
      pointer-events: none;
    `;
    bar.textContent = '';
    bar.style.opacity = '0';
    document.body.appendChild(bar);
  });
}

async function showSubtitle(page, text) {
  await page.evaluate((t) => {
    const bar = document.getElementById('demo-subtitle');
    if (!bar) return;
    if (t) {
      bar.textContent = t;
      bar.style.opacity = '1';
    } else {
      bar.style.opacity = '0';
    }
  }, text);
  if (text) await page.waitForTimeout(800);
}
```

ナビゲーションごとに `injectCursor(page)` と一緒に `injectSubtitleBar(page)` を呼び出してください。

使用パターン:

```javascript
await showSubtitle(page, 'Step 1 - Logging in');
await showSubtitle(page, 'Step 2 - Dashboard overview');
await showSubtitle(page, '');
```

ガイドライン:

- 字幕テキストは短く、理想的には60文字以内にします。
- 一貫性のために `Step N - Action` 形式を使用します。
- UIが自ら語る長い間には字幕を消去します。

## スクリプトテンプレート

```javascript
'use strict';
const { chromium } = require('playwright');
const path = require('path');
const fs = require('fs');

const BASE_URL = process.env.QA_BASE_URL || 'http://localhost:3000';
const VIDEO_DIR = path.join(__dirname, 'screenshots');
const OUTPUT_NAME = 'demo-FEATURE.webm';
const REHEARSAL = process.argv.includes('--rehearse');

// Paste injectCursor, injectSubtitleBar, showSubtitle, moveAndClick,
// typeSlowly, ensureVisible, and panElements here.

(async () => {
  const browser = await chromium.launch({ headless: true });

  if (REHEARSAL) {
    const context = await browser.newContext({ viewport: { width: 1280, height: 720 } });
    const page = await context.newPage();
    // Navigate through the flow and run ensureVisible for each selector.
    await browser.close();
    return;
  }

  const context = await browser.newContext({
    recordVideo: { dir: VIDEO_DIR, size: { width: 1280, height: 720 } },
    viewport: { width: 1280, height: 720 }
  });
  const page = await context.newPage();

  try {
    await injectCursor(page);
    await injectSubtitleBar(page);

    await showSubtitle(page, 'Step 1 - Logging in');
    // login actions

    await page.goto(`${BASE_URL}/dashboard`);
    await injectCursor(page);
    await injectSubtitleBar(page);
    await showSubtitle(page, 'Step 2 - Dashboard overview');
    // pan dashboard

    await showSubtitle(page, 'Step 3 - Main workflow');
    // action sequence

    await showSubtitle(page, 'Step 4 - Result');
    // final reveal
    await showSubtitle(page, '');
  } catch (err) {
    console.error('DEMO ERROR:', err.message);
  } finally {
    await context.close();
    const video = page.video();
    if (video) {
      const src = await video.path();
      const dest = path.join(VIDEO_DIR, OUTPUT_NAME);
      try {
        fs.copyFileSync(src, dest);
        console.log('Video saved:', dest);
      } catch (e) {
        console.error('ERROR: Failed to copy video:', e.message);
        console.error('  Source:', src);
        console.error('  Destination:', dest);
      }
    }
    await browser.close();
  }
})();
```

使用方法:

```bash
# Phase 2: Rehearse
node demo-script.cjs --rehearse

# Phase 3: Record
node demo-script.cjs
```

## 録画前チェックリスト

- [ ] 発見フェーズが完了している
- [ ] すべてのセレクターがOKでリハーサルがパスしている
- [ ] ヘッドレスモードが有効になっている
- [ ] 解像度が `1280x720` に設定されている
- [ ] ナビゲーションごとにカーソルと字幕オーバーレイが再注入されている
- [ ] 主要な遷移で `showSubtitle(page, 'Step N - ...')` を使用している
- [ ] 説明的なラベル付きですべてのクリックに `moveAndClick` を使用している
- [ ] 可視入力に `typeSlowly` を使用している
- [ ] サイレントなcatchなし。ヘルパーは警告をログに出力する
- [ ] コンテンツ表示にスムーズスクロールを使用している
- [ ] 重要な間隔が人間の視聴者に見える
- [ ] フローがリクエストされたストーリーの順序に一致している
- [ ] スクリプトがフェーズ1で発見した実際のUIを反映している

## よくある落とし穴

1. ナビゲーション後にカーソルが消える - 再注入する。
2. 動画が速すぎる - 間隔を追加する。
3. カーソルが矢印ではなく点になっている - SVGオーバーレイを使用する。
4. カーソルがテレポートする - クリックする前に移動する。
5. セレクトドロップダウンが正しく見えない - 移動を表示してからオプションを選択する。
6. モーダルが唐突に感じる - 確認する前に読む間隔を追加する。
7. 動画ファイルパスがランダムになっている - 安定した出力名にコピーする。
8. セレクターの失敗が飲み込まれている - サイレントcatchブロックを使わない。
9. フィールドタイプが思い込まれた - まず発見する。
10. 機能が思い込まれた - スクリプト化する前に実際のUIを調査する。
11. プレースホルダーの選択値が本物に見える - `"0"` と `"Select..."` に注意する。
12. ポップアップが別々の動画を作成する - ポップアップページを明示的にキャプチャし、必要に応じて後でマージする。
