---
name: motion-advanced
description: React / Next.js向けの高度なモーションパターン。ドラッグ＆ドロップ、ジェスチャー、テキストアニメーション、SVGパス描画、カスタムフック、命令的シーケンス（useAnimate）、ローダー、APIデシジョンツリーを含む。motion-foundationsが必要。
version: 1.0
tags: [motion, animation, advanced, gestures, svg]
category: frontend
author: jeff
---

# Motion Advanced

複雑でインタラクティブな、物理ベースのアニメーションパターン。
事前に`motion-foundations`のセットアップが必要です。
`motion-patterns`では不十分なときに使用してください。

## アクティブにするタイミング

- ドラッグで消えるシート、スワイプジェスチャー、並べ替えリストを構築するとき
- テキストを単語ごと、文字ごと、またはライブカウンターとしてアニメーションするとき
- SVGパスを描画したり、アイコンをモーフィングしたり、円形プログレスをアニメーションするとき
- カスタムアニメーションフック（`useScrollReveal`、マグネティックボタン、カーソルフォロワー）を書くとき
- `useAnimate`を使って複数ステップのアニメーションを命令的にシーケンス化するとき
- スピナー、シマースケルトン、パルスインジケーター、またはローディングボタン状態を構築するとき

## 出力

このスキルが生成するもの:

- ドラッグインタラクション: ドラッグ可能なカード、ドラッグで消えるシート、`Reorder.Group`リスト
- ジェスチャーフック: スワイプ検出、ロングプレス、ピンチアウトライン
- テキストアニメーションコンポーネント: 単語リビール、文字タイプライター、数値カウンター
- SVGアニメーション: パス描画、アイコンモーフ、ストロークプログレスリング
- カスタムフック: `useScrollReveal`、`useHoverScale`、`useNavigationDirection`、`useInViewOnce`
- 割り込み安全な`async/await`を使った`useAnimate`による命令的シーケンス
- ローダーコンポーネント: スピナー、シマー、パルスドット、プログレスバー、ボタンローディング状態

## 原則

- 物理ベースのモーション（`useSpring`、`springs.*`）は、直接操作においては常に時間ベースよりも自然に感じられる。
- `useMotionValue` + `useTransform`は、再レンダーを発生させずに派生値を計算する。
- `useAnimate`シーケンスは命令的で割り込み安全 — 実行中に`animate()`を呼び出すと自動的に前のアニメーションがキャンセルされる。
- モーション値（`useMotionValue`、`useSpring`）はSSRセーフで、ハイドレーションエラーを起こさない。

## ルール

1. **ドラッグインタラクションはタッチデバイスでテストすること**（マウスだけでなく）。`drag`プロップは両方で動作するが、感触としきい値が異なる。
2. **無限アニメーションは`document.visibilityState === "hidden"`のときに一時停止すること。** バックグラウンドタブはGPU/CPUを消費してはならない。
3. **スワイプのしきい値は明示的に設定すること。** 速度だけから意図を推測しない。`offset` + `velocity`の両方を確認する。
4. **`useAnimate`のスコープrefはマウントされたDOM要素に紐付けること。** マウント前に`animate()`を呼び出すとサイレントにエラーになる。
5. **モーション値はレンダーごとに再作成してはならない。** コンポーネント本体内の`useMotionValue(0)`は正しい。レンダー内の`new MotionValue(0)`は誤り。
6. **全トークン値は`motion-foundations`からインポートすること。** インライン数値禁止。
7. **カスタムフックはクリーンアップを処理すること。** `window.addEventListener`は全て`useEffect`の戻り値で対応する`removeEventListener`が必要。
8. **SVGモーフィングは同数のパスコマンドが必要。** コマンド構造が異なるパスはアニメーションせずにスナップする。

## 判断の指針

### 適切な高度なAPIの選び方

| シナリオ | API |
| ------------------------------ | -------------------------------- |
| 離した後の物理ドラッグ | `drag` + `dragTransition: springs.release` |
| 順番付きドラッグで並べ替えリスト | `Reorder.Group` + `Reorder.Item` |
| ドラッグオフセットで消える | `drag="y"` + `onDragEnd`のオフセット確認 |
| 左右スワイプ | `drag="x"` + `onDragEnd`のオフセット確認 |
| ロングプレス | `useLongPress`フック |
| 時間的に平滑化された値 | `useSpring` |
| 別の値から派生した値 | `useTransform` |
| 複数ステップシーケンス | `useAnimate`と`async/await` |
| 一発の命令的アニメーション | `motion`からの`animate()` |
| テキストを単語ごとに表示 | `inline-block`スパンのスタガー |
| SVG描画 | `pathLength` 0 → 1 |
| SVGモーフ | `d`属性トウィーン（同数コマンド） |
| 円形プログレス | `strokeDashoffset`トウィーン |

### `useSpring`とスプリングトランジションの使い分け

| | `useSpring` | `transition: springs.*` |
| -------------- | ---------------------------------------- | ----------------------- |
| 使い所 | カーソルフォロワー、ポインタ追跡値 | 離散的な状態変化 |
| 更新 | 毎フレーム連続 | 状態変化で起動 |
| 割り込み | スムーズ — 物理が速度から引き継ぐ | 現在値から再起動 |

## 核となる概念

### useMotionValue + useTransform

再レンダーなしのリアクティブ計算:

```tsx
const x = useMotionValue(0)
const opacity = useTransform(x, [-200, 0, 200], [0, 1, 0])
// opacityはxが変化するたびに毎フレーム更新される — setStateなし、再レンダーなし
```

### useAnimate

`[scope, animate]`を返す。スコープrefはDOM要素に紐付ける必要がある。
`animate()`呼び出しは割り込み安全 — 実行中に呼び出すと前の実行がキャンセルされる。

```tsx
const [scope, animate] = useAnimate()

async function play() {
  await animate(".step-1", { opacity: 1 }, { duration: 0.3 })
  await animate(".step-2", { x: 0 },       { duration: 0.4 })
        animate(".step-3", { scale: 1 },    { duration: 0.25 })  // fire and forget
}

return <div ref={scope}>...</div>
```

## コード例

### ドラッグ可能なカード

```tsx
"use client"
import { motion } from "motion/react"
import { springs, motionTokens } from "@/lib/motion-tokens"

<motion.div
  drag
  dragConstraints={{ left: -100, right: 100, top: -100, bottom: 100 }}
  dragElastic={0.1}
  whileDrag={{
    scale: motionTokens.scale.pop,
    boxShadow: "0 16px 40px rgba(0,0,0,0.2)",
  }}
  dragTransition={springs.release}
/>
```

### ドラッグで消えるシート

```tsx
"use client"
import { motion, useMotionValue, useTransform } from "motion/react"

export function BottomSheet({ onClose }: { onClose: () => void }) {
  const y = useMotionValue(0)
  const opacity = useTransform(y, [0, 200], [1, 0])

  return (
    <motion.div
      drag="y"
      dragConstraints={{ top: 0 }}
      style={{ y, opacity }}
      onDragEnd={(_, info) => {
        // ルール3: offsetとvelocityの両方を確認
        if (info.offset.y > 120 || info.velocity.y > 500) onClose()
      }}
    />
  )
}
```

### 並べ替え可能なリスト

```tsx
"use client"
import { Reorder } from "motion/react"

export function SortableList() {
  const [items, setItems] = useState(initialItems)
  return (
    <Reorder.Group axis="y" values={items} onReorder={setItems}>
      {items.map((item) => (
        <Reorder.Item key={item.id} value={item}>
          {item.label}
        </Reorder.Item>
      ))}
    </Reorder.Group>
  )
}
```

### スワイプ検出

```tsx
"use client"
import { motion } from "motion/react"

const OFFSET_THRESHOLD  = 50
const VELOCITY_THRESHOLD = 300

<motion.div
  drag="x"
  dragConstraints={{ left: 0, right: 0 }}
  onDragEnd={(_, info) => {
    const swipedRight = info.offset.x > OFFSET_THRESHOLD  || info.velocity.x > VELOCITY_THRESHOLD
    const swipedLeft  = info.offset.x < -OFFSET_THRESHOLD || info.velocity.x < -VELOCITY_THRESHOLD
    if (swipedRight) onSwipeRight()
    if (swipedLeft)  onSwipeLeft()
  }}
/>
```

### ロングプレスフック

```tsx
import { useRef } from "react"

export function useLongPress(callback: () => void, ms = 600) {
  const timerRef = useRef<ReturnType<typeof setTimeout>>()
  return {
    onPointerDown:  () => { timerRef.current = setTimeout(callback, ms) },
    onPointerUp:    () => clearTimeout(timerRef.current),
    onPointerLeave: () => clearTimeout(timerRef.current),
  }
}
```

### 単語ごとのリビール

```tsx
"use client"
import { motion } from "motion/react"
import { springs } from "@/lib/motion-tokens"

export function AnimatedText({ text }: { text: string }) {
  return (
    <motion.p
      variants={{ visible: { transition: { staggerChildren: 0.05 } } }}
      initial="hidden"
      animate="visible"
    >
      {text.split(" ").map((word, i) => (
        <motion.span
          key={i}
          className="inline-block mr-1"
          variants={{
            hidden:  { opacity: 0, y: 12 },
            visible: { opacity: 1, y: 0, transition: springs.gentle },
          }}
        >
          {word}
        </motion.span>
      ))}
    </motion.p>
  )
}
```

### 数値カウンター

```tsx
"use client"
import { useRef, useEffect } from "react"
import { animate } from "motion"
import { motionTokens } from "@/lib/motion-tokens"

export function Counter({ to }: { to: number }) {
  const nodeRef = useRef<HTMLSpanElement>(null)

  useEffect(() => {
    const controls = animate(0, to, {
      duration: motionTokens.duration.crawl,
      ease: motionTokens.easing.smooth,
      onUpdate: (v) => {
        if (nodeRef.current) nodeRef.current.textContent = Math.round(v).toString()
      },
    })
    return controls.stop   // ルール7: クリーンアップ
  }, [to])

  return <span ref={nodeRef} />
}
```

### SVGパスの描画

```tsx
"use client"
import { motion } from "motion/react"
import { motionTokens } from "@/lib/motion-tokens"

<motion.path
  d="M 0 100 Q 50 0 100 100"
  initial={{ pathLength: 0, opacity: 0 }}
  animate={{ pathLength: 1, opacity: 1 }}
  transition={{ duration: motionTokens.duration.slow, ease: motionTokens.easing.smooth }}
/>
```

### ストロークプログレスリング

```tsx
"use client"
import { motion } from "motion/react"
import { motionTokens } from "@/lib/motion-tokens"

const CIRCUMFERENCE = 2 * Math.PI * 40   // r=40

export function ProgressRing({ progress }: { progress: number }) {
  return (
    <svg width="100" height="100" viewBox="0 0 100 100">
      <circle cx="50" cy="50" r="40" fill="none" stroke="#e5e7eb" strokeWidth="8" />
      <motion.circle
        cx="50" cy="50" r="40"
        fill="none" stroke="#6366f1" strokeWidth="8"
        strokeLinecap="round"
        strokeDasharray={CIRCUMFERENCE}
        animate={{ strokeDashoffset: CIRCUMFERENCE - (progress / 100) * CIRCUMFERENCE }}
        transition={{ duration: motionTokens.duration.normal, ease: motionTokens.easing.smooth }}
        style={{ rotate: -90, transformOrigin: "center" }}
      />
    </svg>
  )
}
```

### useScrollRevealフック

```tsx
"use client"
import { useRef } from "react"
import { useScroll, useTransform } from "motion/react"
import { motionTokens } from "@/lib/motion-tokens"

export function useScrollReveal() {
  const ref = useRef(null)
  const { scrollYProgress } = useScroll({ target: ref, offset: ["start end", "end start"] })
  const opacity = useTransform(scrollYProgress, [0, 0.3], [0, 1])
  const y       = useTransform(scrollYProgress, [0, 0.3], [motionTokens.distance.lg, 0])
  return { ref, style: { opacity, y } }
}

// 使用例
const { ref, style } = useScrollReveal()
<motion.section ref={ref} style={style} />
```

### カーソルフォロワー

```tsx
"use client"
import { useEffect } from "react"
import { motion, useMotionValue, useSpring } from "motion/react"
import { springs } from "@/lib/motion-tokens"

export function CursorFollower() {
  const x = useMotionValue(-100)
  const y = useMotionValue(-100)
  const sx = useSpring(x, springs.gentle)
  const sy = useSpring(y, springs.gentle)

  useEffect(() => {
    const move = (e: MouseEvent) => { x.set(e.clientX); y.set(e.clientY) }
    window.addEventListener("mousemove", move)
    return () => window.removeEventListener("mousemove", move)   // ルール7
  }, [])

  return (
    <motion.div
      className="fixed top-0 left-0 w-6 h-6 rounded-full bg-indigo-500
                 pointer-events-none -translate-x-1/2 -translate-y-1/2 z-50"
      style={{ x: sx, y: sy }}
    />
  )
}
```

### シマースケルトン

```tsx
"use client"
import { useEffect } from "react"
import { motion, useAnimation } from "motion/react"
import { motionTokens } from "@/lib/motion-tokens"

export function ShimmerSkeleton({ className = "" }: { className?: string }) {
  const controls = useAnimation()

  useEffect(() => {
    const play = () =>
      controls.start({
        x: ["-100%", "100%"],
        transition: {
          repeat: Infinity,
          duration: motionTokens.duration.crawl,
          ease: motionTokens.easing.linear,
        },
      })

    const handleVisibility = () => {
      if (document.visibilityState === "hidden") controls.stop()
      else void play()
    }

    void play()
    document.addEventListener("visibilitychange", handleVisibility)
    return () => {
      controls.stop()
      document.removeEventListener("visibilitychange", handleVisibility)
    }
  }, [controls])

  return (
    <div className={`relative overflow-hidden bg-gray-200 rounded ${className}`}>
      <motion.div
        className="absolute inset-0 bg-gradient-to-r from-transparent via-white/60 to-transparent"
        initial={{ x: "-100%" }}
        animate={controls}
      />
    </div>
  )
}
```

### ボタンのローディング状態

```tsx
"use client"
import { motion, AnimatePresence } from "motion/react"
import { motionTokens, springs } from "@/lib/motion-tokens"

export function LoadingButton({
  loading,
  label,
  onClick,
}: {
  loading: boolean
  label: string
  onClick: () => void
}) {
  return (
    <motion.button
      onClick={onClick}
      animate={{ opacity: loading ? 0.7 : 1 }}
      whileTap={loading ? {} : { scale: motionTokens.scale.press }}
      transition={springs.snappy}
      disabled={loading}
    >
      <AnimatePresence mode="wait">
        {loading ? (
          <motion.span
            key="loading"
            initial={{ opacity: 0 }} animate={{ opacity: 1 }} exit={{ opacity: 0 }}
            transition={{ duration: motionTokens.duration.fast }}
          >
            …
          </motion.span>
        ) : (
          <motion.span
            key="label"
            initial={{ opacity: 0 }} animate={{ opacity: 1 }} exit={{ opacity: 0 }}
            transition={{ duration: motionTokens.duration.fast }}
          >
            {label}
          </motion.span>
        )}
      </AnimatePresence>
    </motion.button>
  )
}
```

### 表示非表示時に一時停止する無限アニメーション

```tsx
"use client"
import { useEffect } from "react"
import { motion, useAnimation } from "motion/react"
import { motionTokens } from "@/lib/motion-tokens"

export function PulseDot() {
  const controls = useAnimation()

  useEffect(() => {
    const pulse = () =>
      controls.start({
        scale: [1, 1.4, 1],
        opacity: [1, 0.6, 1],
        transition: { repeat: Infinity, duration: motionTokens.duration.crawl },
      })

    // ルール2: タブが非表示のときに一時停止
    const handleVisibility = () => {
      if (document.visibilityState === "hidden") controls.stop()
      else void pulse()
    }

    void pulse()
    document.addEventListener("visibilitychange", handleVisibility)
    // ルール7: アンマウント時にcontrolsを停止しリスナーを削除する
    return () => {
      controls.stop()
      document.removeEventListener("visibilitychange", handleVisibility)
    }
  }, [controls])

  return <motion.span className="w-2 h-2 rounded-full bg-green-400" animate={controls} />
}
```

## エンドツーエンド例

`useMotionValue`、`useTransform`、`useSafeMotion`、`AnimatePresence`、および`motion-foundations`のトークンを組み合わせた、シマーコンテンツ、ローディング状態、モーション削減対応を備えたドラッグで消えるシート:

```tsx
"use client"
import { useState } from "react"
import { motion, AnimatePresence, useMotionValue, useTransform } from "motion/react"
import { springs, motionTokens } from "@/lib/motion-tokens"
import { useSafeMotion } from "@/hooks/use-reduced-motion"
import { ShimmerSkeleton } from "./shimmer-skeleton"

export function DismissibleSheet({
  isOpen,
  onClose,
  loading,
  children,
}: {
  isOpen: boolean
  onClose: () => void
  loading: boolean
  children: React.ReactNode
}) {
  const safe = useSafeMotion(motionTokens.distance.xl)
  const y = useMotionValue(0)
  const opacity = useTransform(y, [0, 200], [1, 0])

  return (
    <AnimatePresence>
      {isOpen && (
        <>
          {/* バックドロップ */}
          <motion.div
            key="backdrop"
            className="fixed inset-0 bg-black/40"
            initial={{ opacity: 0 }}
            animate={{ opacity: 1 }}
            exit={{ opacity: 0 }}
            onClick={onClose}
          />

          {/* シート — ドラッグで消える */}
          <motion.div
            key="sheet"
            className="fixed bottom-0 inset-x-0 rounded-t-2xl bg-white p-6"
            drag="y"
            dragConstraints={{ top: 0 }}
            style={{ y, opacity }}
            onDragEnd={(_, info) => {
              if (info.offset.y > 120 || info.velocity.y > 500) onClose()
            }}
            initial={safe.initial}
            animate={safe.animate}
            exit={safe.exit}
            transition={springs.gentle}
          >
            {loading ? (
              <div className="space-y-3">
                <ShimmerSkeleton className="h-4 w-3/4" />
                <ShimmerSkeleton className="h-4 w-1/2" />
                <ShimmerSkeleton className="h-20 w-full" />
              </div>
            ) : children}
          </motion.div>
        </>
      )}
    </AnimatePresence>
  )
}
```

## 制約 / 対象外

このスキルは以下を**カバーしない**:

- トークンとスプリング定義 → `motion-foundations`を参照
- 標準的なUIパターン（ボタン、モーダル、スタガー、ページトランジション）→ `motion-patterns`を参照
- CSSのみのアニメーションや`motion/react`なしのTailwindの`animate-*`
- キャンバスまたはWebGLベースのアニメーション（Three.js、Pixiなど）
- 外部状態マネージャーを使ったフルドラッグ＆ドロップシステム（dnd-kit、react-beautiful-dnd）
- ゲームループやフレームバイフレームアニメーション

## アンチパターン

| アンチパターン | 違反ルール | 修正方法 |
| ---------------------------------------------- | ------- | ------------------------------------------------ |
| `drag`をデスクトップのみでテスト | ルール1 | タッチエミュレーターと実機でテストする |
| `animate={{ repeat: Infinity }}`で一時停止なし | ルール2 | `visibilitychange`リスナーを追加する |
| `onDragEnd`でオフセットのみ確認し速度を確認しない | ルール3 | `info.offset`と`info.velocity`の両方を確認する |
| `useEffect`の前に`animate(scope, ...)`を呼び出す | ルール4 | マウント後にのみ`animate()`を呼び出す |
| レンダー内で`const x = new MotionValue(0)` | ルール5 | `const x = useMotionValue(0)`を使う |
| インラインで`transition={{ duration: 1.2 }}` | ルール6 | `motionTokens.duration.crawl`を使う |
| クリーンアップなしの`useEffect` | ルール7 | `removeEventListener` / `controls.stop`を返す |
| 異なるコマンドを持つパス間でSVGモーフ | ルール8 | アニメーション前にパスコマンドを正規化する |

## 関連スキル

- **`motion-foundations`** — このスキルでインポートされる全トークン、スプリング、`useSafeMotion`、SSRガードを定義する。このスキルを使う前に必ずセットアップすること。
- **`motion-patterns`** — 標準的なUIパターン（ボタン、モーダル、スタガー、ページトランジション、スクロールリビール）を担当する。高度なパターンを使う前にまずこちらを使うこと。
