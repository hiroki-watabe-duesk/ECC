# ストリーミングと再生

VideoDB はオンデマンドでストリームを生成し、あらゆる標準ビデオプレーヤーで即座に再生可能な HLS 互換の URL を返します。レンダリング時間やエクスポートの待ち時間はなく、編集、検索、コンポジションをすぐにストリーミングできます。

## 前提条件

ストリームを生成する前に、動画をコレクションに**アップロードしておく必要があります**。検索ベースのストリームでは、動画が**インデックス化**されている（発話内容やシーンが対象）必要もあります。インデックス化の詳細は [search.md](search.md) を参照してください。

## コアコンセプト

### ストリーム生成

VideoDB のすべての動画、検索結果、タイムラインは**ストリーム URL** を生成できます。この URL はオンデマンドでコンパイルされる HLS（HTTP ライブストリーミング）マニフェストを指します。

```python
# 動画から
stream_url = video.generate_stream()

# タイムラインから
stream_url = timeline.generate_stream()

# 検索結果から
stream_url = results.compile()
```

## 単一動画のストリーミング

### 基本再生

```python
import videodb

conn = videodb.connect()
coll = conn.get_collection()
video = coll.get_video("your-video-id")

# ストリーム URL を生成
stream_url = video.generate_stream()
print(f"Stream: {stream_url}")

# デフォルトブラウザで開く
video.play()
```

### 字幕付き

```python
# まずインデックス化して字幕を追加
video.index_spoken_words(force=True)
stream_url = video.add_subtitle()

# 返された URL にはすでに字幕が含まれている
print(f"Subtitled stream: {stream_url}")
```

### 特定セグメント

タイムスタンプ範囲のタイムラインを渡して動画の一部だけをストリーミングします:

```python
# 10〜30秒と60〜90秒をストリーミング
stream_url = video.generate_stream(timeline=[(10, 30), (60, 90)])
print(f"Segment stream: {stream_url}")
```

## タイムラインコンポジションのストリーミング

マルチアセットのコンポジションをビルドしてリアルタイムにストリーミングします:

```python
import videodb
from videodb.timeline import Timeline
from videodb.asset import VideoAsset, AudioAsset, ImageAsset, TextAsset, TextStyle

conn = videodb.connect()
coll = conn.get_collection()

video = coll.get_video(video_id)
music = coll.get_audio(music_id)

timeline = Timeline(conn)

# メインの動画コンテンツ
timeline.add_inline(VideoAsset(asset_id=video.id))

# バックグラウンドミュージックのオーバーレイ（0秒から開始）
timeline.add_overlay(0, AudioAsset(asset_id=music.id))

# 冒頭のテキストオーバーレイ
timeline.add_overlay(0, TextAsset(
    text="Live Demo",
    duration=3,
    style=TextStyle(fontsize=48, fontcolor="white", boxcolor="#000000"),
))

# コンポジションのストリームを生成
stream_url = timeline.generate_stream()
print(f"Composed stream: {stream_url}")
```

**重要:** `add_inline()` は `VideoAsset` のみを受け付けます。`AudioAsset`、`ImageAsset`、`TextAsset` には `add_overlay()` を使用してください。

タイムライン編集の詳細は [editor.md](editor.md) を参照してください。

## 検索結果のストリーミング

検索結果を一つのストリームにコンパイルしてすべての一致セグメントを再生します:

```python
from videodb import SearchType
from videodb.exceptions import InvalidRequestError

video.index_spoken_words(force=True)
try:
    results = video.search("key announcement", search_type=SearchType.semantic)

    # 一致したショットをすべて一つのストリームにコンパイル
    stream_url = results.compile()
    print(f"Search results stream: {stream_url}")

    # または直接再生
    results.play()
except InvalidRequestError as exc:
    if "No results found" in str(exc):
        print("No matching announcement segments were found.")
    else:
        raise
```

### 個別の検索ヒットをストリーミング

```python
from videodb.exceptions import InvalidRequestError

try:
    results = video.search("product demo", search_type=SearchType.semantic)
    for i, shot in enumerate(results.get_shots()):
        stream_url = shot.generate_stream()
        print(f"Hit {i+1} [{shot.start:.1f}s-{shot.end:.1f}s]: {stream_url}")
except InvalidRequestError as exc:
    if "No results found" in str(exc):
        print("No product demo segments matched the query.")
    else:
        raise
```

## 音声の再生

音声コンテンツの署名付き再生 URL を取得します:

```python
audio = coll.get_audio(audio_id)
playback_url = audio.generate_url()
print(f"Audio URL: {playback_url}")
```

## ワークフローの完全な例

### 検索からストリームへのパイプライン

検索、タイムラインコンポジション、ストリーミングを一つのワークフローに統合します:

```python
import videodb
from videodb import SearchType
from videodb.exceptions import InvalidRequestError
from videodb.timeline import Timeline
from videodb.asset import VideoAsset, TextAsset, TextStyle

conn = videodb.connect()
coll = conn.get_collection()
video = coll.get_video("your-video-id")

video.index_spoken_words(force=True)

# 重要なシーンを検索
queries = ["introduction", "main demo", "Q&A"]
timeline = Timeline(conn)
timeline_offset = 0.0

for query in queries:
    try:
        results = video.search(query, search_type=SearchType.semantic)
        shots = results.get_shots()
    except InvalidRequestError as exc:
        if "No results found" in str(exc):
            shots = []
        else:
            raise

    if not shots:
        continue

    # このバッチがコンパイルされたタイムラインで始まる位置にセクションラベルを追加
    timeline.add_overlay(timeline_offset, TextAsset(
        text=query.title(),
        duration=2,
        style=TextStyle(fontsize=36, fontcolor="white", boxcolor="#222222"),
    ))

    for shot in shots:
        timeline.add_inline(
            VideoAsset(asset_id=shot.video_id, start=shot.start, end=shot.end)
        )
        timeline_offset += shot.end - shot.start

stream_url = timeline.generate_stream()
print(f"Dynamic compilation: {stream_url}")
```

### マルチ動画ストリーム

異なる動画のクリップを一つのストリームに結合します:

```python
import videodb
from videodb.timeline import Timeline
from videodb.asset import VideoAsset

conn = videodb.connect()
coll = conn.get_collection()

video_clips = [
    {"id": "vid_001", "start": 0, "end": 15},
    {"id": "vid_002", "start": 10, "end": 30},
    {"id": "vid_003", "start": 5, "end": 25},
]

timeline = Timeline(conn)
for clip in video_clips:
    timeline.add_inline(
        VideoAsset(asset_id=clip["id"], start=clip["start"], end=clip["end"])
    )

stream_url = timeline.generate_stream()
print(f"Multi-video stream: {stream_url}")
```

### 条件付きストリームアセンブリ

検索の有無に応じて動的にストリームを構築します:

```python
import videodb
from videodb import SearchType
from videodb.exceptions import InvalidRequestError
from videodb.timeline import Timeline
from videodb.asset import VideoAsset, TextAsset, TextStyle

conn = videodb.connect()
coll = conn.get_collection()
video = coll.get_video("your-video-id")

video.index_spoken_words(force=True)

timeline = Timeline(conn)

# 特定のコンテンツを探し、見つからなければ動画全体にフォールバック
topics = ["opening remarks", "technical deep dive", "closing"]

found_any = False
timeline_offset = 0.0
for topic in topics:
    try:
        results = video.search(topic, search_type=SearchType.semantic)
        shots = results.get_shots()
    except InvalidRequestError as exc:
        if "No results found" in str(exc):
            shots = []
        else:
            raise

    if shots:
        found_any = True
        timeline.add_overlay(timeline_offset, TextAsset(
            text=topic.title(),
            duration=2,
            style=TextStyle(fontsize=32, fontcolor="white", boxcolor="#1a1a2e"),
        ))
        for shot in shots:
            timeline.add_inline(
                VideoAsset(asset_id=shot.video_id, start=shot.start, end=shot.end)
            )
            timeline_offset += shot.end - shot.start

if found_any:
    stream_url = timeline.generate_stream()
    print(f"Curated stream: {stream_url}")
else:
    # 動画全体のストリームにフォールバック
    stream_url = video.generate_stream()
    print(f"Full video stream: {stream_url}")
```

### ライブイベントのダイジェスト

イベントの録画を複数のセクションを持つストリーミング可能なダイジェストに加工します:

```python
import videodb
from videodb import SearchType
from videodb.exceptions import InvalidRequestError
from videodb.timeline import Timeline
from videodb.asset import VideoAsset, AudioAsset, ImageAsset, TextAsset, TextStyle

conn = videodb.connect()
coll = conn.get_collection()

# イベント録画をアップロード
event = coll.upload(url="https://example.com/event-recording.mp4")
event.index_spoken_words(force=True)

# バックグラウンドミュージックを生成
music = coll.generate_music(
    prompt="upbeat corporate background music",
    duration=120,
)

# タイトル画像を生成
title_img = coll.generate_image(
    prompt="modern event recap title card, dark background, professional",
    aspect_ratio="16:9",
)

# ダイジェストのタイムラインを構築
timeline = Timeline(conn)
timeline_offset = 0.0

# 検索から主要なビデオセグメントを取得
try:
    keynote = event.search("keynote announcement", search_type=SearchType.semantic)
    keynote_shots = keynote.get_shots()[:5]
except InvalidRequestError as exc:
    if "No results found" in str(exc):
        keynote_shots = []
    else:
        raise
if keynote_shots:
    keynote_start = timeline_offset
    for shot in keynote_shots:
        timeline.add_inline(
            VideoAsset(asset_id=shot.video_id, start=shot.start, end=shot.end)
        )
        timeline_offset += shot.end - shot.start
else:
    keynote_start = None

try:
    demo = event.search("product demo", search_type=SearchType.semantic)
    demo_shots = demo.get_shots()[:5]
except InvalidRequestError as exc:
    if "No results found" in str(exc):
        demo_shots = []
    else:
        raise
if demo_shots:
    demo_start = timeline_offset
    for shot in demo_shots:
        timeline.add_inline(
            VideoAsset(asset_id=shot.video_id, start=shot.start, end=shot.end)
        )
        timeline_offset += shot.end - shot.start
else:
    demo_start = None

# タイトルカード画像をオーバーレイ
timeline.add_overlay(0, ImageAsset(
    asset_id=title_img.id, width=100, height=100, x=80, y=20, duration=5
))

# 正しいタイムラインオフセットにセクションラベルをオーバーレイ
if keynote_start is not None:
    timeline.add_overlay(max(5, keynote_start), TextAsset(
        text="Keynote Highlights",
        duration=3,
        style=TextStyle(fontsize=40, fontcolor="white", boxcolor="#0d1117"),
    ))
if demo_start is not None:
    timeline.add_overlay(max(5, demo_start), TextAsset(
        text="Demo Highlights",
        duration=3,
        style=TextStyle(fontsize=36, fontcolor="white", boxcolor="#0d1117"),
    ))

# バックグラウンドミュージックをオーバーレイ
timeline.add_overlay(0, AudioAsset(
    asset_id=music.id, fade_in_duration=3
))

# 最終的なダイジェストをストリーミング
stream_url = timeline.generate_stream()
print(f"Event recap: {stream_url}")
```

---

## ヒント

- **HLS 互換性**: ストリーム URL は HLS マニフェスト（`.m3u8`）を返します。Safari ではネイティブに動作し、その他のブラウザでは hls.js などのライブラリ経由で動作します。
- **オンデマンドコンパイル**: ストリームはリクエスト時にサーバーサイドでコンパイルされます。初回の再生ではコンパイルのわずかな遅延が発生することがありますが、同じコンポジションの以降の再生はキャッシュされます。
- **キャッシュ**: 引数なしで `video.generate_stream()` を再度呼び出すと、再コンパイルせずにキャッシュされたストリーム URL を返します。
- **セグメントストリーム**: `video.generate_stream(timeline=[(start, end)])` は、完全な `Timeline` オブジェクトを構築せずに特定のクリップをストリーミングする最も速い方法です。
- **インライン vs オーバーレイ**: `add_inline()` は `VideoAsset` のみを受け付け、アセットをメイントラックに順番に配置します。`add_overlay()` は `AudioAsset`、`ImageAsset`、`TextAsset` を受け付け、指定された開始時間に上に重ねて配置します。
- **TextStyle のデフォルト**: `TextStyle` のデフォルトは `font='Sans'`、`fontcolor='black'` です。テキストの背景色には `bgcolor` ではなく `boxcolor` を使用してください。
- **生成との組み合わせ**: `coll.generate_music(prompt, duration)` と `coll.generate_image(prompt, aspect_ratio)` を使ってタイムラインコンポジション用のアセットを作成できます。
- **再生**: `.play()` はデフォルトのシステムブラウザでストリーム URL を開きます。プログラムによる利用には URL 文字列を直接扱ってください。
