---
name: frontend-a11y
description: "ReactとNext.jsのアクセシビリティパターン — セマンティックHTML、ARIA属性、フォームラベリング、キーボードナビゲーション、フォーカス管理、スクリーンリーダー対応。インタラクティブなUIコンポーネントやフォームを構築する際に使用する。"
origin: community
---

# フロントエンドアクセシビリティパターン

ReactとNext.js向けの実践的なアクセシビリティパターン。コードレビューで最もよく指摘される問題をカバー：フォームラベルの欠落、誤ったARIAの使用、非セマンティックなインタラクティブ要素、壊れたキーボードナビゲーション。

## 起動すべき状況

- フォームコンポーネント（`<input>`、`<select>`、`<textarea>`）を構築またはレビューする場合
- インタラクティブ要素（モーダル、ドロップダウン、ツールチップ、タブ）を作成する場合
- `onClick`を持つ`<div>`または`<span>`を使用する場合
- 任意の要素に`aria-*`属性を追加する場合
- キーボードナビゲーションまたはフォーカス管理を実装する場合
- コードレビューツール（CodeRabbit、ESLint a11y）からアクセシビリティのフィードバックを受けた場合
- スクリーンリーダーをサポートする必要があるコンポーネントを構築する場合

## フォームアクセシビリティ

`htmlFor`/`id`のペアリングの欠落と切り離されたエラーメッセージが、コードレビューで最も多く指摘される問題。

### ラベルの接続

```tsx
// BAD: ラベルが入力に接続されていない — スクリーンリーダーが関連付けられない
<label>Email</label>
<input type="email" />

// GOOD: htmlForが入力のidと一致する
<label htmlFor="email">Email</label>
<input id="email" type="email" />
```

### 必須フィールド

```tsx
// BAD: 視覚的なアスタリスクはスクリーンリーダーに何も伝えない
<label htmlFor="email">Email *</label>
<input id="email" type="email" />

// GOOD: requiredがネイティブブラウザバリデーションを有効にする；aria-requiredがスクリーンリーダーに伝える
<label htmlFor="email">
  Email <span aria-hidden="true">*</span>
</label>
<input id="email" type="email" required aria-required="true" />
```

### エラーメッセージ

```tsx
// BAD: エラーテキストは視覚的に存在するが入力にリンクされていない
<input id="email" type="email" />
<span className="error">Invalid email address</span>

// GOOD: aria-describedbyが入力をエラーメッセージに接続する
// aria-invalidが無効な状態をスクリーンリーダーに伝える
<input
  id="email"
  type="email"
  aria-describedby="email-error"
  aria-invalid={!!error}
/>
{error && (
  <span id="email-error" role="alert">
    {error}
  </span>
)}
```

### 完全なアクセシブルフォーム

```tsx
interface LoginFormProps {
  onSubmit: (email: string, password: string) => void;
}

export function LoginForm({ onSubmit }: LoginFormProps) {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [errors, setErrors] = useState<{ email?: string; password?: string }>({});

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    const newErrors: typeof errors = {};
    if (!email) newErrors.email = 'Email is required';
    if (!password) newErrors.password = 'Password is required';
    if (Object.keys(newErrors).length) {
      setErrors(newErrors);
      return;
    }
    onSubmit(email, password);
  };

  return (
    <form onSubmit={handleSubmit} noValidate>
      <div>
        <label htmlFor="email">
          Email <span aria-hidden="true">*</span>
        </label>
        <input
          id="email"
          type="email"
          value={email}
          onChange={e => setEmail(e.target.value)}
          aria-required="true"
          aria-describedby={errors.email ? 'email-error' : undefined}
          aria-invalid={!!errors.email}
          autoComplete="email"
        />
        {errors.email && (
          <span id="email-error" role="alert">
            {errors.email}
          </span>
        )}
      </div>

      <div>
        <label htmlFor="password">
          Password <span aria-hidden="true">*</span>
        </label>
        <input
          id="password"
          type="password"
          value={password}
          onChange={e => setPassword(e.target.value)}
          aria-required="true"
          aria-describedby={errors.password ? 'password-error' : undefined}
          aria-invalid={!!errors.password}
          autoComplete="current-password"
        />
        {errors.password && (
          <span id="password-error" role="alert">
            {errors.password}
          </span>
        )}
      </div>

      <button type="submit">Log in</button>
    </form>
  );
}
```

## セマンティックHTML

意図に合った要素を使用する。スクリーンリーダーとキーボードユーザーはネイティブセマンティクスに依存している。

```tsx
// BAD: divにはroleもキーボードサポートもアクセシブルな名前もない
<div onClick={handleClick}>Submit</div>

// GOOD: buttonはフォーカス可能で、Enter/Spaceで起動し、「button」と読み上げられる
<button type="button" onClick={handleClick}>Submit</button>
```

```tsx
// BAD: 非セマンティックなナビゲーション
<div onClick={() => navigate('/home')}>Home</div>

// GOOD: アンカーは右クリック、中クリック、キーボードナビゲーションをサポートする
<a href="/home">Home</a>
```

```tsx
// BAD: 見出し階層のスキップ（h1からh4へ）
<h1>Dashboard</h1>
<h4>Recent Activity</h4>

// GOOD: 連続した見出しレベル
<h1>Dashboard</h1>
<h2>Recent Activity</h2>
```

## ARIA属性

ネイティブHTMLセマンティクスが不十分な場合にのみARIAを使用する。誤ったARIAはARIAなしよりも悪い。

### aria-label vs aria-labelledby

```tsx
// aria-label: インライン文字列ラベル — 可視ラベルテキストが存在しない場合に使用
<button aria-label="Close modal">
  <XIcon />
</button>

// aria-labelledby: 別の要素のテキストを参照 — 可視ラベルが存在する場合に使用
<section aria-labelledby="section-title">
  <h2 id="section-title">Recent Orders</h2>
  {/* コンテンツ */}
</section>
```

### aria-describedby

```tsx
// ラベルを超えた補足説明を提供する
<button
  aria-describedby="delete-warning"
  onClick={handleDelete}
>
  Delete account
</button>
<p id="delete-warning">This action cannot be undone.</p>
```

### 動的コンテンツ用aria-live

```tsx
// ページのリロードなしに更新されるコンテンツを読み上げるためにaria-liveを使用する
// polite: ユーザーが現在のアクションを終了するまで待ってから読み上げる
// assertive: 即座に割り込む — 緊急エラーにのみ使用

export function StatusMessage({ message, isError }: { message: string; isError?: boolean }) {
  return (
    <div role="status" aria-live={isError ? 'assertive' : 'polite'} aria-atomic="true">
      {message}
    </div>
  );
}
```

### aria-expandedとaria-controls

```tsx
export function Accordion({ title, children }: { title: string; children: React.ReactNode }) {
  const [isOpen, setIsOpen] = useState(false);
  const contentId = useId();

  return (
    <div>
      <button aria-expanded={isOpen} aria-controls={contentId} onClick={() => setIsOpen(prev => !prev)}>
        {title}
      </button>
      <div id={contentId} hidden={!isOpen}>
        {children}
      </div>
    </div>
  );
}
```

## キーボードナビゲーション

すべてのインタラクティブ要素はキーボードだけで到達可能かつ操作可能でなければならない。

### カスタムドロップダウン

```tsx
export function Dropdown({ options, onSelect }: { options: string[]; onSelect: (value: string) => void }) {
  const [isOpen, setIsOpen] = useState(false);
  const [activeIndex, setActiveIndex] = useState(0);
  const listId = useId();

  if (!options.length) return null;

  const handleKeyDown = (e: React.KeyboardEvent) => {
    switch (e.key) {
      case 'ArrowDown':
        e.preventDefault();
        setActiveIndex(i => Math.min(i + 1, options.length - 1));
        break;
      case 'ArrowUp':
        e.preventDefault();
        setActiveIndex(i => Math.max(i - 1, 0));
        break;
      case 'Enter':
      case ' ':
        e.preventDefault();
        if (isOpen) onSelect(options[activeIndex]);
        setIsOpen(prev => !prev);
        break;
      case 'Escape':
        setIsOpen(false);
        break;
    }
  };

  return (
    <div
      role="combobox"
      aria-expanded={isOpen}
      aria-haspopup="listbox"
      aria-controls={listId}
      tabIndex={0}
      onKeyDown={handleKeyDown}
      onClick={() => setIsOpen(prev => !prev)}
    >
      <span>{options[activeIndex]}</span>
      {isOpen && (
        <ul id={listId} role="listbox">
          {options.map((option, index) => (
            <li
              key={option}
              role="option"
              aria-selected={index === activeIndex}
              onClick={() => {
                onSelect(option);
                setIsOpen(false);
              }}
            >
              {option}
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

## フォーカス管理

UIの状態が変化するとき — 特にモーダルとルート遷移の場合 — フォーカスは論理的に移動する必要がある。

### モーダルのフォーカス復元

> この例は初期フォーカスと復元をカバーする。完全なフォーカストラップ（モーダル内でのTab/Shift+Tabのサイクリング）については、動的コンテンツやネストされたポータルなどのエッジケースを処理する[`focus-trap-react`](https://github.com/focus-trap/focus-trap-react)のようなライブラリを使用すること。

```tsx
export function Modal({ isOpen, onClose, title, children }: { isOpen: boolean; onClose: () => void; title: string; children: React.ReactNode }) {
  const modalRef = useRef<HTMLDivElement>(null);
  const previousFocusRef = useRef<HTMLElement | null>(null);

  useEffect(() => {
    if (isOpen) {
      // 現在フォーカスされている要素を保存し、モーダルにフォーカスを移動
      previousFocusRef.current = document.activeElement as HTMLElement;
      modalRef.current?.focus();
    } else {
      // モーダルを開いた要素にフォーカスを復元
      previousFocusRef.current?.focus();
    }
  }, [isOpen]);

  if (!isOpen) return null;

  return (
    <div ref={modalRef} role="dialog" aria-modal="true" aria-labelledby="modal-title" tabIndex={-1} onKeyDown={e => e.key === 'Escape' && onClose()}>
      <h2 id="modal-title">{title}</h2>
      {children}
      <button onClick={onClose}>Close</button>
    </div>
  );
}
```

## 画像とアイコン

```tsx
// BAD: 装飾アイコンがラベルのない画像として読み上げられる
<img src="/icon.svg" />

// GOOD: 装飾画像をスクリーンリーダーから隠す
<img src="/decoration.png" alt="" aria-hidden="true" />

// GOOD: 説明的なaltテキストを持つ意味のある画像
<img src="/chart.png" alt="Monthly revenue increased 23% from January to March" />

// GOOD: アクセシブルなラベルを持つアイコンボタン
<button aria-label="Delete item">
  <TrashIcon aria-hidden="true" />
</button>
```

## モーション軽減

OS設定でモーション軽減をリクエストしたユーザーを尊重する。

```tsx
export function useReducedMotion(): boolean {
  const [prefersReduced, setPrefersReduced] = useState(false);

  useEffect(() => {
    const mq = window.matchMedia('(prefers-reduced-motion: reduce)');
    setPrefersReduced(mq.matches);
    const handler = (e: MediaQueryListEvent) => setPrefersReduced(e.matches);
    mq.addEventListener('change', handler);
    return () => mq.removeEventListener('change', handler);
  }, []);

  return prefersReduced;
}

// 使用例
export function AnimatedCard({ children }: { children: React.ReactNode }) {
  const reduceMotion = useReducedMotion();

  return (
    <div
      style={{
        transition: reduceMotion ? 'none' : 'transform 300ms ease'
      }}
    >
      {children}
    </div>
  );
}
```

## アンチパターン

```tsx
// BAD: キーボードサポートなしで非インタラクティブ要素にonClick
<div onClick={handleClick}>Click me</div>

// BAD: roleのないdivにaria-label
<div aria-label="Navigation">...</div>

// BAD: ラベルの代替としてplaceholderを使用
<input placeholder="Enter your email" />

// BAD: 正のtabIndexが予測不可能なタブ順序を作る
<button tabIndex={3}>Submit</button>

// BAD: フォーカス可能な要素にaria-hidden — キーボードユーザーが罠にはまる
<button aria-hidden="true">Open</button>

// BAD: キーボードハンドラーなしでdivにrole="button"
<div role="button" onClick={handleClick}>Submit</div>
// 欠落：tabIndex={0}、Enter/SpaceのためのonKeyDown
```

## チェックリスト

レビュー提出前に任意のインタラクティブコンポーネントを確認：

- [ ] すべての`<input>`、`<select>`、`<textarea>`が`htmlFor`/`id`経由で`<label>`に接続されている
- [ ] エラーメッセージが`aria-describedby`でリンクされ`role="alert"`でマークされている
- [ ] `role`、`tabIndex`、`onKeyDown`なしで`<div>`または`<span>`に`onClick`がない
- [ ] アイコンのみのボタンに`aria-label`がある
- [ ] 装飾的な画像に`alt=""`と`aria-hidden="true"`が使われている
- [ ] モーダルがクローズ時にフォーカスを復元する（Tab/Shift+Tabサイクリングを含む完全なフォーカストラップには`focus-trap-react`のようなライブラリを使用する）
- [ ] 動的コンテンツの更新に`aria-live`を使用している
- [ ] アニメーションで`prefers-reduced-motion`を尊重している

## 関連スキル

- `frontend-patterns` — 一般的なReactコンポーネントと状態パターン
- `design-system` — デザイントークンとコンポーネントの一貫性
- `motion-ui` — アクセシビリティを考慮したアニメーションパターン
