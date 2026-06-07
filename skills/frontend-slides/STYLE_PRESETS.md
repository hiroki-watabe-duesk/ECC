# スタイルプリセットリファレンス

`frontend-slides` のキュレーション済みビジュアルスタイル集。

このファイルの用途:
- 必須のビューポートフィット CSS ベース
- プリセットの選択とムードのマッピング
- CSS の注意点とバリデーションルール

抽象的な図形のみ使用する。ユーザーが明示的に求めない限りイラストは避ける。

## ビューポートフィットは絶対条件

すべてのスライドは 1 つのビューポートに完全に収まらなければならない。

### 黄金ルール

```text
Each slide = exactly one viewport height.
Too much content = split into more slides.
Never scroll inside a slide.
```

### 密度の上限

| スライドの種類 | コンテンツの最大量 |
|------------|-----------------|
| タイトルスライド | 見出し 1 つ + サブタイトル 1 つ + 任意のタグライン |
| コンテンツスライド | 見出し 1 つ + 箇条書き 4〜6 つ または 段落 2 つ |
| フィーチャーグリッド | カード 6 枚まで |
| コードスライド | 最大 8〜10 行 |
| 引用スライド | 引用 1 つ + 出典 |
| 画像スライド | 画像 1 枚、理想的には 60vh 未満 |

## 必須ベース CSS

このブロックをすべての生成プレゼンテーションにコピーし、その上にテーマを適用する。

```css
/* ===========================================
   VIEWPORT FITTING: MANDATORY BASE STYLES
   =========================================== */

html, body {
    height: 100%;
    overflow-x: hidden;
}

html {
    scroll-snap-type: y mandatory;
    scroll-behavior: smooth;
}

.slide {
    width: 100vw;
    height: 100vh;
    height: 100dvh;
    overflow: hidden;
    scroll-snap-align: start;
    display: flex;
    flex-direction: column;
    position: relative;
}

.slide-content {
    flex: 1;
    display: flex;
    flex-direction: column;
    justify-content: center;
    max-height: 100%;
    overflow: hidden;
    padding: var(--slide-padding);
}

:root {
    --title-size: clamp(1.5rem, 5vw, 4rem);
    --h2-size: clamp(1.25rem, 3.5vw, 2.5rem);
    --h3-size: clamp(1rem, 2.5vw, 1.75rem);
    --body-size: clamp(0.75rem, 1.5vw, 1.125rem);
    --small-size: clamp(0.65rem, 1vw, 0.875rem);

    --slide-padding: clamp(1rem, 4vw, 4rem);
    --content-gap: clamp(0.5rem, 2vw, 2rem);
    --element-gap: clamp(0.25rem, 1vw, 1rem);
}

.card, .container, .content-box {
    max-width: min(90vw, 1000px);
    max-height: min(80vh, 700px);
}

.feature-list, .bullet-list {
    gap: clamp(0.4rem, 1vh, 1rem);
}

.feature-list li, .bullet-list li {
    font-size: var(--body-size);
    line-height: 1.4;
}

.grid {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(min(100%, 250px), 1fr));
    gap: clamp(0.5rem, 1.5vw, 1rem);
}

img, .image-container {
    max-width: 100%;
    max-height: min(50vh, 400px);
    object-fit: contain;
}

@media (max-height: 700px) {
    :root {
        --slide-padding: clamp(0.75rem, 3vw, 2rem);
        --content-gap: clamp(0.4rem, 1.5vw, 1rem);
        --title-size: clamp(1.25rem, 4.5vw, 2.5rem);
        --h2-size: clamp(1rem, 3vw, 1.75rem);
    }
}

@media (max-height: 600px) {
    :root {
        --slide-padding: clamp(0.5rem, 2.5vw, 1.5rem);
        --content-gap: clamp(0.3rem, 1vw, 0.75rem);
        --title-size: clamp(1.1rem, 4vw, 2rem);
        --body-size: clamp(0.7rem, 1.2vw, 0.95rem);
    }

    .nav-dots, .keyboard-hint, .decorative {
        display: none;
    }
}

@media (max-height: 500px) {
    :root {
        --slide-padding: clamp(0.4rem, 2vw, 1rem);
        --title-size: clamp(1rem, 3.5vw, 1.5rem);
        --h2-size: clamp(0.9rem, 2.5vw, 1.25rem);
        --body-size: clamp(0.65rem, 1vw, 0.85rem);
    }
}

@media (max-width: 600px) {
    :root {
        --title-size: clamp(1.25rem, 7vw, 2.5rem);
    }

    .grid {
        grid-template-columns: 1fr;
    }
}

@media (prefers-reduced-motion: reduce) {
    *, *::before, *::after {
        animation-duration: 0.01ms !important;
        transition-duration: 0.2s !important;
    }

    html {
        scroll-behavior: auto;
    }
}
```

## ビューポートチェックリスト

- すべての `.slide` に `height: 100vh`、`height: 100dvh`、`overflow: hidden` がある
- すべてのタイポグラフィが `clamp()` を使用している
- すべての間隔が `clamp()` またはビューポート単位を使用している
- 画像に `max-height` の制約がある
- グリッドが `auto-fit` + `minmax()` で適応している
- `700px`、`600px`、`500px` の高さに対するブレークポイントが存在する
- 窮屈に感じる場合はスライドを分割する

## ムードとプリセットのマッピング

| ムード | 適したプリセット |
|------|--------------|
| 感動的・自信に満ちた | Bold Signal、Electric Studio、Dark Botanical |
| 興奮・エネルギッシュ | Creative Voltage、Neon Cyber、Split Pastel |
| 落ち着いた・集中した | Notebook Tabs、Paper & Ink、Swiss Modern |
| インスパイアされた・心動かされた | Dark Botanical、Vintage Editorial、Pastel Geometry |

## プリセットカタログ

### 1. Bold Signal

- 雰囲気: 自信に満ち、インパクト大、基調講演向け
- 最適な用途: ピッチデッキ、ローンチ、ステートメント
- フォント: Archivo Black + Space Grotesk
- パレット: チャコールベース、ホットオレンジのフォーカルカード、クリスプな白いテキスト
- 特徴: 特大のセクション番号、ダークフィールドのハイコントラストカード

### 2. Electric Studio

- 雰囲気: クリーン、大胆、エージェンシー仕上げ
- 最適な用途: クライアントプレゼンテーション、戦略レビュー
- フォント: Manrope のみ
- パレット: ブラック、ホワイト、彩度の高いコバルトアクセント
- 特徴: 二分割パネルとシャープなエディトリアルアライメント

### 3. Creative Voltage

- 雰囲気: エネルギッシュ、レトロモダン、遊び心のある自信
- 最適な用途: クリエイティブスタジオ、ブランドワーク、プロダクトストーリーテリング
- フォント: Syne + Space Mono
- パレット: エレクトリックブルー、ネオンイエロー、深いネイビー
- 特徴: ハーフトーンテクスチャ、バッジ、パンチの効いたコントラスト

### 4. Dark Botanical

- 雰囲気: エレガント、プレミアム、大気感
- 最適な用途: ラグジュアリーブランド、思慮深いナラティブ、プレミアムプロダクトデッキ
- フォント: Cormorant + IBM Plex Sans
- パレット: ほぼブラック、温かみのあるアイボリー、ブラッシュ、ゴールド、テラコッタ
- 特徴: ぼかした抽象的な円、細いルール、抑制されたモーション

### 5. Notebook Tabs

- 雰囲気: エディトリアル、整理された、触覚的
- 最適な用途: レポート、レビュー、構造化されたストーリーテリング
- フォント: Bodoni Moda + DM Sans
- パレット: チャコール上のクリーム紙にパステルタブ
- 特徴: ペーパーシート、カラーサイドタブ、バインダーの詳細

### 6. Pastel Geometry

- 雰囲気: 親しみやすい、モダン、フレンドリー
- 最適な用途: プロダクト概要、オンボーディング、ライトなブランドデッキ
- フォント: Plus Jakarta Sans のみ
- パレット: 薄いブルーフィールド、クリームカード、ソフトなピンク/ミント/ラベンダーアクセント
- 特徴: 縦ピル、丸みのあるカード、ソフトシャドウ

### 7. Split Pastel

- 雰囲気: 遊び心、モダン、クリエイティブ
- 最適な用途: エージェンシーイントロ、ワークショップ、ポートフォリオ
- フォント: Outfit のみ
- パレット: ピーチ + ラベンダーの分割にミントのバッジ
- 特徴: 分割バックドロップ、丸みのあるタグ、ライトグリッドオーバーレイ

### 8. Vintage Editorial

- 雰囲気: ウィットに富んだ、個性的、マガジン風
- 最適な用途: パーソナルブランド、独自の視点を持つトーク、ストーリーテリング
- フォント: Fraunces + Work Sans
- パレット: クリーム、チャコール、くすんだ温かみのあるアクセント
- 特徴: ジオメトリックアクセント、ボーダー付きコールアウト、パンチの効いたセリフ見出し

### 9. Neon Cyber

- 雰囲気: 未来的、テッキー、キネティック
- 最適な用途: AI、インフラ、開発ツール、未来像のトーク
- フォント: Clash Display + Satoshi
- パレット: ミッドナイトネイビー、シアン、マゼンタ
- 特徴: グロー、パーティクル、グリッド、データレーダーエネルギー

### 10. Terminal Green

- 雰囲気: 開発者向け、ハッカークリーン
- 最適な用途: API、CLI ツール、エンジニアリングデモ
- フォント: JetBrains Mono のみ
- パレット: GitHub ダーク + ターミナルグリーン
- 特徴: スキャンライン、コマンドラインフレーミング、精密なモノスペースリズム

### 11. Swiss Modern

- 雰囲気: ミニマル、精密、データ重視
- 最適な用途: コーポレート、プロダクト戦略、アナリティクス
- フォント: Archivo + Nunito
- パレット: ホワイト、ブラック、シグナルレッド
- 特徴: 視認できるグリッド、非対称性、ジオメトリックな規律

### 12. Paper & Ink

- 雰囲気: 文学的、思慮深い、ストーリー主導
- 最適な用途: エッセイ、基調講演のナラティブ、マニフェストデッキ
- フォント: Cormorant Garamond + Source Serif 4
- パレット: 温かみのあるクリーム、チャコール、クリムゾンアクセント
- 特徴: プルクォート、ドロップキャップ、エレガントなルール

## 直接選択プロンプト

ユーザーが望むスタイルをすでに把握している場合、プレビュー生成を強制せず、上記のプリセット名から直接選択させる。

## アニメーションの感覚マッピング

| 感覚 | モーションの方向 |
|---------|------------------|
| ドラマティック・シネマティック | スローフェード、パラックス、大きなスケールイン |
| テッキー・未来的 | グロー、パーティクル、グリッドモーション、スクランブルテキスト |
| 遊び心・フレンドリー | バネのようなイージング、丸みのある形、浮遊するモーション |
| プロフェッショナル・コーポレート | 控えめな 200〜300ms のトランジション、クリーンなスライド |
| 落ち着いた・ミニマル | 非常に抑制されたモーション、ホワイトスペース優先 |
| エディトリアル・マガジン | 強い階層性、テキストと画像の段階的なインタープレイ |

## CSS の注意点: 関数の否定

次のようには書かない:

```css
right: -clamp(28px, 3.5vw, 44px);
margin-left: -min(10vw, 100px);
```

ブラウザはこれらを無視する。

代わりに常に次のように書く:

```css
right: calc(-1 * clamp(28px, 3.5vw, 44px));
margin-left: calc(-1 * min(10vw, 100px));
```

## バリデーションサイズ

最低限以下でテストする:
- デスクトップ: `1920x1080`、`1440x900`、`1280x720`
- タブレット: `1024x768`、`768x1024`
- モバイル: `375x667`、`414x896`
- ランドスケープスマートフォン: `667x375`、`896x414`

## アンチパターン

使用しないもの:
- パープルオンホワイトのスタートアップテンプレート
- ユーザーがユーティリティの中立性を明示的に求めない限り Inter / Roboto / Arial をビジュアルボイスとして使用しない
- 箇条書きの壁、小さな文字、スクロールが必要なコードブロック
- 抽象的なジオメトリで代替できる場面での装飾的なイラスト
