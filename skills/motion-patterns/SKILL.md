---
name: motion-patterns
description: React / Next.js 向けのプロダクション対応アニメーションパターン — ボタン、モーダル、トースト、スタガー、ページトランジション、退場アニメーション、スクロール、レイアウト — motion-foundations トークンとスプリングで構築。
version: 1.0
tags: [motion, animation, ui-patterns]
category: frontend
author: jeff
---

# モーションパターン

最も一般的な UI アニメーションニーズのためのコピー＆ペーストパターン。
ここにあるすべてのパターンは `motion-foundations` トークンとスプリングで構築されています。
ここに新しい duration または easing の値を定義しないでください — インポートして使用してください。

## アクティブにするタイミング

- ボタン、カード、モーダル、またはトースト通知をアニメーションさせるとき
- スタガーを使ったリストのエントランスを構築するとき
- Next.js App Router でページトランジションを設定するとき
- 条件付きコンテンツにエントランスまたは退場アニメーションを追加するとき
- スクロール表示、スクロール連動プログレス、またはスティッキーストーリーセクションを実装するとき
- 展開カード、アコーディオン、または共有要素トランジションを構築するとき

## 出力

このスキルは以下を生成します:

- すべての標準 UI コンポーネントに対するアクセシブルで SSR セーフなアニメーション
- 正しい退場動作を持つ `AnimatePresence` でラップされた条件付きレンダー
- Next.js App Router 向けのページトランジションラッパーコンポーネント
- `useScroll` + `useTransform` を使用したスクロール表示とスクロール連動パターン
- 展開とクロスフェードのためのレイアウトアニメーションパターン（`layout`、`layoutId`）

## 原則

- すべてのパターンは `motion-foundations` からインポートします。生の数値は使用しません。
- すべての条件付きレンダーは `key` を持つ `AnimatePresence` でラップされます。
- 退場アニメーションは常にエントランスアニメーションと一緒に定義されます — 後付けにしません。
- `layout` は小さく孤立した変化のみに使用します。大きなサブツリーには明示的なトランスフォームを使用します。

## ルール

1. **常に `AnimatePresence` で条件付きレンダーをラップし、直接の子に `key` を付ける。** key がなければ退場アニメーションは実行されません。
2. **`initial` + `animate` を定義する場合は常に `exit` も定義する。** 退場のないアニメーションは不完全です。
3. **ページトランジションには `mode="wait"` を使用する。** 退場が完了するまでエントランスを開始しないでください。
4. **5 つ以上の子または深くネストされた DOM を持つサブツリーには `layout` を使用しない。** 代わりに明示的な `x`/`y` トランスフォームを使用してください。
5. **スタガーインターバルは `0.05s` から `0.10s` の間に維持する。** 以下では機械的に感じ、以上では緩慢に感じます。
6. **モーダルには常に以下を含める:** フォーカストラップ、Escape キーによるクローズ、スクロールロック、`role="dialog"`、`aria-modal="true"`。
7. **スクロール表示には `viewport={{ once: true }}` を使用する。** スクロールアウト時に繰り返すのは有益ではなく、気が散ります。
8. **すべてのトークン値は `motion-foundations` からインポートする。** インライン数値は使用しません。

## 判断ガイダンス

### 適切なパターンの選択

| 状況 | パターン |
| ---------------------------------------- | ---------------------- |
| 要素が現れる / 消える                    | `AnimatePresence`      |
| アイテムのリストが順番に読み込まれる     | スタガーバリアント     |
| ルート間のナビゲーション                 | ページトランジションラッパー |
| 要素がその場でサイズ変化する             | `layout` プロップ      |
| 同じ要素がページコンテキストをまたいで移動 | `layoutId`           |
| スクロールして表示エリアに入ったときに要素が現れる | `whileInView`  |
| 値がスクロール位置に連動する             | `useScroll` + `useTransform` |

### `mode="wait"` と `mode="sync"` の使い分け

| モード | 使用するとき |
| ------- | --------------------------------------- |
| `wait` | ページトランジション、コンテンツのスワップ（一度に一つ） |
| `sync` | 積み重ねられた通知、リストアイテム（重なりが許容される） |
| `popLayout` | リフローリストから削除されるアイテム |

## 中核コンセプト

### AnimatePresence コントラクト

常に以下の 3 つが満たされている必要があります:

1. `AnimatePresence` が条件をラップしている
2. 直接の子に `key` がある
3. 子に `exit` プロップがある

これらのうちいずれか 1 つでも欠けると、退場アニメーションはサイレントに失敗します。

### layout と layoutId の比較

- `layout` — 要素自身のサイズ/位置変化をその場でアニメーションする
- `layoutId` — 2 つの別々の要素を連結し、レンダー間でクロスフェードする

展開するコンテナ内のテキストに `layout="position"` を使用して、テキストのリフローがアニメーションしないようにします。

## コード例

### ボタンフィードバック

```tsx
"use client"
import { motion } from "motion/react"
import { springs, motionTokens } from "@/lib/motion-tokens"

<motion.button
  whileHover={{ scale: motionTokens.scale.pop }}
  whileTap={{ scale: motionTokens.scale.press }}
  transition={springs.snappy}
/>
```

### スタガーリスト

```tsx
"use client"
import { motion } from "motion/react"
import { motionTokens, springs } from "@/lib/motion-tokens"

const container = {
  hidden: {},
  visible: {
    transition: {
      staggerChildren: 0.08,   // within the 0.05–0.10 rule
      delayChildren: 0.1,
    },
  },
}

const item = {
  hidden:  { opacity: 0, y: motionTokens.distance.md },
  visible: { opacity: 1, y: 0, transition: springs.gentle },
}

<motion.ul variants={container} initial="hidden" animate="visible">
  {items.map((i) => (
    <motion.li key={i.id} variants={item} />
  ))}
</motion.ul>
```

### モーダル

```tsx
"use client"
import { motion, AnimatePresence } from "motion/react"
import { motionTokens, springs } from "@/lib/motion-tokens"

// Wrap at the call site:
// <AnimatePresence>{isOpen && <Modal key="modal" />}</AnimatePresence>

export function Modal({ onClose }: { onClose: () => void }) {
  return (
    <>
      {/* Overlay */}
      <motion.div
        className="fixed inset-0 bg-black/50"
        initial={{ opacity: 0 }}
        animate={{ opacity: 1 }}
        exit={{ opacity: 0 }}
        onClick={onClose}
      />

      {/* Panel — accessibility requirements: focus trap, Escape close,
          scroll lock, role="dialog", aria-modal="true" */}
      <motion.div
        role="dialog"
        aria-modal="true"
        className="fixed inset-x-4 top-1/2 -translate-y-1/2 rounded-xl bg-white p-6"
        initial={{
          opacity: 0,
          scale: motionTokens.scale.press,
          y: motionTokens.distance.sm,
        }}
        animate={{ opacity: 1, scale: 1, y: 0 }}
        exit={{
          opacity: 0,
          scale: motionTokens.scale.press,
          y: motionTokens.distance.sm,
        }}
        transition={springs.gentle}
      />
    </>
  )
}
```

### トーストスタック

```tsx
"use client"
import { motion, AnimatePresence } from "motion/react"
import { motionTokens, springs } from "@/lib/motion-tokens"

<AnimatePresence mode="sync">
  {toasts.map((t) => (
    <motion.div
      key={t.id}
      layout
      initial={{
        opacity: 0,
        x: motionTokens.distance.xl,
        scale: motionTokens.scale.subtle,
      }}
      animate={{ opacity: 1, x: 0, scale: 1 }}
      exit={{
        opacity: 0,
        x: motionTokens.distance.xl,
        scale: motionTokens.scale.subtle,
      }}
      transition={springs.snappy}
    />
  ))}
</AnimatePresence>
```

### ページトランジション（Next.js App Router）

```tsx
// components/page-transition.tsx
"use client"
import { motion, AnimatePresence } from "motion/react"
import { usePathname } from "next/navigation"
import { motionTokens } from "@/lib/motion-tokens"

const variants = {
  initial: { opacity: 0, y: motionTokens.distance.sm },
  enter:   { opacity: 1, y: 0 },
  exit:    { opacity: 0, y: -motionTokens.distance.sm },
}

export function PageTransition({ children }: { children: React.ReactNode }) {
  const pathname = usePathname()
  return (
    <AnimatePresence mode="wait">
      <motion.div
        key={pathname}
        variants={variants}
        initial="initial"
        animate="enter"
        exit="exit"
        transition={{
          duration: motionTokens.duration.normal,
          ease: motionTokens.easing.smooth,
        }}
      >
        {children}
      </motion.div>
    </AnimatePresence>
  )
}
```

### スクロール表示

```tsx
"use client"
import { motion } from "motion/react"
import { motionTokens, springs } from "@/lib/motion-tokens"

<motion.div
  initial={{ opacity: 0, y: motionTokens.distance.lg }}
  whileInView={{ opacity: 1, y: 0 }}
  viewport={{ once: true, margin: "-80px" }}   // once: true — rule 7
  transition={{ duration: motionTokens.duration.slow, ease: motionTokens.easing.smooth }}
/>
```

### スクロールプログレスバー

```tsx
"use client"
import { motion, useScroll } from "motion/react"

export function ScrollProgress() {
  const { scrollYProgress } = useScroll()
  return (
    <motion.div
      className="fixed top-0 left-0 h-1 bg-indigo-500 origin-left w-full"
      style={{ scaleX: scrollYProgress }}
    />
  )
}
```

### 展開カード

```tsx
"use client"
import { useState } from "react"
import { motion, AnimatePresence } from "motion/react"
import { springs, motionTokens } from "@/lib/motion-tokens"

export function ExpandingCard({ title, body }: { title: string; body: string }) {
  const [expanded, setExpanded] = useState(false)

  return (
    <motion.div layout onClick={() => setExpanded(!expanded)} className="cursor-pointer">
      {/* layout="position" prevents text reflow from animating */}
      <motion.h2 layout="position" className="font-semibold">
        {title}
      </motion.h2>

      <AnimatePresence>
        {expanded && (
          <motion.p
            key="body"
            initial={{ opacity: 0 }}
            animate={{ opacity: 1 }}
            exit={{ opacity: 0 }}
            transition={{ duration: motionTokens.duration.fast }}
          >
            {body}
          </motion.p>
        )}
      </AnimatePresence>
    </motion.div>
  )
}
```

### 共有要素クロスフェード

```tsx
// Source context
<motion.img layoutId="hero-image" src={src} className="w-16 h-16 rounded" />

// Destination context (same layoutId — motion handles the transition)
<motion.img layoutId="hero-image" src={src} className="w-full rounded-xl" />
```

### アコーディオン

```tsx
<motion.div
  initial={false}
  animate={{ opacity: open ? 1 : 0, scaleY: open ? 1 : 0 }}
  style={{ transformOrigin: "top", overflow: "hidden" }}
  transition={{
    duration: motionTokens.duration.normal,
    ease: motionTokens.easing.smooth,
  }}
>
  {children}
</motion.div>
```

## エンドツーエンドの例

マウント時にスタガーで入場し、条件付きプレゼンスを処理し、
reduced motion を考慮したスタガーリスト — トークン、スプリング、AnimatePresence、
`motion-foundations` からのアクセシビリティフックを組み合わせます:

```tsx
"use client"
import { useState } from "react"
import { motion, AnimatePresence } from "motion/react"
import { motionTokens, springs } from "@/lib/motion-tokens"
import { useSafeMotion } from "@/hooks/use-reduced-motion"

const containerVariants = {
  hidden: {},
  visible: {
    transition: { staggerChildren: 0.08, delayChildren: 0.1 },
  },
}

function ListItem({ label, onRemove }: { label: string; onRemove: () => void }) {
  const safe = useSafeMotion(motionTokens.distance.sm)
  return (
    <motion.li
      variants={{
        hidden:  safe.initial,
        visible: safe.animate,
      }}
      exit={safe.exit}
      transition={springs.gentle}
      className="flex items-center justify-between p-3 rounded-lg bg-white shadow-sm"
    >
      <span>{label}</span>
      <button onClick={onRemove}>Remove</button>
    </motion.li>
  )
}

export function AnimatedList({ items, onRemove }: {
  items: { id: string; label: string }[]
  onRemove: (id: string) => void
}) {
  return (
    <motion.ul
      variants={containerVariants}
      initial="hidden"
      animate="visible"
      className="space-y-2"
    >
      <AnimatePresence mode="popLayout">
        {items.map((item) => (
          <ListItem
            key={item.id}
            label={item.label}
            onRemove={() => onRemove(item.id)}
          />
        ))}
      </AnimatePresence>
    </motion.ul>
  )
}
```

## 制約 / 非目標

このスキルは以下をカバーしません:

- トークンとスプリングの定義 → `motion-foundations` を参照
- ドラッグインタラクション、スワイプジェスチャー、並び替え可能なリスト → `motion-advanced` を参照
- テキストアニメーション（単語/文字の表示、カウンター） → `motion-advanced` を参照
- SVG パスの描画またはモーフィング → `motion-advanced` を参照
- カスタムアニメーションフック → `motion-advanced` を参照
- `motion/react` を使用しない CSS のみのトランジション

## アンチパターン

| アンチパターン | 違反ルール | 修正方法 |
| -------------------------------------------- | ------- | ------------------------------------------ |
| `AnimatePresence` の子に `key` がない | ルール 1 | 直接の子に安定した `key` を追加する |
| `exit` なしの `initial` + `animate` | ルール 2 | 常に 3 つをセットで定義する |
| `mode="wait"` なしのページトランジション | ルール 3 | `AnimatePresence` に `mode="wait"` を追加する |
| 50 アイテムリストへの `layout` | ルール 4 | `mode="popLayout"` または明示的なトランスフォームを使用する |
| 10 アイテムリストへの `staggerChildren: 0.2` | ルール 5 | `0.08〜0.10` に制限する |
| フォーカストラップなしのモーダル | ルール 6 | `focus-trap-react` または Radix Dialog を追加する |
| `viewport={{ once: true }}` なしの `whileInView` | ルール 7 | 繰り返しエントランスは有益ではなく気が散る |
| インラインの `transition={{ duration: 0.3 }}` | ルール 8 | `motionTokens.duration.normal` を使用する |

## 関連スキル

- **`motion-foundations`** — ここにあるすべてのパターンがインポートするトークン、スプリング、`useSafeMotion` フック、SSR ガードをすべて定義しています。最初にセットアップする必要があります。
- **`motion-advanced`** — ドラッグ、ジェスチャー、SVG、テキスト、カスタムフック、命令型シーケンスでこれらのパターンを拡張します。このスキルのパターンは再定義しません。
