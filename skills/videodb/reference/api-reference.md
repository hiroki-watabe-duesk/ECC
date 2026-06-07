# 完全 API リファレンス

VideoDB スキルのリファレンス資料です。使用方法やワークフローの選択については、まず [../SKILL.md](../SKILL.md) をご覧ください。

## 接続

```python
import videodb

conn = videodb.connect(
    api_key="your-api-key",      # or set VIDEO_DB_API_KEY env var
    base_url=None,                # custom API endpoint (optional)
)
```

**戻り値:** `Connection` オブジェクト

### 接続メソッド

| メソッド | 戻り値 | 説明 |
|--------|---------|-------------|
| `conn.get_collection(collection_id="default")` | `Collection` | コレクションを取得する（ID 未指定の場合はデフォルト） |
| `conn.get_collections()` | `list[Collection]` | すべてのコレクションを一覧表示する |
| `conn.create_collection(name, description, is_public=False)` | `Collection` | 新しいコレクションを作成する |
| `conn.update_collection(id, name, description)` | `Collection` | コレクションを更新する |
| `conn.check_usage()` | `dict` | アカウントの使用状況を取得する |
| `conn.upload(source, media_type, name, ...)` | `Video\|Audio\|Image` | デフォルトコレクションにアップロードする |
| `conn.record_meeting(meeting_url, bot_name, ...)` | `Meeting` | ミーティングを録画する |
| `conn.create_capture_session(...)` | `CaptureSession` | キャプチャセッションを作成する（[capture-reference.md](capture-reference.md) を参照） |
| `conn.youtube_search(query, result_threshold, duration)` | `list[dict]` | YouTube を検索する |
| `conn.transcode(source, callback_url, mode, ...)` | `str` | 動画をトランスコードする（ジョブ ID を返す） |
| `conn.get_transcode_details(job_id)` | `dict` | トランスコードジョブのステータスと詳細を取得する |
| `conn.connect_websocket(collection_id)` | `WebSocketConnection` | WebSocket に接続する（[capture-reference.md](capture-reference.md) を参照） |

### トランスコード

URL から動画をカスタム解像度、品質、オーディオ設定でトランスコードします。処理はサーバー側で行われるため、ローカルの ffmpeg は不要です。

```python
from videodb import TranscodeMode, VideoConfig, AudioConfig

job_id = conn.transcode(
    source="https://example.com/video.mp4",
    callback_url="https://example.com/webhook",
    mode=TranscodeMode.economy,
    video_config=VideoConfig(resolution=720, quality=23),
    audio_config=AudioConfig(mute=False),
)
```

#### transcode パラメーター

| パラメーター | 型 | デフォルト | 説明 |
|-----------|------|---------|-------------|
| `source` | `str` | 必須 | トランスコードする動画の URL（ダウンロード可能な URL が望ましい） |
| `callback_url` | `str` | 必須 | トランスコード完了時にコールバックを受け取る URL |
| `mode` | `TranscodeMode` | `TranscodeMode.economy` | トランスコード速度: `economy` または `lightning` |
| `video_config` | `VideoConfig` | `VideoConfig()` | 映像エンコード設定 |
| `audio_config` | `AudioConfig` | `AudioConfig()` | 音声エンコード設定 |

ジョブ ID（`str`）を返します。`conn.get_transcode_details(job_id)` でジョブのステータスを確認してください。

```python
details = conn.get_transcode_details(job_id)
```

#### VideoConfig

```python
from videodb import VideoConfig, ResizeMode

config = VideoConfig(
    resolution=720,              # Target resolution height (e.g. 480, 720, 1080)
    quality=23,                  # Encoding quality (lower = better, default 23)
    framerate=30,                # Target framerate
    aspect_ratio="16:9",         # Target aspect ratio
    resize_mode=ResizeMode.crop, # How to fit: crop, fit, or pad
)
```

| フィールド | 型 | デフォルト | 説明 |
|-------|------|---------|-------------|
| `resolution` | `int\|None` | `None` | 目標解像度の高さ（ピクセル） |
| `quality` | `int` | `23` | エンコード品質（値が低いほど高品質） |
| `framerate` | `int\|None` | `None` | 目標フレームレート |
| `aspect_ratio` | `str\|None` | `None` | 目標アスペクト比（例: `"16:9"`、`"9:16"`） |
| `resize_mode` | `str` | `ResizeMode.crop` | リサイズ方法: `crop`、`fit`、または `pad` |

#### AudioConfig

```python
from videodb import AudioConfig

config = AudioConfig(mute=False)
```

| フィールド | 型 | デフォルト | 説明 |
|-------|------|---------|-------------|
| `mute` | `bool` | `False` | 音声トラックをミュートする |

## コレクション

```python
coll = conn.get_collection()
```

### コレクションメソッド

| メソッド | 戻り値 | 説明 |
|--------|---------|-------------|
| `coll.get_videos()` | `list[Video]` | すべての動画を一覧表示する |
| `coll.get_video(video_id)` | `Video` | 特定の動画を取得する |
| `coll.get_audios()` | `list[Audio]` | すべての音声を一覧表示する |
| `coll.get_audio(audio_id)` | `Audio` | 特定の音声を取得する |
| `coll.get_images()` | `list[Image]` | すべての画像を一覧表示する |
| `coll.get_image(image_id)` | `Image` | 特定の画像を取得する |
| `coll.upload(url=None, file_path=None, media_type=None, name=None)` | `Video\|Audio\|Image` | メディアをアップロードする |
| `coll.search(query, search_type, index_type, score_threshold, namespace, scene_index_id, ...)` | `SearchResult` | コレクション全体を検索する（セマンティック検索のみ。キーワード検索とシーン検索は `NotImplementedError` を発生させる） |
| `coll.generate_image(prompt, aspect_ratio="1:1")` | `Image` | AI で画像を生成する |
| `coll.generate_video(prompt, duration=5)` | `Video` | AI で動画を生成する |
| `coll.generate_music(prompt, duration=5)` | `Audio` | AI で音楽を生成する |
| `coll.generate_sound_effect(prompt, duration=2)` | `Audio` | 効果音を生成する |
| `coll.generate_voice(text, voice_name="Default")` | `Audio` | テキストから音声を生成する |
| `coll.generate_text(prompt, model_name="basic", response_type="text")` | `dict` | LLM によるテキスト生成 — `["output"]` で結果にアクセスする |
| `coll.dub_video(video_id, language_code)` | `Video` | 動画を別の言語に吹き替える |
| `coll.record_meeting(meeting_url, bot_name, ...)` | `Meeting` | ライブミーティングを録画する |
| `coll.create_capture_session(...)` | `CaptureSession` | キャプチャセッションを作成する（[capture-reference.md](capture-reference.md) を参照） |
| `coll.get_capture_session(...)` | `CaptureSession` | キャプチャセッションを取得する（[capture-reference.md](capture-reference.md) を参照） |
| `coll.connect_rtstream(url, name, ...)` | `RTStream` | ライブストリームに接続する（[rtstream-reference.md](rtstream-reference.md) を参照） |
| `coll.make_public()` | `None` | コレクションを公開にする |
| `coll.make_private()` | `None` | コレクションを非公開にする |
| `coll.delete_video(video_id)` | `None` | 動画を削除する |
| `coll.delete_audio(audio_id)` | `None` | 音声を削除する |
| `coll.delete_image(image_id)` | `None` | 画像を削除する |
| `coll.delete()` | `None` | コレクションを削除する |

### アップロードパラメーター

```python
video = coll.upload(
    url=None,            # Remote URL (HTTP, YouTube)
    file_path=None,      # Local file path
    media_type=None,     # "video", "audio", or "image" (auto-detected if omitted)
    name=None,           # Custom name for the media
    description=None,    # Description
    callback_url=None,   # Webhook URL for async notification
)
```

## Video オブジェクト

```python
video = coll.get_video(video_id)
```

### Video プロパティ

| プロパティ | 型 | 説明 |
|----------|------|-------------|
| `video.id` | `str` | 動画の一意な ID |
| `video.collection_id` | `str` | 親コレクションの ID |
| `video.name` | `str` | 動画名 |
| `video.description` | `str` | 動画の説明 |
| `video.length` | `float` | 秒単位の再生時間 |
| `video.stream_url` | `str` | デフォルトのストリーム URL |
| `video.player_url` | `str` | プレイヤーの埋め込み URL |
| `video.thumbnail_url` | `str` | サムネイル URL |

### Video メソッド

| メソッド | 戻り値 | 説明 |
|--------|---------|-------------|
| `video.generate_stream(timeline=None)` | `str` | ストリーム URL を生成する（オプションで `[(start, end)]` タプルのタイムラインを指定可能） |
| `video.play()` | `str` | ブラウザでストリームを開き、プレイヤー URL を返す |
| `video.index_spoken_words(language_code=None, force=False)` | `None` | 検索用に音声をインデックス化する。既にインデックス済みの場合にスキップするには `force=True` を使用。 |
| `video.index_scenes(extraction_type, prompt, extraction_config, metadata, model_name, name, scenes, callback_url)` | `str` | 視覚的シーンをインデックス化する（scene_index_id を返す） |
| `video.index_visuals(prompt, batch_config, ...)` | `str` | ビジュアルをインデックス化する（scene_index_id を返す） |
| `video.index_audio(prompt, model_name, ...)` | `str` | LLM を使って音声をインデックス化する（scene_index_id を返す） |
| `video.get_transcript(start=None, end=None)` | `list[dict]` | タイムスタンプ付きのトランスクリプトを取得する |
| `video.get_transcript_text(start=None, end=None)` | `str` | トランスクリプト全文を取得する |
| `video.generate_transcript(force=None)` | `dict` | トランスクリプトを生成する |
| `video.translate_transcript(language, additional_notes)` | `list[dict]` | トランスクリプトを翻訳する |
| `video.search(query, search_type, index_type, filter, **kwargs)` | `SearchResult` | 動画内を検索する |
| `video.add_subtitle(style=SubtitleStyle())` | `str` | 字幕を追加する（ストリーム URL を返す） |
| `video.generate_thumbnail(time=None)` | `str\|Image` | サムネイルを生成する |
| `video.get_thumbnails()` | `list[Image]` | すべてのサムネイルを取得する |
| `video.extract_scenes(extraction_type, extraction_config)` | `SceneCollection` | シーンを抽出する |
| `video.reframe(start, end, target, mode, callback_url)` | `Video\|None` | 動画のアスペクト比を変換する |
| `video.clip(prompt, content_type, model_name)` | `str` | プロンプトからクリップを生成する（ストリーム URL を返す） |
| `video.insert_video(video, timestamp)` | `str` | タイムスタンプの位置に動画を挿入する |
| `video.download(name=None)` | `dict` | 動画をダウンロードする |
| `video.delete()` | `None` | 動画を削除する |

### リフレーム

動画をオプションのスマートオブジェクト追跡付きで別のアスペクト比に変換します。処理はサーバー側で行われます。

> **警告:** リフレームはサーバー側の低速な処理です。長い動画では数分かかることがあり、タイムアウトすることもあります。必ず `start`/`end` でセグメントを制限するか、非同期処理のために `callback_url` を渡してください。

```python
from videodb import ReframeMode

# Always prefer short segments to avoid timeouts:
reframed = video.reframe(start=0, end=60, target="vertical", mode=ReframeMode.smart)

# Async reframe for full-length videos (returns None, result via webhook):
video.reframe(target="vertical", callback_url="https://example.com/webhook")

# Custom dimensions
reframed = video.reframe(start=0, end=60, target={"width": 1080, "height": 1080})
```

#### reframe パラメーター

| パラメーター | 型 | デフォルト | 説明 |
|-----------|------|---------|-------------|
| `start` | `float\|None` | `None` | 開始時間（秒）（None = 先頭から） |
| `end` | `float\|None` | `None` | 終了時間（秒）（None = 動画末尾まで） |
| `target` | `str\|dict` | `"vertical"` | プリセット文字列（`"vertical"`、`"square"`、`"landscape"`）または `{"width": int, "height": int}` |
| `mode` | `str` | `ReframeMode.smart` | `"simple"`（中央クロップ）または `"smart"`（オブジェクト追跡） |
| `callback_url` | `str\|None` | `None` | 非同期通知のための Webhook URL |

`callback_url` が指定されていない場合は `Video` オブジェクトを、指定されている場合は `None` を返します。

## Audio オブジェクト

```python
audio = coll.get_audio(audio_id)
```

### Audio プロパティ

| プロパティ | 型 | 説明 |
|----------|------|-------------|
| `audio.id` | `str` | 音声の一意な ID |
| `audio.collection_id` | `str` | 親コレクションの ID |
| `audio.name` | `str` | 音声名 |
| `audio.length` | `float` | 秒単位の再生時間 |

### Audio メソッド

| メソッド | 戻り値 | 説明 |
|--------|---------|-------------|
| `audio.generate_url()` | `str` | 再生用の署名付き URL を生成する |
| `audio.get_transcript(start=None, end=None)` | `list[dict]` | タイムスタンプ付きのトランスクリプトを取得する |
| `audio.get_transcript_text(start=None, end=None)` | `str` | トランスクリプト全文を取得する |
| `audio.generate_transcript(force=None)` | `dict` | トランスクリプトを生成する |
| `audio.delete()` | `None` | 音声を削除する |

## Image オブジェクト

```python
image = coll.get_image(image_id)
```

### Image プロパティ

| プロパティ | 型 | 説明 |
|----------|------|-------------|
| `image.id` | `str` | 画像の一意な ID |
| `image.collection_id` | `str` | 親コレクションの ID |
| `image.name` | `str` | 画像名 |
| `image.url` | `str\|None` | 画像 URL（生成画像の場合は `None` になることがある — 代わりに `generate_url()` を使用） |

### Image メソッド

| メソッド | 戻り値 | 説明 |
|--------|---------|-------------|
| `image.generate_url()` | `str` | 署名付き URL を生成する |
| `image.delete()` | `None` | 画像を削除する |

## タイムライン & エディター

### タイムライン

```python
from videodb.timeline import Timeline

timeline = Timeline(conn)
```

| メソッド | 戻り値 | 説明 |
|--------|---------|-------------|
| `timeline.add_inline(asset)` | `None` | メイントラックに `VideoAsset` を順番に追加する |
| `timeline.add_overlay(start, asset)` | `None` | タイムスタンプの位置に `AudioAsset`、`ImageAsset`、または `TextAsset` をオーバーレイする |
| `timeline.generate_stream()` | `str` | コンパイルしてストリーム URL を取得する |

### アセットタイプ

#### VideoAsset

```python
from videodb.asset import VideoAsset

asset = VideoAsset(
    asset_id=video.id,
    start=0,              # trim start (seconds)
    end=None,             # trim end (seconds, None = full)
)
```

#### AudioAsset

```python
from videodb.asset import AudioAsset

asset = AudioAsset(
    asset_id=audio.id,
    start=0,
    end=None,
    disable_other_tracks=True,   # mute original audio when True
    fade_in_duration=0,          # seconds (max 5)
    fade_out_duration=0,         # seconds (max 5)
)
```

#### ImageAsset

```python
from videodb.asset import ImageAsset

asset = ImageAsset(
    asset_id=image.id,
    duration=None,        # display duration (seconds)
    width=100,            # display width
    height=100,           # display height
    x=80,                 # horizontal position (px from left)
    y=20,                 # vertical position (px from top)
)
```

#### TextAsset

```python
from videodb.asset import TextAsset, TextStyle

asset = TextAsset(
    text="Hello World",
    duration=5,
    style=TextStyle(
        fontsize=24,
        fontcolor="black",
        boxcolor="white",       # background box colour
        alpha=1.0,
        font="Sans",
        text_align="T",         # text alignment within box
    ),
)
```

#### CaptionAsset（エディター API）

CaptionAsset はエディター API に属しており、独自のタイムライン、トラック、クリップシステムを持ちます:

```python
from videodb.editor import CaptionAsset, FontStyling

asset = CaptionAsset(
    src="auto",                    # "auto" or base64 ASS string
    font=FontStyling(name="Clear Sans", size=30),
    primary_color="&H00FFFFFF",
)
```

エディター API での CaptionAsset の完全な使用方法については [editor.md](editor.md#caption-overlays) を参照してください。

## 動画検索パラメーター

```python
results = video.search(
    query="your query",
    search_type=SearchType.semantic,       # semantic, keyword, or scene
    index_type=IndexType.spoken_word,      # spoken_word or scene
    result_threshold=None,                 # max number of results
    score_threshold=None,                  # minimum relevance score
    dynamic_score_percentage=None,         # percentage of dynamic score
    scene_index_id=None,                   # target a specific scene index (pass via **kwargs)
    filter=[],                             # metadata filters for scene search
)
```

> **注意:** `filter` は `video.search()` の明示的な名前付きパラメーターです。`scene_index_id` は `**kwargs` を通じて API に渡されます。
>
> **重要:** `video.search()` は一致する結果がない場合、`"No results found"` というメッセージで `InvalidRequestError` を発生させます。検索呼び出しは常に try/except で囲んでください。シーン検索では、低関連性のノイズをフィルタリングするために `score_threshold=0.3` 以上を使用してください。

シーン検索では、`index_type=IndexType.scene` と `search_type=SearchType.semantic` を組み合わせて使用します。特定のシーンインデックスを対象とする場合は `scene_index_id` を渡してください。詳細は [search.md](search.md) を参照してください。

## SearchResult オブジェクト

```python
results = video.search("query", search_type=SearchType.semantic)
```

| メソッド | 戻り値 | 説明 |
|--------|---------|-------------|
| `results.get_shots()` | `list[Shot]` | 一致するセグメントのリストを取得する |
| `results.compile()` | `str` | すべてのショットをストリーム URL にコンパイルする |
| `results.play()` | `str` | コンパイルされたストリームをブラウザで開く |

### Shot プロパティ

| プロパティ | 型 | 説明 |
|----------|------|-------------|
| `shot.video_id` | `str` | ソース動画の ID |
| `shot.video_length` | `float` | ソース動画の再生時間 |
| `shot.video_title` | `str` | ソース動画のタイトル |
| `shot.start` | `float` | 開始時間（秒） |
| `shot.end` | `float` | 終了時間（秒） |
| `shot.text` | `str` | 一致したテキストコンテンツ |
| `shot.search_score` | `float` | 検索の関連性スコア |

| メソッド | 戻り値 | 説明 |
|--------|---------|-------------|
| `shot.generate_stream()` | `str` | この特定のショットをストリーミングする |
| `shot.play()` | `str` | ショットのストリームをブラウザで開く |

## Meeting オブジェクト

```python
meeting = coll.record_meeting(
    meeting_url="https://meet.google.com/...",
    bot_name="Bot",
    callback_url=None,          # Webhook URL for status updates
    callback_data=None,         # Optional dict passed through to callbacks
    time_zone="UTC",            # Time zone for the meeting
)
```

### Meeting プロパティ

| プロパティ | 型 | 説明 |
|----------|------|-------------|
| `meeting.id` | `str` | ミーティングの一意な ID |
| `meeting.collection_id` | `str` | 親コレクションの ID |
| `meeting.status` | `str` | 現在のステータス |
| `meeting.video_id` | `str` | 録画した動画の ID（完了後） |
| `meeting.bot_name` | `str` | ボット名 |
| `meeting.meeting_title` | `str` | ミーティングのタイトル |
| `meeting.meeting_url` | `str` | ミーティングの URL |
| `meeting.speaker_timeline` | `dict` | 話者のタイムラインデータ |
| `meeting.is_active` | `bool` | 初期化中または処理中の場合に True |
| `meeting.is_completed` | `bool` | 完了した場合に True |

### Meeting メソッド

| メソッド | 戻り値 | 説明 |
|--------|---------|-------------|
| `meeting.refresh()` | `Meeting` | サーバーからデータを更新する |
| `meeting.wait_for_status(target_status, timeout=14400, interval=120)` | `bool` | ステータスに達するまでポーリングする |

## RTStream & キャプチャ

RTStream（ライブ取り込み、インデックス化、文字起こし）については [rtstream-reference.md](rtstream-reference.md) を参照してください。

キャプチャセッション（デスクトップ録画、CaptureClient、チャンネル）については [capture-reference.md](capture-reference.md) を参照してください。

## 列挙型 & 定数

### SearchType

```python
from videodb import SearchType

SearchType.semantic    # Natural language semantic search
SearchType.keyword     # Exact keyword matching
SearchType.scene       # Visual scene search (may require paid plan)
SearchType.llm         # LLM-powered search
```

### SceneExtractionType

```python
from videodb import SceneExtractionType

SceneExtractionType.shot_based   # Automatic shot boundary detection
SceneExtractionType.time_based   # Fixed time interval extraction
SceneExtractionType.transcript   # Transcript-based scene extraction
```

### SubtitleStyle

```python
from videodb import SubtitleStyle

style = SubtitleStyle(
    font_name="Arial",
    font_size=18,
    primary_colour="&H00FFFFFF",
    bold=False,
    # ... see SubtitleStyle for all options
)
video.add_subtitle(style=style)
```

### SubtitleAlignment & SubtitleBorderStyle

```python
from videodb import SubtitleAlignment, SubtitleBorderStyle
```

### TextStyle

```python
from videodb import TextStyle
# or: from videodb.asset import TextStyle

style = TextStyle(
    fontsize=24,
    fontcolor="black",
    boxcolor="white",
    font="Sans",
    text_align="T",
    alpha=1.0,
)
```

### その他の定数

```python
from videodb import (
    IndexType,          # spoken_word, scene
    MediaType,          # video, audio, image
    Segmenter,          # word, sentence, time
    SegmentationType,   # sentence, llm
    TranscodeMode,      # economy, lightning
    ResizeMode,         # crop, fit, pad
    ReframeMode,        # simple, smart
    RTStreamChannelType,
)
```

## 例外

```python
from videodb.exceptions import (
    AuthenticationError,     # Invalid or missing API key
    InvalidRequestError,     # Bad parameters or malformed request
    RequestTimeoutError,     # Request timed out
    SearchError,             # Search operation failure (e.g. not indexed)
    VideodbError,            # Base exception for all VideoDB errors
)
```

| 例外 | 一般的な原因 |
|-----------|-------------|
| `AuthenticationError` | `VIDEO_DB_API_KEY` がないまたは無効 |
| `InvalidRequestError` | 無効な URL、サポートされていない形式、不正なパラメーター |
| `RequestTimeoutError` | サーバーの応答に時間がかかりすぎた |
| `SearchError` | インデックス化前の検索、無効な検索タイプ |
| `VideodbError` | サーバーエラー、ネットワーク問題、一般的な障害 |
