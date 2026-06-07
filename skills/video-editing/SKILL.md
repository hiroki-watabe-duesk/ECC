---
name: video-editing
description: 実映像のカット・構成・補強を行うAI支援動画編集ワークフロー。生素材の取り込みからFFmpeg、Remotion、ElevenLabs、fal.aiを経てDescriptまたはCapCutでの最終仕上げまで、パイプライン全体をカバー。動画の編集・カット・Vlog作成・動画コンテンツ制作を行いたいときに使用。
origin: ECC
---

# 動画編集

実映像をAI支援で編集する。プロンプトからの生成ではない。既存動画を素早く編集するためのワークフロー。

## アクティベートするタイミング

- 映像のカット・構成を行いたい
- 長尺録画を短尺コンテンツに変換したい
- 生素材からVlog・チュートリアル・デモ動画を作りたい
- 既存動画にオーバーレイ・字幕・音楽・ナレーションを追加したい
- プラットフォーム別（YouTube・TikTok・Instagram）にリフレームしたい
- 「動画を編集して」「この映像をカットして」「Vlogを作って」「動画ワークフロー」と言われたとき

## 基本的な考え方

AI動画編集が真価を発揮するのは、動画全体を生成させようとするのをやめ、実映像の圧縮・構成・補強に使い始めたときだ。価値は生成にあるのではない。価値は圧縮にある。

## パイプライン

```
Screen Studio / 生素材
  → Claude / Codex
  → FFmpeg
  → Remotion
  → ElevenLabs / fal.ai
  → Descript または CapCut
```

各レイヤーには固有の役割がある。レイヤーをスキップしない。1つのツールにすべてをやらせようとしない。

## レイヤー1: キャプチャ（Screen Studio / 生素材）

ソース素材を収集する:
- **Screen Studio**: アプリデモ・コーディングセッション・ブラウザ操作の洗練されたスクリーン録画
- **生カメラ映像**: Vlog映像・インタビュー・イベント録画
- **VideoDB経由のデスクトップキャプチャ**: リアルタイムコンテキスト付きのセッション録画（`videodb`スキル参照）

出力: 整理済みの生ファイル。

## レイヤー2: 整理（Claude / Codex）

Claude CodeまたはCodexを使って:
- **文字起こしとラベリング**: トランスクリプト生成、トピックとテーマの特定
- **構成のプランニング**: 残すもの・カットするもの・順序の決定
- **無駄なセクションの特定**: 間・脱線・繰り返しテイクの発見
- **編集決定リストの生成**: カットのタイムスタンプ・残すセグメント
- **FFmpegとRemotionコードの雛形生成**: コマンドとコンポジションの生成

```
プロンプト例:
「4時間録画のトランスクリプトです。24分のVlog用に最も良い8セグメントを特定し、
各セグメントのFFmpegカットコマンドを教えてください。」
```

このレイヤーは構成に関するもので、最終的なクリエイティブの判断ではない。

## レイヤー3: 確定的なカット（FFmpeg）

FFmpegは退屈だが重要な作業を担う: 分割・トリミング・結合・前処理。

### タイムスタンプによるセグメント抽出

```bash
ffmpeg -i raw.mp4 -ss 00:12:30 -to 00:15:45 -c copy segment_01.mp4
```

### 編集決定リストからの一括カット

```bash
#!/bin/bash
# cuts.txt: start,end,label
while IFS=, read -r start end label; do
  ffmpeg -i raw.mp4 -ss "$start" -to "$end" -c copy "segments/${label}.mp4"
done < cuts.txt
```

### セグメントの結合

```bash
# ファイルリストの作成
for f in segments/*.mp4; do echo "file '$f'"; done > concat.txt
ffmpeg -f concat -safe 0 -i concat.txt -c copy assembled.mp4
```

### 編集高速化用プロキシの作成

```bash
ffmpeg -i raw.mp4 -vf "scale=960:-2" -c:v libx264 -preset ultrafast -crf 28 proxy.mp4
```

### 文字起こし用音声の抽出

```bash
ffmpeg -i raw.mp4 -vn -acodec pcm_s16le -ar 16000 audio.wav
```

### 音声レベルの正規化

```bash
ffmpeg -i segment.mp4 -af loudnorm=I=-16:TP=-1.5:LRA=11 -c:v copy normalized.mp4
```

## レイヤー4: プログラマブルなコンポジション（Remotion）

Remotionは編集問題をコンポーザブルなコードに変換する。従来のエディタでは困難な作業に使う:

### Remotionを使うタイミング

- オーバーレイ: テキスト・画像・ブランディング・ロワーサード
- データビジュアライゼーション: チャート・統計・アニメーション数値
- モーショングラフィクス: トランジション・説明アニメーション
- コンポーザブルなシーン: 動画間で再利用できるテンプレート
- プロデモ: アノテーション付きスクリーンショット・UIハイライト

### 基本的なRemotionコンポジション

```tsx
import { AbsoluteFill, Sequence, Video, useCurrentFrame } from "remotion";

export const VlogComposition: React.FC = () => {
  const frame = useCurrentFrame();

  return (
    <AbsoluteFill>
      {/* メイン映像 */}
      <Sequence from={0} durationInFrames={300}>
        <Video src="/segments/intro.mp4" />
      </Sequence>

      {/* タイトルオーバーレイ */}
      <Sequence from={30} durationInFrames={90}>
        <AbsoluteFill style={{
          justifyContent: "center",
          alignItems: "center",
        }}>
          <h1 style={{
            fontSize: 72,
            color: "white",
            textShadow: "2px 2px 8px rgba(0,0,0,0.8)",
          }}>
            The AI Editing Stack
          </h1>
        </AbsoluteFill>
      </Sequence>

      {/* 次のセグメント */}
      <Sequence from={300} durationInFrames={450}>
        <Video src="/segments/demo.mp4" />
      </Sequence>
    </AbsoluteFill>
  );
};
```

### 出力のレンダリング

```bash
npx remotion render src/index.ts VlogComposition output.mp4
```

詳細なパターンとAPIリファレンスは[Remotionのドキュメント](https://www.remotion.dev/docs)を参照。

## レイヤー5: 生成アセット（ElevenLabs / fal.ai）

必要なものだけを生成する。動画全体を生成しない。

### ElevenLabsでのナレーション

```python
import os
import requests

resp = requests.post(
    f"https://api.elevenlabs.io/v1/text-to-speech/{voice_id}",
    headers={
        "xi-api-key": os.environ["ELEVENLABS_API_KEY"],
        "Content-Type": "application/json"
    },
    json={
        "text": "Your narration text here",
        "model_id": "eleven_turbo_v2_5",
        "voice_settings": {"stability": 0.5, "similarity_boost": 0.75}
    }
)
with open("voiceover.mp3", "wb") as f:
    f.write(resp.content)
```

### fal.aiを使った音楽とSFX

`fal-ai-media`スキルを使う用途:
- バックグラウンドミュージックの生成
- 効果音（動画対応のThinkSoundモデル）
- トランジションサウンド

### fal.aiを使ったビジュアル生成

存在しない挿入カット・サムネイル・Bロール用:
```
generate(app_id: "fal-ai/nano-banana-pro", input_data: {
  "prompt": "professional thumbnail for tech vlog, dark background, code on screen",
  "image_size": "landscape_16_9"
})
```

### VideoDBでの生成音声

VideoDBが設定済みの場合:
```python
voiceover = coll.generate_voice(text="Narration here", voice="alloy")
music = coll.generate_music(prompt="lo-fi background for coding vlog", duration=120)
sfx = coll.generate_sound_effect(prompt="subtle whoosh transition")
```

## レイヤー6: 最終仕上げ（Descript / CapCut）

最後のレイヤーは人間の作業。従来のエディタを使う場面:
- **ペーシング**: 速すぎる・遅すぎるカットの調整
- **字幕**: 自動生成後に手動でクリーニング
- **カラーグレーディング**: 基本的な補正とムード
- **最終的な音声ミックス**: ボイス・音楽・SFXのバランス調整
- **エクスポート**: プラットフォーム別のフォーマットと画質設定

ここにセンスが宿る。AIが繰り返し作業を排除し、最終判断はあなたが下す。

## SNS向けリフレーム

プラットフォームによってアスペクト比が異なる:

| プラットフォーム | アスペクト比 | 解像度 |
|----------|-------------|------------|
| YouTube | 16:9 | 1920x1080 |
| TikTok / Reels | 9:16 | 1080x1920 |
| Instagram フィード | 1:1 | 1080x1080 |
| X / Twitter | 16:9 または 1:1 | 1280x720 または 720x720 |

### FFmpegでのリフレーム

```bash
# 16:9 → 9:16（センタークロップ）
ffmpeg -i input.mp4 -vf "crop=ih*9/16:ih,scale=1080:1920" vertical.mp4

# 16:9 → 1:1（センタークロップ）
ffmpeg -i input.mp4 -vf "crop=ih:ih,scale=1080:1080" square.mp4
```

### VideoDBでのリフレーム

```python
from videodb import ReframeMode

# スマートリフレーム（AI誘導による被写体トラッキング）
reframed = video.reframe(start=0, end=60, target="vertical", mode=ReframeMode.smart)
```

## シーン検出と自動カット

### FFmpegによるシーン検出

```bash
# シーン変化の検出（閾値0.3 = 中程度の感度）
ffmpeg -i input.mp4 -vf "select='gt(scene,0.3)',showinfo" -vsync vfr -f null - 2>&1 | grep showinfo
```

### 自動カット用の無音検出

```bash
# 無音セグメントの検出（無駄な間のカットに有効）
ffmpeg -i input.mp4 -af silencedetect=noise=-30dB:d=2 -f null - 2>&1 | grep silence
```

### ハイライト抽出

Claudeでトランスクリプトとシーンタイムスタンプを分析する:
```
「タイムスタンプ付きのトランスクリプトとシーン変化ポイントを元に、
SNS向けの最も魅力的な30秒クリップを5つ特定してください。」
```

## 各ツールの得意領域

| ツール | 強み | 弱み |
|------|----------|----------|
| Claude / Codex | 整理・プランニング・コード生成 | クリエイティブな判断層ではない |
| FFmpeg | 確定的カット・バッチ処理・フォーマット変換 | ビジュアル編集UIなし |
| Remotion | プログラマブルなオーバーレイ・コンポーザブルなシーン・再利用テンプレート | 非開発者には学習コストがある |
| Screen Studio | 即座に洗練されたスクリーン録画 | スクリーンキャプチャのみ |
| ElevenLabs | ボイス・ナレーション・音楽・SFX | ワークフローの中心ではない |
| Descript / CapCut | 最終ペーシング・字幕・仕上げ | 手動作業、自動化不可 |

## 基本原則

1. **生成ではなく編集。** このワークフローは実映像のカット用であり、プロンプトからの生成ではない。
2. **スタイルより先に構成。** レイヤー2でストーリーを正しく組み立ててから、ビジュアルに手を付ける。
3. **FFmpegが骨格。** 地味だが重要。長尺映像を扱いやすくする場所。
4. **繰り返す作業にはRemotionを。** 複数回行うならRemotionコンポーネントにする。
5. **生成は選択的に。** 存在しないアセットにのみAI生成を使う。すべてに使わない。
6. **センスは最後のレイヤー。** AIが繰り返し作業を排除する。最終クリエイティブな判断はあなたが下す。

## 関連スキル

- `fal-ai-media` — AI画像・動画・音声の生成
- `videodb` — サーバーサイドの動画処理・インデックス・ストリーミング
- `content-engine` — プラットフォームネイティブのコンテンツ配信
