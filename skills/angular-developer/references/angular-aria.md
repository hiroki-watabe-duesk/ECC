# Angular Aria

Angular Aria（`@angular/aria`）は、一般的な WAI-ARIA パターンを実装したヘッドレスでアクセシブルなディレクティブのコレクションです。これらのディレクティブは、キーボード操作・ARIA 属性・フォーカス管理・スクリーンリーダーサポートを担います。

**AI エージェントとしての役割は、HTML 構造と CSS スタイリングを提供すること**であり、複雑なアクセシビリティロジックはディレクティブが処理します。

## ヘッドレスコンポーネントのスタイリング

Angular Aria コンポーネントはヘッドレスのため、デフォルトスタイルは付属しません。ディレクティブが自動的に適用する ARIA 属性や構造的クラスをもとに、CSS で各状態をスタイリングする**必要があります**。

CSS でターゲットにする一般的な ARIA 属性:

- `[aria-expanded="true"]` / `[aria-expanded="false"]`
- `[aria-selected="true"]`
- `[aria-disabled="true"]`
- `[aria-current="page"]`（ナビゲーション用）

---

**重要**: このパッケージを使用する前に、パッケージマネージャー経由でインストールする必要があります。プロジェクトにインストール済みであることを確認してください。必要な場合は `npm install @angular/aria` でインストールしてください。

## 1. アコーディオン

関連するコンテンツを展開・折りたたみ可能なセクションにまとめます。

**使用場面:** アコーディオンは、コンテンツ量の多いページでスクロールを減らすために、コンテンツを論理的なグループに整理して一度に 1 つずつ展開するレイアウトコンポーネントです。FAQ・長いフォーム・段階的な情報開示に適していますが、プライマリナビゲーションや複数のセクションを同時に確認する必要があるシナリオには向きません。

**インポート:** `import { AccordionContent, AccordionGroup, AccordionPanel, AccordionTrigger } from '@angular/aria/accordion';`

**ディレクティブ:** `ngAccordionGroup`, `ngAccordionTrigger`, `ngAccordionPanel`, `ngAccordionContent`（遅延ロード用）。

```ts
@Component({
  selector: 'app-cmp',
  imports: [AccordionContent, AccordionGroup, AccordionPanel, AccordionTrigger],
  template: `...`,
  styles: [],
})
export class App {
  protected readonly title = signal('angular-app');
}
```

```html
<div ngAccordionGroup [multiExpandable]="false">
  <div class="accordion-item">
    <button ngAccordionTrigger panelId="panel-1" class="accordion-header">
      Section 1
      <span class="icon">▼</span>
    </button>
    <div ngAccordionPanel panelId="panel-1" class="accordion-panel">
      <ng-template ngAccordionContent>
        <p>Lazy loaded content here.</p>
      </ng-template>
    </div>
  </div>
</div>
```

**スタイリング方針:**
トリガーの `[aria-expanded]` 属性をターゲットにしてアイコンを回転させ、パネルの表示をスタイリングします。

```css
.accordion-header[aria-expanded='true'] .icon {
  transform: rotate(180deg);
}

/* The panel directive handles DOM removal, but you can style the transition */
.accordion-panel {
  padding: 1rem;
  border-top: 1px solid #ccc;
}
```

---

## 2. リストボックス

オプションのリストを表示するための基礎ディレクティブです。表示形式の選択リスト（ドロップダウンではない）に使用します。

**使用場面:** 表示形式の選択可能なリスト（シングルまたはマルチ選択）。

**インポート:** `import {Listbox, Option} from '@angular/aria/listbox';`

**ディレクティブ:** `ngListbox`, `ngOption`。

```ts
@Component({
  selector: 'app-cmp',
  imports: [Listbox, Option],
  template: `...`,
  styles: [],
})
export class App {
  protected readonly title = signal('angular-app');
}
```

```html
<!-- horizontal or vertical orientation -->
<ul ngListbox [(values)]="selectedItems" orientation="horizontal" [multi]="true">
  <li ngOption value="apple" class="option">Apple</li>
  <li ngOption value="banana" class="option">Banana</li>
</ul>
```

**スタイリング方針:**
選択状態には `[aria-selected="true"]`、フォーカス中のアイテムには `:focus-visible` または `[data-active]` をターゲットにします（Angular Aria はローリング tabindex または activedescendant を使用します）。

```css
.option {
  padding: 8px;
  cursor: pointer;
}
.option[aria-selected='true'] {
  background: #e0f7fa;
  font-weight: bold;
}
/* Focus state managed by aria */
.option:focus-visible {
  outline: 2px solid blue;
}
```

---

## 3. コンボボックス・セレクト・マルチセレクト

これらのパターンは `ngCombobox` とポップアップ内の `ngListbox` を組み合わせます。

- **コンボボックス**: テキスト入力 + ポップアップ（オートコンプリートに使用）。
- **セレクト**: 読み取り専用コンボボックス + シングルセレクトリストボックス。
- **マルチセレクト**: 読み取り専用コンボボックス + マルチセレクトリストボックス。

**使用場面:** コンボボックスは、テキスト入力とポップアップを同期する低レベルのプリミティブディレクティブで、オートコンプリート・セレクト・マルチセレクトパターンの基礎ロジックとなります。カスタムフィルタリング・独自の選択要件・標準文書化コンポーネントから逸脱した入力とポップアップの連携を構築する場合に使用します。

**インポート:**

```
  import {Combobox, ComboboxInput, ComboboxPopupContainer} from '@angular/aria/combobox';
  import {Listbox, Option} from '@angular/aria/listbox';
```

**ディレクティブ:** `ngCombobox`, `ngComboboxInput`, `ngComboboxPopupContainer`, `ngListbox`, `ngOption`。

```html
<!-- Example: Standard Select -->
<div ngCombobox [readonly]="true">
  <button ngComboboxInput class="select-trigger">
    {{ selectedValue() || 'Choose an option' }}
  </button>

  <ng-template ngComboboxPopupContainer>
    <ul ngListbox [(values)]="selectedValue" class="dropdown-menu">
      <li ngOption value="option1">Option 1</li>
      <li ngOption value="option2">Option 2</li>
    </ul>
  </ng-template>
</div>
```

**スタイリング方針:**
ポップアップコンテナをコンテンツの上に浮かぶドロップダウンのように見せるスタイルを設定します（CDK Overlay と組み合わせることが多い）。

```css
.select-trigger {
  width: 200px;
  padding: 8px;
  text-align: left;
}
.dropdown-menu {
  list-style: none;
  padding: 0;
  margin: 0;
  border: 1px solid #ccc;
  background: white;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
}
```

---

## 4. メニューとメニューバー

アクション・コマンド・コンテキストメニュー用（フォーム選択には使用しません）。

**使用場面:** メニューバーは、デスクトップスタイルのアプリケーションコマンドバー（例：ファイル・編集・表示）を構築するための高レベルナビゲーションパターンで、インターフェース全体にわたって持続的に表示されます。複雑なコマンドを論理的なトップレベルカテゴリにまとめ、水平方向のフルキーボードサポートを必要とする場合に最適ですが、シンプルなスタンドアロンアクションリストや水平スペースが制限されるモバイルファーストのレイアウトには適しません。

**インポート:** `import {MenuBar, Menu, MenuContent, MenuItem} from '@angular/aria/menu';`

**ディレクティブ:** `ngMenuBar`, `ngMenu`, `ngMenuItem`, `ngMenuTrigger`。

```html
<!-- Menubar Example -->
<ul ngMenuBar class="menubar">
  <li ngMenuItem value="file">
    <button ngMenuTrigger [menu]="fileMenu">File</button>
  </li>
</ul>

<ul ngMenu #fileMenu="ngMenu" class="menu">
  <li ngMenuItem value="new">New</li>
  <li ngMenuItem value="open">Open</li>
</ul>
```

**スタイリング方針:**
メニューバーには flexbox を使用します。サブメニューの表示・非表示はトリガーの状態に基づいて切り替えます。

```css
.menubar {
  display: flex;
  gap: 10px;
  list-style: none;
  padding: 0;
}
.menu {
  background: white;
  border: 1px solid #ccc;
  padding: 5px 0;
}
.menu li {
  padding: 5px 15px;
  cursor: pointer;
}
```

---

## 5. タブ

1 つのパネルのみ表示される、レイヤード型のコンテンツセクションです。

**使用場面:** タブコンポーネントは、関連するコンテンツを独立したナビゲート可能なセクションに整理し、ページを離れることなくカテゴリやビューを切り替えられるようにします。設定パネル・複数トピックのドキュメント・ダッシュボードに最適ですが、逐次ワークフロー（ステッパー）や 7〜8 セクションを超えるナビゲーションには避けてください。

**インポート:** `import {Tab, Tabs, TabList, TabPanel, TabContent} from '@angular/aria/tabs';`

**ディレクティブ:** `ngTabs`, `ngTabList`, `ngTab`, `ngTabPanel`, `ngTabContent`。

```html
<div ngTabs>
  <ul ngTabList class="tab-list">
    <li ngTab value="profile" class="tab-btn">Profile</li>
    <li ngTab value="security" class="tab-btn">Security</li>
  </ul>

  <div ngTabPanel value="profile" class="tab-panel">
    <ng-template ngTabContent>Profile Settings</ng-template>
  </div>
  <div ngTabPanel value="security" class="tab-panel">
    <ng-template ngTabContent>Security Settings</ng-template>
  </div>
</div>
```

**スタイリング方針:**
タブボタンの `[aria-selected="true"]` をターゲットにします。

```css
.tab-list {
  display: flex;
  border-bottom: 2px solid #ccc;
  list-style: none;
  padding: 0;
}
.tab-btn {
  padding: 10px 20px;
  cursor: pointer;
  border-bottom: 2px solid transparent;
}
.tab-btn[aria-selected='true'] {
  border-bottom-color: blue;
  font-weight: bold;
}
.tab-panel {
  padding: 20px;
}
```

---

## 6. ツールバー

関連するコントロール（テキスト書式設定など）をグループ化します。

**使用場面:** ツールバーは、頻繁にアクセスする関連コントロールを単一の論理的なコンテナにグループ化するためのコンポーネントです。テキスト書式設定やメディアコントロールなど、繰り返し操作が必要なワークフローにおけるキーボード効率（矢印キーナビゲーション）と視覚的な構造を強化するために最適です。

**インポート:** `import {Toolbar, ToolbarWidget, ToolbarWidgetGroup} from '@angular/aria/toolbar';`

**ディレクティブ:** `ngToolbar`, `ngToolbarWidget`, `ngToolbarWidgetGroup`。

```html
<div ngToolbar class="toolbar">
  <div ngToolbarWidgetGroup [multi]="true" role="group" aria-label="Formatting">
    <button ngToolbarWidget value="bold" class="tool-btn">B</button>
    <button ngToolbarWidget value="italic" class="tool-btn">I</button>
  </div>
</div>
```

**スタイリング方針:**
ツールバー内のトグルボタンには `[aria-pressed="true"]`、ラジオグループには `[aria-checked="true"]` をターゲットにします。

```css
.toolbar {
  display: flex;
  gap: 5px;
  padding: 8px;
  background: #f5f5f5;
}
.tool-btn {
  padding: 5px 10px;
  border: 1px solid #ccc;
}
.tool-btn[aria-pressed='true'],
.tool-btn[aria-checked='true'] {
  background: #ddd;
}
```

---

## 7. ツリー

階層データ（ファイルシステム・ネストされたナビゲーションなど）を表示します。

**使用場面:** ツリーコンポーネントは、ファイルシステム・組織図・複雑なサイトアーキテクチャなど、深くネストされた階層データ構造をナビゲート・表示するために設計されています。ユーザーがブランチを展開・折りたたむ必要がある複数レベルの関係を持つ場合に使用しますが、フラットリスト・データテーブル・シンプルな選択メニューには適しません。

**インポート:** `import {Tree, TreeItem, TreeItemGroup} from '@angular/aria/tree';`

**ディレクティブ:** `ngTree`, `ngTreeItem`, `ngTreeGroup`。

```html
<ul ngTree class="tree">
  <li ngTreeItem value="documents">
    <span class="tree-label">Documents</span>
    <ul ngTreeGroup class="tree-group">
      <li ngTreeItem value="resume">Resume.pdf</li>
    </ul>
  </li>
</ul>
```

**スタイリング方針:**
`[aria-expanded]` をターゲットにして子要素の表示・非表示やシェブロンアイコンの回転を実装します。階層を示すためにネストされたグループに `padding-left` を使用します。

```css
.tree,
.tree-group {
  list-style: none;
  padding-left: 20px;
}
.tree-label::before {
  content: '> ';
  display: inline-block;
  transition: transform 0.2s;
}
li[aria-expanded='true'] > .tree-label::before {
  transform: rotate(90deg);
}
```

## 8. グリッド

矢印キーによるナビゲーションを可能にする、セルの双方向インタラクティブコレクションです。

**使用場面:** データテーブル・カレンダー・スプレッドシート・インタラクティブ要素のレイアウトパターン。
**ディレクティブ:** `ngGrid`, `ngGridRow`, `ngGridCell`, `ngGridCellWidget`。

```html
<table ngGrid [multi]="true" [enableSelection]="true" class="grid-table">
  <tr ngGridRow>
    <th ngGridCell role="columnheader">Name</th>
    <th ngGridCell role="columnheader">Status</th>
  </tr>
  <tr ngGridRow>
    <td ngGridCell>Project A</td>
    <td ngGridCell [(selected)]="isSelected">
      <button ngGridCellWidget (activated)="onActivate()">Active</button>
    </td>
  </tr>
</table>
```

**スタイリング方針:**
選択されたセルには `[aria-selected="true"]`、アクティブセルには `:focus-visible`（ローリング tabindex）またはコンテナの `[aria-activedescendant]` をターゲットにします。

```css
.grid-table {
  border-collapse: collapse;
}
[ngGridCell] {
  padding: 8px;
  border: 1px solid #ddd;
}
[ngGridCell][aria-selected='true'] {
  background: #e3f2fd;
}
/* Focus state managed by roving tabindex */
[ngGridCell]:focus-visible {
  outline: 2px solid #2196f3;
  outline-offset: -2px;
}
```

## エージェントの一般的なルール

1. **これらの特定の Aria パターンを実装する際は、`<select>` などのネイティブ HTML 要素を使用しない**: `ng*` ディレクティブを使用してください。
2. **CSS は手動で処理する**: `Angular Aria` はスタイルを提供しません。ディレクティブが自動的に切り替えるネイティブ ARIA 属性（`aria-expanded`・`aria-selected` など）をターゲットにした CSS を自分で記述する必要があります。
3. **遅延ロード**: 重いコンテンツパネルには、提供された構造的ディレクティブ（`ngAccordionContent`・`ngTabContent`）を `ng-template` の内側で常に使用し、遅延レンダリングを確保してください。
