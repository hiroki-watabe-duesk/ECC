---
name: motion-ui
description: "React/Next.js 向けプロダクション対応UIモーションシステム。アニメーション・トランジション・モーションパターンの実装時に使用してください。"
origin: ECC
---

# モーションシステム v4.2

React / Next.js 向けプロダクション対応 UI モーションシステム。

**パフォーマンス・アクセシビリティ・ユーザビリティ**に特化しており、装飾目的ではありません。

## 使用する場面

以下のような場合にこのモーションシステムを使用してください:

* 注意を誘導する（例: オンボーディング、重要なアクション）
* 状態を伝達する（ローディング、成功、エラー、トランジション）
* 空間的な連続性を保持する（レイアウト変更、ナビゲーション）

### 適切なシナリオ

* インタラクティブなコンポーネント（ボタン、モーダル、メニュー）
* 状態遷移（ローディング→完了、開く→閉じる）
* ナビゲーションとレイアウトの連続性（共有要素、クロスフェード）

### 注意事項

* **アクセシビリティ**: 動作軽減設定を常にサポートする
* **デバイス適応**: 低スペックデバイス向けに調整する
* **パフォーマンスのトレードオフ**: 視覚的な滑らかさより応答性を優先する

### モーションを使わない場面

* 純粋に装飾目的である場合
* ユーザビリティや明確さを損なう場合
* パフォーマンスに悪影響を与える場合

---

## 仕組み

### 基本原則

モーションは以下を実現しなければなりません:

* 注意の誘導
* 状態の伝達
* 空間的な連続性の保持

いずれも満たさない場合 → 削除する。

---

### インストール

```bash
npm install motion
```

---

### バージョン

* `motion/react` - 現在の Motion for React プロジェクトのデフォルト（パッケージ: `motion`）
* `framer-motion` - まだ Framer Motion に依存しているプロジェクト向けのレガシーインポートパス

**混在禁止。** 混在させると内部スケジューラーが競合し `AnimatePresence` コンテキストが壊れます — 一方のパッケージのコンポーネントは、もう一方のコンポーネントの終了アニメーションと連携しません。

プロジェクトが使用しているバージョンの確認方法:

```bash
cat package.json | grep -E '"motion"|"framer-motion"'
```

常に一つのソースから一貫してインポートしてください:

```ts
// 正しい（モダン）
import { motion, AnimatePresence } from "motion/react"

// 正しい（レガシー）
import { motion, AnimatePresence } from "framer-motion"

// 同一プロジェクトで両方を混在させないこと
```

---

### モーショントークン

```ts
// motionTokens.ts
export const motionTokens = {
  duration: {
    fast: 0.18,
    normal: 0.35,
    slow: 0.6
  },
  // Use these as the `ease` value inside a `transition` object:
  // transition={{ duration: motionTokens.duration.normal, ease: motionTokens.easing.smooth }}
  easing: {
    smooth: [0.22, 1, 0.36, 1] as [number, number, number, number],
    sharp:  [0.4,  0, 0.2, 1] as [number, number, number, number]
  },
  distance: {
    sm: 8,
    md: 16,
    lg: 24
  }
}
```

使用例:

```tsx
import { motionTokens } from "@/lib/motionTokens"

<motion.div
  initial={{ opacity: 0, y: motionTokens.distance.md }}
  animate={{ opacity: 1, y: 0 }}
  transition={{
    duration: motionTokens.duration.normal,
    ease: motionTokens.easing.smooth
  }}
/>
```

---

### パフォーマンスルール

**安全**

* transform
* opacity

**避ける**

* width / height
* top / left

ルール: 応答性 > 滑らかさ

---

### デバイス適応

このヒューリスティックは CPU コア数と利用可能なメモリの両方を組み合わせ、より信頼性の高い判定を行います。`deviceMemory` は Chrome/Android で利用可能であり、Safari や Firefox にはフォールバックが適用されます。

```ts
const isLowEnd =
  typeof navigator !== "undefined" && (
    // Low memory (Chrome/Android only; undefined elsewhere → treat as capable)
    (navigator.deviceMemory !== undefined && navigator.deviceMemory <= 2) ||
    // Few cores AND no memory API (covers Safari/Firefox on weak hardware)
    (navigator.deviceMemory === undefined && navigator.hardwareConcurrency <= 4)
  )

const duration = isLowEnd ? 0.2 : 0.4
```

---

### アクセシビリティ

#### JS（useReducedMotion）

```tsx
import { motion, useReducedMotion } from "motion/react"

export function FadeIn() {
  const reduce = useReducedMotion()

  return (
    <motion.div
      initial={{ opacity: 0, y: reduce ? 0 : 24 }}
      animate={{ opacity: 1, y: 0 }}
    />
  )
}
```

#### CSS

```css
@media (prefers-reduced-motion: reduce) {
  .motion-safe-transition {
    transition: opacity 0.2s;
  }

  .motion-reduce-transform {
    transform: none !important;
  }
}
```

#### Tailwind

```html
<div class="motion-safe:animate-fade motion-reduce:opacity-100"></div>
```

---

### アーキテクチャとパターン

#### コアパターン

| シナリオ | パターン |
|---|---|
| ホバーフィードバック | `whileHover` |
| タップ / プレスフィードバック | `whileTap` |
| スクロール時に表示 | `whileInView` |
| スクロール連動値 | `useScroll` + `useTransform` |
| 条件付きマウント / アンマウント | `AnimatePresence` |
| 小さなレイアウト変化（単一要素、〜300px未満の変化） | `layout` プロパティ |
| 大きなレイアウト変化またはページ全体のリフロー | `layout` を避ける。CSSトランジションまたはページレベルのルーティングを使用する |
| 複雑な命令型シーケンス | `useAnimate` |

> **大きなコンテナで `layout` を避ける理由:** Framer のレイアウトアニメーションは `transform` を使用して位置を調整しますが、ビューポート全体に広がる要素や深いリフローをトリガーする要素では、測定コストが顕著なジャンクや CLS を引き起こします。CSS Grid/Flexbox のトランジションを優先するか、特定の子要素にのみ `layoutId` を使用して連携してください。

#### レイアウトとトランジション

* 共有要素のトランジション → `layoutId`（マウントされたインスタンスごとに一意である必要がある）
* 入場 / 退場トランジション → `AnimatePresence`（以下の `mode` ガイダンスを参照）

#### AnimatePresence の `mode`

`mode` を常に明示的に指定してください — デフォルト（`"sync"`）は入場と退場を同時に実行するため、ほとんどの UI パターンで視覚的な重なりが生じます。

| `mode` | 使用する場面 |
|---|---|
| `"wait"` | 退場が完了してから入場が開始する。**モーダル、トースト、ページトランジション**に使用。 |
| `"sync"`（デフォルト） | 入場と退場が重なる。重なりが意図的な場合にのみ使用（例: クロスフェードカルーセル）。 |
| `"popLayout"` | 退場する要素がすぐにフローから除外され、残りのアイテムがアニメーションで埋まる。**リスト、タブ、削除可能なカード**に使用。 |

```tsx
// Modal — always use "wait"
<AnimatePresence mode="wait">
  {open && <Modal key="modal" />}
</AnimatePresence>

// Dismissible list item — use "popLayout"
<AnimatePresence mode="popLayout">
  {items.map(item => <Card key={item.id} />)}
</AnimatePresence>
```

---

### 高度なパターン（概念）

* パララックス（スクロール連動トランスフォーム）
* スクロールストーリーテリング（スティッキーセクション）
* 3D チルト（ポインターベースのトランスフォーム）
* クロスフェード（共有 `layoutId`）
* プログレッシブリビール（クリップパス）
* スケルトンローディング（ループするオパシティ）
* マイクロインタラクション（ホバー / タップフィードバック）
* スプリングシステム（物理ベースのモーション）

---

### モーダルの基本要件

* フォーカストラップ
* Escape キーで閉じる
* スクロールロック
* ARIA ロール
* 次のモーダルが入場する前に退場アニメーションが完了するよう `AnimatePresence mode="wait"` を使用する

#### 完全なサンプル

```tsx
import React, { useEffect, useRef, useState } from "react"
import { motion, AnimatePresence } from "motion/react"

function useFocusTrap(ref: React.RefObject<HTMLDivElement | null>, active: boolean) {
  useEffect(() => {
    if (!active || !ref.current) return
    const el = ref.current
    const focusable = el.querySelectorAll<HTMLElement>(
      'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
    )
    const first = focusable[0]
    const last  = focusable[focusable.length - 1]

    function handleKey(e: KeyboardEvent) {
      if (e.key !== "Tab") return
      if (e.shiftKey && document.activeElement === first) {
        e.preventDefault()
        last?.focus()
      } else if (!e.shiftKey && document.activeElement === last) {
        e.preventDefault()
        first?.focus()
      }
    }

    el.addEventListener("keydown", handleKey)
    first?.focus()
    return () => el.removeEventListener("keydown", handleKey)
  }, [active, ref])
}

function useScrollLock(active: boolean) {
  useEffect(() => {
    if (!active) return
    const prev = document.body.style.overflow
    document.body.style.overflow = "hidden"
    return () => { document.body.style.overflow = prev }
  }, [active])
}

function Modal({ open, closeModal }: { open: boolean; closeModal: () => void }) {
  const ref = useRef<HTMLDivElement>(null)

  useFocusTrap(ref, open)
  useScrollLock(open)

  useEffect(() => {
    function onKey(e: KeyboardEvent) {
      if (e.key === "Escape") closeModal()
    }
    if (open) window.addEventListener("keydown", onKey)
    return () => window.removeEventListener("keydown", onKey)
  }, [open, closeModal])

  return (
    // mode="wait" ensures exit animation finishes before any new modal enters
    <AnimatePresence mode="wait">
      {open && (
        <motion.div
          role="dialog"
          aria-modal="true"
          aria-labelledby="modal-title"
          initial={{ opacity: 0 }}
          animate={{ opacity: 1 }}
          exit={{ opacity: 0 }}
          transition={{ duration: 0.2 }}
          className="fixed inset-0 flex items-center justify-center bg-black/40"
        >
          <motion.div
            ref={ref}
            initial={{ scale: 0.95, opacity: 0 }}
            animate={{ scale: 1,    opacity: 1 }}
            exit={{    scale: 0.95, opacity: 0 }}
            transition={{ duration: 0.2, ease: [0.22, 1, 0.36, 1] }}
            className="bg-white p-6 rounded"
          >
            <h2 id="modal-title">Dialog Title</h2>
            <button onClick={closeModal}>Close</button>
          </motion.div>
        </motion.div>
      )}
    </AnimatePresence>
  )
}

export function Example() {
  const [open, setOpen] = useState(false)

  return (
    <>
      <button onClick={() => setOpen(true)}>Open</button>
      <Modal open={open} closeModal={() => setOpen(false)} />
    </>
  )
}
```

---

### SSR の安全性

* サーバーとクライアントのレンダー間で初期状態を一致させる
* 暗黙的なアニメーション起点を避ける（常に `initial` を明示的に設定する）
* Next.js App Router では `"use client"` 内に motion コンポーネントをラップする

---

### デバッグ

以下を確認してください:

* 誤ったインポート（`motion/react` と `framer-motion` の混在）
* Next.js App Router で `"use client"` ディレクティブが欠けている
* `AnimatePresence` の子に `key` プロパティがない
* ハイドレーションの不一致（SSR とクライアントで初期状態が異なる）
* 大きなコンテナでの `layout` プロパティの誤用によるリフロージャンク
* 状態駆動のアニメーションが発火しない（依存配列を確認する）

---

### QA

* CLS がない
* キーボード操作が可能
* モーダル内でフォーカスがトラップされている
* ARIA ロールが正しい（`role="dialog"`、`aria-modal="true"`）
* 動作軽減設定が尊重されている（`useReducedMotion` + CSS メディアクエリ）
* Next.js でハイドレーションの警告がない
* アンマウント時にアニメーションがクリーンに停止する（メモリリークなし）
* すべての使用箇所で `AnimatePresence mode` が明示的に設定されている

---

### アンチパターン

* レイアウトプロパティのアニメーション（`width`、`height`、`top`、`left`）
* 目的のない無限アニメーション（常に問う: どの状態を伝えているのか？）
* リストの過剰なスタガー（`staggerChildren` を 0.1s 以下に保つ; それ以上は遅く感じる）
* 動作軽減設定の無視
* 大きなまたはビューポート全体のコンテナに `layout` を使用する
* `AnimatePresence` で `mode` を省略する（デフォルトの `"sync"` は視覚的な重なりを引き起こす）
* 純粋に装飾目的でモーションを使用する

---

### フィロソフィー

モーションはインタラクションデザインです。

---

### 最終ルール

> モーションが UX を改善しないなら → 削除する。

---

## サンプル

### ボタンインタラクション

```tsx
import { motion } from "motion/react"

export function Button() {
  return (
    <motion.button
      whileHover={{ scale: 1.02 }}
      whileTap={{ scale: 0.97 }}
      transition={{ duration: 0.15, ease: [0.4, 0, 0.2, 1] }}
    >
      Click me
    </motion.button>
  )
}
```

---

### 動作軽減のサンプル

```tsx
import { motion, useReducedMotion } from "motion/react"

export function FadeIn() {
  const reduce = useReducedMotion()

  return (
    <motion.div
      initial={{ opacity: 0, y: reduce ? 0 : 24 }}
      animate={{ opacity: 1, y: 0 }}
      transition={{ duration: reduce ? 0.1 : 0.35, ease: [0.22, 1, 0.36, 1] }}
    />
  )
}
```

---

### スタガーリスト

```tsx
import { motion } from "motion/react"

const container = {
  hidden: {},
  visible: {
    transition: { staggerChildren: 0.08 } // keep ≤ 0.1s to avoid sluggishness
  }
}

const item = {
  hidden:  { opacity: 0, y: 10 },
  visible: { opacity: 1, y: 0,  transition: { duration: 0.3, ease: [0.22, 1, 0.36, 1] } }
}

export function List() {
  return (
    <motion.ul variants={container} initial="hidden" animate="visible">
      {[1, 2, 3].map(i => (
        <motion.li key={i} variants={item}>Item {i}</motion.li>
      ))}
    </motion.ul>
  )
}
```

---

### AnimatePresence を使用したモーダル

```tsx
import { motion, AnimatePresence } from "motion/react"

export function Modal({ open }: { open: boolean }) {
  return (
    <AnimatePresence mode="wait">
      {open && (
        <motion.div
          initial={{ opacity: 0, scale: 0.95 }}
          animate={{ opacity: 1, scale: 1    }}
          exit={{    opacity: 0, scale: 0.95 }}
          transition={{ duration: 0.2, ease: [0.22, 1, 0.36, 1] }}
        />
      )}
    </AnimatePresence>
  )
}
```

---

### スクロールパララックス

```tsx
import { useScroll, useTransform, motion } from "motion/react"

export function Parallax() {
  const { scrollYProgress } = useScroll()
  const y = useTransform(scrollYProgress, [0, 1], [0, -80])

  return <motion.div style={{ y }} />
}
```

---

### スケルトンローディング

```tsx
import { motion } from "motion/react"

export function Skeleton() {
  return (
    <motion.div
      className="bg-gray-200 h-6 w-full rounded"
      animate={{ opacity: [0.5, 1, 0.5] }}
      transition={{
        duration: 1.5,       // comfortable pulse — was missing, caused fast flash
        repeat: Infinity,
        ease: "easeInOut"
      }}
    />
  )
}
```

---

### 共有レイアウト（クロスフェード）

```tsx
import { motion } from "motion/react"

// layoutId must be unique per mounted instance.
// If multiple instances can exist simultaneously, append a unique id:
// layoutId={`shared-${item.id}`}
export function Shared() {
  return <motion.div layoutId="shared" />
}
```
