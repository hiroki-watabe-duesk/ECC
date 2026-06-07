# タイムライン編集ガイド

VideoDBは、複数のアセットからビデオを合成し、テキストや画像のオーバーレイを追加し、オーディオトラックをミックスし、クリップをトリミングするための非破壊的なタイムラインエディターを提供します。これらはすべてサーバーサイドで行われ、再エンコードやローカルツールは不要です。トリミング、クリップの結合、動画へのオーディオ/音楽のオーバーレイ、字幕の追加、テキストや画像のレイヤリングに使用します。

## 前提条件

動画、音声、画像はタイムラインアセットとして使用する前に、コレクションに**アップロードされていなければなりません**。キャプションのオーバーレイには、動画が**発話単語でインデックス化**されている必要もあります。

## 核となる概念

### タイムライン

`Timeline` は仮想的な合成レイヤーです。アセットは**インライン**（メイントラック上に順番に）または**オーバーレイ**（特定のタイムスタンプにレイヤー化）として配置されます。元のメディアは何も変更されず、最終ストリームはオンデマンドでコンパイルされます。

```python
from videodb.timeline import Timeline

timeline = Timeline(conn)
```

### アセット

タイムライン上のすべての要素は**アセット**です。VideoDBは5種類のアセットタイプを提供します:

| アセット | インポート | 主な用途 |
|-------|--------|-------------|
| `VideoAsset` | `from videodb.asset import VideoAsset` | 動画クリップ（トリミング、シーケンシング） |
| `AudioAsset` | `from videodb.asset import AudioAsset` | 音楽、効果音、ナレーション |
| `ImageAsset` | `from videodb.asset import ImageAsset` | ロゴ、サムネイル、オーバーレイ |
| `TextAsset` | `from videodb.asset import TextAsset, TextStyle` | タイトル、キャプション、ローワーサード |
| `CaptionAsset` | `from videodb.editor import CaptionAsset` | 自動レンダリング字幕（Editor API） |

## タイムラインの構築

### 動画クリップのインライン追加

インラインアセットはメイン動画トラック上で次々と再生されます。`add_inline` メソッドは `VideoAsset` のみを受け付けます:

```python
from videodb.asset import VideoAsset

video_a = coll.get_video(video_id_a)
video_b = coll.get_video(video_id_b)

timeline = Timeline(conn)
timeline.add_inline(VideoAsset(asset_id=video_a.id))
timeline.add_inline(VideoAsset(asset_id=video_b.id))

stream_url = timeline.generate_stream()
```

### トリム / サブクリップ

`VideoAsset` に `start` と `end` を使用してソース動画の一部を取り出します:

```python
# Take only seconds 10–30 from the source video
clip = VideoAsset(asset_id=video.id, start=10, end=30)
timeline.add_inline(clip)
```

### VideoAssetのパラメーター

| パラメーター | 型 | デフォルト | 説明 |
|-----------|------|---------|-------------|
| `asset_id` | `str` | 必須 | 動画メディアID |
| `start` | `float` | `0` | トリム開始位置（秒） |
| `end` | `float\|None` | `None` | トリム終了位置（`None` = 全体） |

> **警告:** SDKは負のタイムスタンプを検証しません。`start=-5` を渡してもサイレントに受け入れられますが、壊れた出力や予期しない出力が生成されます。`VideoAsset` を作成する前に必ず `start >= 0`、`start < end`、`end <= video.length` を確認してください。

## テキストオーバーレイ

タイムラインの任意の位置にタイトル、ローワーサード、キャプションを追加します:

```python
from videodb.asset import TextAsset, TextStyle

title = TextAsset(
    text="Welcome to the Demo",
    duration=5,
    style=TextStyle(
        fontsize=36,
        fontcolor="white",
        boxcolor="black",
        alpha=0.8,
        font="Sans",
    ),
)

# Overlay the title at the very start (t=0)
timeline.add_overlay(0, title)
```

### TextStyleのパラメーター

| パラメーター | 型 | デフォルト | 説明 |
|-----------|------|---------|-------------|
| `fontsize` | `int` | `24` | フォントサイズ（ピクセル） |
| `fontcolor` | `str` | `"black"` | CSS色名または16進数 |
| `fontcolor_expr` | `str` | `""` | 動的フォントカラー式 |
| `alpha` | `float` | `1.0` | テキストの不透明度（0.0〜1.0） |
| `font` | `str` | `"Sans"` | フォントファミリー |
| `box` | `bool` | `True` | 背景ボックスを有効にする |
| `boxcolor` | `str` | `"white"` | 背景ボックスの色 |
| `boxborderw` | `str` | `"10"` | ボックスの境界線幅 |
| `boxw` | `int` | `0` | ボックス幅の上書き |
| `boxh` | `int` | `0` | ボックス高さの上書き |
| `line_spacing` | `int` | `0` | 行間隔 |
| `text_align` | `str` | `"T"` | ボックス内のテキスト配置 |
| `y_align` | `str` | `"text"` | 垂直配置の基準 |
| `borderw` | `int` | `0` | テキストの境界線幅 |
| `bordercolor` | `str` | `"black"` | テキストの境界線の色 |
| `expansion` | `str` | `"normal"` | テキスト拡張モード |
| `basetime` | `int` | `0` | 時間ベース式の基準時間 |
| `fix_bounds` | `bool` | `False` | テキストの境界を固定する |
| `text_shaping` | `bool` | `True` | テキストシェーピングを有効にする |
| `shadowcolor` | `str` | `"black"` | 影の色 |
| `shadowx` | `int` | `0` | 影のXオフセット |
| `shadowy` | `int` | `0` | 影のYオフセット |
| `tabsize` | `int` | `4` | スペース単位のタブサイズ |
| `x` | `str` | `"(main_w-text_w)/2"` | 水平位置の式 |
| `y` | `str` | `"(main_h-text_h)/2"` | 垂直位置の式 |

## オーディオオーバーレイ

動画トラックの上にBGM、効果音、ボイスオーバーをレイヤー追加します:

```python
from videodb.asset import AudioAsset

music = coll.get_audio(music_id)

audio_layer = AudioAsset(
    asset_id=music.id,
    disable_other_tracks=False,
    fade_in_duration=2,
    fade_out_duration=2,
)

# Start the music at t=0, overlaid on the video track
timeline.add_overlay(0, audio_layer)
```

### AudioAssetのパラメーター

| パラメーター | 型 | デフォルト | 説明 |
|-----------|------|---------|-------------|
| `asset_id` | `str` | 必須 | 音声メディアID |
| `start` | `float` | `0` | トリム開始位置（秒） |
| `end` | `float\|None` | `None` | トリム終了位置（`None` = 全体） |
| `disable_other_tracks` | `bool` | `True` | Trueの場合、他のオーディオトラックをミュートする |
| `fade_in_duration` | `float` | `0` | フェードイン秒数（最大5） |
| `fade_out_duration` | `float` | `0` | フェードアウト秒数（最大5） |

## 画像オーバーレイ

ロゴ、透かし、生成された画像をオーバーレイとして追加します:

```python
from videodb.asset import ImageAsset

logo = coll.get_image(logo_id)

logo_overlay = ImageAsset(
    asset_id=logo.id,
    duration=10,
    width=120,
    height=60,
    x=20,
    y=20,
)

timeline.add_overlay(0, logo_overlay)
```

### ImageAssetのパラメーター

| パラメーター | 型 | デフォルト | 説明 |
|-----------|------|---------|-------------|
| `asset_id` | `str` | 必須 | 画像メディアID |
| `width` | `int\|str` | `100` | 表示幅 |
| `height` | `int\|str` | `100` | 表示高さ |
| `x` | `int` | `80` | 水平位置（左からのピクセル） |
| `y` | `int` | `20` | 垂直位置（上からのピクセル） |
| `duration` | `float\|None` | `None` | 表示時間（秒） |

## キャプションオーバーレイ

動画にキャプションを追加する方法は2つあります。

### 方法1: 字幕ワークフロー（最もシンプル）

`video.add_subtitle()` を使用して字幕を動画ストリームに直接バーンインします。これは内部的に `videodb.timeline.Timeline` を使用します:

```python
from videodb import SubtitleStyle

# Video must have spoken words indexed first (force=True skips if already done)
video.index_spoken_words(force=True)

# Add subtitles with default styling
stream_url = video.add_subtitle()

# Or customise the subtitle style
stream_url = video.add_subtitle(style=SubtitleStyle(
    font_name="Arial",
    font_size=22,
    primary_colour="&H00FFFFFF",
    bold=True,
))
```

### 方法2: Editor API（高度な使い方）

Editor API（`videodb.editor`）は、`CaptionAsset`、`Clip`、`Track`、独自の `Timeline` を備えたトラックベースの合成システムを提供します。これは上記で使用した `videodb.timeline.Timeline` とは別のAPIです。

```python
from videodb.editor import (
    CaptionAsset,
    Clip,
    Track,
    Timeline as EditorTimeline,
    FontStyling,
    BorderAndShadow,
    Positioning,
    CaptionAnimation,
)

# Video must have spoken words indexed first (force=True skips if already done)
video.index_spoken_words(force=True)

# Create a caption asset
caption = CaptionAsset(
    src="auto",
    font=FontStyling(name="Clear Sans", size=30),
    primary_color="&H00FFFFFF",
    back_color="&H00000000",
    border=BorderAndShadow(outline=1),
    position=Positioning(margin_v=30),
    animation=CaptionAnimation.box_highlight,
)

# Build an editor timeline with tracks and clips
editor_tl = EditorTimeline(conn)
track = Track()
track.add_clip(start=0, clip=Clip(asset=caption, duration=video.length))
editor_tl.add_track(track)
stream_url = editor_tl.generate_stream()
```

### CaptionAssetのパラメーター

| パラメーター | 型 | デフォルト | 説明 |
|-----------|------|---------|-------------|
| `src` | `str` | `"auto"` | キャプションソース（`"auto"` またはbase64 ASS文字列） |
| `font` | `FontStyling\|None` | `FontStyling()` | フォントスタイリング（名前、サイズ、太字、斜体など） |
| `primary_color` | `str` | `"&H00FFFFFF"` | プライマリテキストカラー（ASS形式） |
| `secondary_color` | `str` | `"&H000000FF"` | セカンダリテキストカラー（ASS形式） |
| `back_color` | `str` | `"&H00000000"` | 背景色（ASS形式） |
| `border` | `BorderAndShadow\|None` | `BorderAndShadow()` | 境界線と影のスタイリング |
| `position` | `Positioning\|None` | `Positioning()` | キャプションの配置とマージン |
| `animation` | `CaptionAnimation\|None` | `None` | アニメーション効果（例: `box_highlight`、`reveal`、`karaoke`） |

## コンパイルとストリーミング

タイムラインを組み立てた後、それをストリーム可能なURLにコンパイルします。ストリームは即座に生成され、レンダリングの待ち時間はありません。

```python
stream_url = timeline.generate_stream()
print(f"Stream: {stream_url}")
```

より多くのストリーミングオプション（セグメントストリーム、検索からストリーム、音声再生）については、[streaming.md](streaming.md) を参照してください。

## 完全なワークフロー例

### タイトルカード付きハイライトリール

```python
import videodb
from videodb import SearchType
from videodb.exceptions import InvalidRequestError
from videodb.timeline import Timeline
from videodb.asset import VideoAsset, TextAsset, TextStyle

conn = videodb.connect()
coll = conn.get_collection()
video = coll.get_video("your-video-id")

# 1. Search for key moments
video.index_spoken_words(force=True)
try:
    results = video.search("product announcement", search_type=SearchType.semantic)
    shots = results.get_shots()
except InvalidRequestError as exc:
    if "No results found" in str(exc):
        shots = []
    else:
        raise

# 2. Build timeline
timeline = Timeline(conn)

# Title card
title = TextAsset(
    text="Product Launch Highlights",
    duration=4,
    style=TextStyle(fontsize=48, fontcolor="white", boxcolor="#1a1a2e", alpha=0.95),
)
timeline.add_overlay(0, title)

# Append each matching clip
for shot in shots:
    asset = VideoAsset(asset_id=shot.video_id, start=shot.start, end=shot.end)
    timeline.add_inline(asset)

# 3. Generate stream
stream_url = timeline.generate_stream()
print(f"Highlight reel: {stream_url}")
```

### BGM付きロゴオーバーレイ

```python
import videodb
from videodb.timeline import Timeline
from videodb.asset import VideoAsset, AudioAsset, ImageAsset

conn = videodb.connect()
coll = conn.get_collection()

main_video = coll.get_video(main_video_id)
music = coll.get_audio(music_id)
logo = coll.get_image(logo_id)

timeline = Timeline(conn)

# Main video track
timeline.add_inline(VideoAsset(asset_id=main_video.id))

# Background music — disable_other_tracks=False to mix with video audio
timeline.add_overlay(
    0,
    AudioAsset(asset_id=music.id, disable_other_tracks=False, fade_in_duration=3),
)

# Logo in top-right corner for first 10 seconds
timeline.add_overlay(
    0,
    ImageAsset(asset_id=logo.id, duration=10, x=1140, y=20, width=120, height=60),
)

stream_url = timeline.generate_stream()
print(f"Final video: {stream_url}")
```

### 複数動画からのマルチクリップモンタージュ

```python
import videodb
from videodb.timeline import Timeline
from videodb.asset import VideoAsset, TextAsset, TextStyle

conn = videodb.connect()
coll = conn.get_collection()

clips = [
    {"video_id": "vid_001", "start": 5, "end": 15, "label": "Scene 1"},
    {"video_id": "vid_002", "start": 0, "end": 20, "label": "Scene 2"},
    {"video_id": "vid_003", "start": 30, "end": 45, "label": "Scene 3"},
]

timeline = Timeline(conn)
timeline_offset = 0.0

for clip in clips:
    # Add a label as an overlay on each clip
    label = TextAsset(
        text=clip["label"],
        duration=2,
        style=TextStyle(fontsize=32, fontcolor="white", boxcolor="#333333"),
    )
    timeline.add_inline(
        VideoAsset(asset_id=clip["video_id"], start=clip["start"], end=clip["end"])
    )
    timeline.add_overlay(timeline_offset, label)
    timeline_offset += clip["end"] - clip["start"]

stream_url = timeline.generate_stream()
print(f"Montage: {stream_url}")
```

## 2つのタイムラインAPI

VideoDBには2つの独立したタイムラインシステムがあります。**互換性はありません**:

| | `videodb.timeline.Timeline` | `videodb.editor.Timeline`（Editor API） |
|---|---|---|
| **インポート** | `from videodb.timeline import Timeline` | `from videodb.editor import Timeline as EditorTimeline` |
| **アセット** | `VideoAsset`、`AudioAsset`、`ImageAsset`、`TextAsset` | `CaptionAsset`、`Clip`、`Track` |
| **メソッド** | `add_inline()`、`add_overlay()` | `Track` / `Clip` を使った `add_track()` |
| **最適な用途** | 動画合成、オーバーレイ、マルチクリップ編集 | アニメーション付きキャプション/字幕スタイリング |

一方のAPIのアセットをもう一方に混在させないでください。`CaptionAsset` はEditor APIでのみ機能します。`VideoAsset` / `AudioAsset` / `ImageAsset` / `TextAsset` は `videodb.timeline.Timeline` でのみ機能します。

## 制限事項と制約

タイムラインエディターは**非破壊的な線形合成**のために設計されています。以下の操作は**サポートされていません**:

### 不可能な操作

| 制限 | 詳細 |
|---|---|
| **トランジションやエフェクトなし** | クリップ間のクロスフェード、ワイプ、ディゾルブ、トランジションはありません。すべてのカットはハードカットです。 |
| **ビデオオンビデオなし（ピクチャーインピクチャー）** | `add_inline()` は `VideoAsset` のみを受け付けます。別の動画ストリームを動画の上に重ねることはできません。画像オーバーレイで静的なPiPを近似できますが、ライブ動画はできません。 |
| **速度や再生コントロールなし** | スローモーション、早送り、逆再生、タイムリマッピングはありません。`VideoAsset` には `speed` パラメーターがありません。 |
| **クロップ、ズーム、パンなし** | 動画フレームの領域をクロップしたり、ズームエフェクトを適用したり、フレームをパンしたりすることはできません。`video.reframe()` はアスペクト比変換専用です。 |
| **動画フィルターやカラーグレーディングなし** | 明るさ、コントラスト、彩度、色相、色補正の調整はありません。 |
| **テキストアニメーションなし** | `TextAsset` はその全期間静的です。フェードイン/アウト、移動、アニメーションはありません。アニメーションキャプションには、Editor APIの `CaptionAsset` を使用してください。 |
| **混合テキストスタイリングなし** | 単一の `TextAsset` には1つの `TextStyle` があります。単一テキストブロック内でボールド、イタリック、色を混在させることはできません。 |
| **空白または単色クリップなし** | 単色フレーム、黒いスクリーン、スタンドアロンのタイトルカードを作成することはできません。テキストと画像のオーバーレイはインライントラック上に `VideoAsset` が必要です。 |
| **音量コントロールなし** | `AudioAsset` には `volume` パラメーターがありません。音声はフルボリュームか、`disable_other_tracks` でミュートされるかのどちらかです。低いレベルでミックスすることはできません。 |
| **キーフレームアニメーションなし** | オーバーレイのプロパティを時間をかけて変化させることはできません（例: 画像を位置Aから位置Bに移動する）。 |

### 制約

| 制約 | 詳細 |
|---|---|
| **オーディオフェード最大5秒** | `fade_in_duration` と `fade_out_duration` はそれぞれ最大5秒です。 |
| **オーバーレイの配置は絶対値** | オーバーレイはタイムライン開始からの絶対タイムスタンプを使用します。インラインクリップを並べ替えても、そのオーバーレイは移動しません。 |
| **インライントラックは動画のみ** | `add_inline()` は `VideoAsset` のみを受け付けます。音声、画像、テキストは `add_overlay()` を使用する必要があります。 |
| **オーバーレイとクリップのバインディングなし** | オーバーレイは固定のタイムラインタイムスタンプに配置されます。特定のインラインクリップにオーバーレイをアタッチして一緒に移動させる方法はありません。 |

## ヒント

- **非破壊的**: タイムラインはソースメディアを変更しません。同じアセットから複数のタイムラインを作成できます。
- **オーバーレイのスタッキング**: 複数のオーバーレイを同じタイムスタンプから開始できます。オーディオオーバーレイは混合され、画像/テキストオーバーレイは追加順にレイヤー化されます。
- **インラインはVideoAssetのみ**: `add_inline()` は `VideoAsset` のみを受け付けます。`AudioAsset`、`ImageAsset`、`TextAsset` には `add_overlay()` を使用します。
- **トリム精度**: `VideoAsset` と `AudioAsset` の `start`/`end` は秒単位です。
- **動画音声のミュート**: 音楽やナレーションをオーバーレイする際に元の動画音声をミュートするには、`AudioAsset` の `disable_other_tracks=True` を設定します。
- **フェードの制限**: `AudioAsset` の `fade_in_duration` と `fade_out_duration` は最大5秒です。
- **生成メディア**: `coll.generate_music()`、`coll.generate_sound_effect()`、`coll.generate_voice()`、`coll.generate_image()` を使用して、タイムラインアセットとしてすぐに使用できるメディアを作成できます。
