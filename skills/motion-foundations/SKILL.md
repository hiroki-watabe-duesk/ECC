---
name: motion-foundations
description: React / Next.js（motion/react使用）向けのモーショントークン・スプリングプリセット・パフォーマンスルール・デバイス適応・アクセシビリティ強制・SSR安全性の基盤レイヤー。他のすべてのモーションスキルがこのスキルに依存する。
version: 1.0
tags: [motion, animation, performance, accessibility]
category: frontend
author: jeff
---

# モーション基盤

モーションシステムの基底レイヤー。下流のスキル（`motion-patterns`・`motion-advanced`）が継承するすべての値・制約・ルールを定義する。
アニメーション作業を始める前に必ずこのスキルを読み込むこと。

## アクティベートするタイミング

- アニメーションコンポーネントをゼロから作成するとき
- トークン・スプリングプリセット・イージング値を設定するとき
- `prefers-reduced-motion`のサポートを実装するとき
- アニメーション初期状態によるハイドレーションミスマッチをデバッグするとき
- アニメーションを存在させるべきかどうかを評価するとき

## 出力

このスキルが生成するもの:

- 共有`motionTokens`オブジェクト（duration・easing・distance・scale）
- 共有`springs`プリセットマップ（5つの名前付き設定）
- すべてのコンポーネントが使用する`shouldAnimate()`ゲート
- `useReducedMotion`によるアクセシビリティ準拠のアニメーションデフォルト
- ハイドレーション警告ゼロのSSR安全な初期状態

## 原則

モーションは以下の少なくとも1つを満たさなければ削除する:

- 注意を誘導する
- 状態を伝える
- 空間的な連続性を維持する

応答性は常にスムーズさより優先される。60fpsのアニメーションでも入力遅延を引き起こすなら、アニメーションなしの方がよい。

## ルール

これらは非交渉的。システム内のすべてのコンポーネントに適用される。

1. **`motion/react`のみを使用する。** `framer-motion`からのインポートは禁止。同一ツリー内での混在も禁止。
2. **`initial`はサーバー出力と一致させる。** サーバーが`opacity: 1`をレンダリングするなら、`initial`プロップも`opacity: 1`でなければならない。例外なし。
3. **モーション低減は常に最優先。** `useReducedMotion()`が`true`を返すか`prefersReduced`が`true`の場合、すべてのトランスフォームを無効にする。0.2s以下のopacityのみのフェードだけが許可されるフォールバック。
4. **レイアウトプロパティはアニメーション禁止。** `width`・`height`・`top`・`left`・`margin`・`padding`を`animate`で使うことを禁止する。`transform`と`opacity`のみを使用すること。
5. **すべてのトークン値は`motionTokens`から取得する。** コンポーネントファイル内にdurationやeasingをハードコードすることを禁止する。
6. **すべてのスプリング設定は`springs`マップから取得する。** `stiffness`/`damping`のインライン値は禁止。
7. **`"use client"`は必須。** `motion/react`からインポートするすべてのファイルに必要。
8. **モジュールレベルで`window`や`navigator`を読まない。** 必ず`typeof window !== "undefined"`でガードする。

## 意思決定ガイダンス

### durationの選択

| トークン | 使用場面 |
| --------- | -------------------------------------------- |
| `instant` | ツールチップの表示/非表示・フォーカスリング・バッジ更新 |
| `fast` | ボタンフィードバック・アイコン切り替え・チップトグル |
| `normal` | モーダルを開く・カード展開・ページ要素の入場 |
| `slow` | ヒーロー入場・フルページトランジション |
| `crawl` | 意図的なストーリーテリング。多用しないこと |

### スプリングの選択

| プリセット | 使用場面 |
| --------- | ------------------------------------------ |
| `snappy` | デフォルトUI――ボタン・チップ・ナビゲーション項目 |
| `gentle` | 柔らかく着地するカード・モーダル・パネル |
| `bouncy` | 遊び心のある場面――空の状態・オンボーディング |
| `instant` | ツールチップ・ポップオーバー・ドロップダウン |
| `release` | ドラッグリリース――自然な物理的感触 |

### アニメーションを完全に無効にするタイミング

次の場合に`shouldAnimate()`が`false`を返すよう無効化する:

- `prefersReduced`が`true`
- `isLowEnd`が`true`かつアニメーションが必須でない
- 要素がビューポート外にあり、ビューポートに入ることがない
- アニメーションが純粋に装飾的でUX上の目的がない

## コアコンセプト

### トークンシステム

```ts
// lib/motion-tokens.ts
export const motionTokens = {
  duration: {
    instant: 0.08,
    fast:    0.18,
    normal:  0.35,
    slow:    0.6,
    crawl:   1.0,
  },
  easing: {
    smooth: [0.22, 1, 0.36, 1],
    sharp:  [0.4, 0, 0.2, 1],
    bounce: [0.34, 1.56, 0.64, 1],
    linear: [0, 0, 1, 1],
  },
  distance: {
    xs: 4,
    sm: 8,
    md: 16,
    lg: 24,
    xl: 48,
  },
  scale: {
    subtle: 0.98,
    press:  0.95,
    pop:    1.04,
  },
}

export const springs = {
  snappy:  { type: "spring", stiffness: 300, damping: 30 },
  gentle:  { type: "spring", stiffness: 120, damping: 14 },
  bouncy:  { type: "spring", stiffness: 400, damping: 10 },
  instant: { type: "spring", stiffness: 600, damping: 35 },
  release: { type: "spring", stiffness: 200, damping: 20, restDelta: 0.001 },
}
```

### ランタイムフラグ

```ts
// lib/motion-config.ts
export const motionConfig = {
  isLowEnd() {
    return (
      typeof navigator !== "undefined" &&
      navigator.hardwareConcurrency <= 4
    )
  },

  prefersReduced() {
    return (
      typeof window !== "undefined" &&
      window.matchMedia("(prefers-reduced-motion: reduce)").matches
    )
  },

  shouldAnimate({ essential = false } = {}) {
    if (this.prefersReduced()) return false
    if (!essential && this.isLowEnd()) return false
    return true
  },

  duration() {
    return this.isLowEnd() || this.prefersReduced()
      ? motionTokens.duration.instant
      : motionTokens.duration.normal
  },
}
```

### アクセシビリティ

**優先順位（高い順）:**

1. `prefers-reduced-motion: reduce` — すべてのトランスフォームを無効化し、opacityのトランジションを0.2s以下に制限する
2. ローエンドデバイス検出 — durationを短縮し、必須でないアニメーションを削除する
3. デザインの好み――それ以外のすべて

モーションはグレースフルにデグレードしなければならない。レイアウトシフトを引き起こしたり向きの感覚を失わせるような突然の消滅は許されない。

```tsx
// hooks/use-reduced-motion.tsx
"use client"
import { useReducedMotion } from "motion/react"

export function useSafeMotion(fullY: number = 16) {
  const reduce = useReducedMotion()
  return {
    initial: { opacity: 0, y: reduce ? 0 : fullY },
    animate: { opacity: 1, y: 0 },
    exit:    { opacity: 0, y: reduce ? 0 : -fullY },
  }
}
```

```css
/* globals.css */
@media (prefers-reduced-motion: reduce) {
  .motion-safe-transition  { transition: opacity 0.15s; }
  .motion-reduce-transform { transform: none !important; }
}
```

```html
<!-- Tailwind -->
<div class="motion-safe:animate-fade motion-reduce:opacity-100"></div>
```

### SSR / ハイドレーションの安全性

**ルール: `initial`は常にサーバーがレンダリングするものと一致させること。**

```tsx
// 誤り――サーバーはopacity:1をレンダリングするが、initialは0と指定 → ハイドレーションミスマッチ
<motion.div initial={{ opacity: 0 }} animate={{ opacity: 1 }} />

// 正しい――AnimatePressenceを使うか、クライアントマウントまで遅延させる
"use client"
const [mounted, setMounted] = useState(false)
useEffect(() => setMounted(true), [])

<motion.div
  initial={{ opacity: mounted ? 0 : 1 }}
  animate={{ opacity: 1 }}
/>
```

## コード例

### エンドツーエンド: トークン + スプリング + アクセシビリティ + SSRガード

```tsx
// components/fade-in-card.tsx
"use client"

import { useState, useEffect } from "react"
import { motion } from "motion/react"
import { motionTokens, springs } from "@/lib/motion-tokens"
import { useSafeMotion } from "@/hooks/use-reduced-motion"
import { motionConfig } from "@/lib/motion-config"

interface FadeInCardProps {
  children: React.ReactNode
  delay?: number
}

export function FadeInCard({ children, delay = 0 }: FadeInCardProps) {
  // SSRガード――initialはサーバー出力（opacity: 1）と一致させる
  const [mounted, setMounted] = useState(false)
  useEffect(() => setMounted(true), [])

  // アクセシビリティ――モーション低減が優先される場合はトランスフォームを無効化
  const safeMotion = useSafeMotion(motionTokens.distance.md)

  // デバイスゲート――ローエンドハードウェアではアニメーションをスキップ
  if (!motionConfig.shouldAnimate() || !mounted) {
    return <div>{children}</div>
  }

  return (
    <motion.div
      initial={safeMotion.initial}
      animate={safeMotion.animate}
      exit={safeMotion.exit}
      transition={{
        ...springs.gentle,
        delay,
      }}
      whileHover={{ scale: motionTokens.scale.pop }}
      whileTap={{ scale: motionTokens.scale.press }}
    >
      {children}
    </motion.div>
  )
}
```

## 制約 / 対象外

このスキルは以下をカバー**しない**:

- UIコンポーネントパターン（ボタン・モーダル・スタガー）→ `motion-patterns`参照
- ドラッグ・ジェスチャー・SVG・テキストアニメーション・カスタムフック → `motion-advanced`参照
- `motion/react`を使わないCSSのみのアニメーションやTailwindの`animate-*`クラス
- サードパーティのアニメーションライブラリ（GSAP・anime.jsなど）
- モーションデザインの判断（何をアニメーション化するか・何を強調するか）――これはコードの制約ではなくデザインの問題

## アンチパターン

| アンチパターン | 違反するルール | 修正方法 |
| --------------------------------------- | ------- | ------------------------------- |
| `import { motion } from "framer-motion"` | ルール1 | `motion/react`を使用する |
| SSRコンポーネントで`initial={{ opacity: 0 }}` | ルール2 | マウントガードを追加する |
| `useReducedMotion`チェックをスキップ | ルール3 | `useSafeMotion`フックを使用する |
| `animate={{ width: "100%" }}` | ルール4 | 代わりに`scaleX`トランスフォームを使用する |
| `transition={{ duration: 0.4 }}`のインライン記述 | ルール5 | `motionTokens.duration.normal`を使用する |
| `{ stiffness: 300, damping: 30 }`のインライン記述 | ルール6 | `springs.snappy`を使用する |
| `"use client"`ディレクティブの欠如 | ルール7 | ファイルの先頭に追加する |
| モジュールレベルでの`navigator.hardwareConcurrency` | ルール8 | `typeof navigator !== "undefined"`でラップする |

## 関連スキル

- **`motion-patterns`** — ここで定義されたトークンとスプリングを使い、ボタン・モーダル・スタガー・ページトランジション・スクロールパターンを構築する。値の再定義はしない。
- **`motion-advanced`** — ここで定義されたトークンとスプリングを使い、ドラッグ・SVG・テキスト・ジェスチャーパターンを実装する。この基盤の上に`useAnimate`シーケンスとカスタムフックを追加する。
