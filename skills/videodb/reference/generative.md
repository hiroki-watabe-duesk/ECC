# 生成メディアガイド

VideoDBはAIを活用した画像、動画、音楽、効果音、音声、テキストコンテンツの生成機能を提供する。すべての生成メソッドは **Collection** オブジェクト上にある。

## 前提条件

生成メソッドを呼び出す前に、接続とコレクション参照が必要である:

```python
import videodb

conn = videodb.connect()
coll = conn.get_collection()
```

## 画像生成

テキストプロンプトから画像を生成する:

```python
image = coll.generate_image(
    prompt="a futuristic cityscape at sunset with flying cars",
    aspect_ratio="16:9",
)

# 生成した画像にアクセス
print(image.id)
print(image.generate_url())  # 署名付きダウンロードURLを返す
```

### generate_image パラメータ

| パラメータ | 型 | デフォルト | 説明 |
|-----------|------|---------|-------------|
| `prompt` | `str` | 必須 | 生成する画像のテキスト説明 |
| `aspect_ratio` | `str` | `"1:1"` | アスペクト比: `"1:1"`, `"9:16"`, `"16:9"`, `"4:3"`, または `"3:4"` |
| `callback_url` | `str\|None` | `None` | 非同期コールバックを受け取るURL |

`Image` オブジェクト（`.id`、`.name`、`.collection_id` を持つ）を返す。生成した画像では `.url` プロパティが `None` になる場合がある — 信頼性の高い署名付きダウンロードURLを取得するには常に `image.generate_url()` を使用すること。

> **注意:** `Video` オブジェクト（`.generate_stream()` を使用）とは異なり、`Image` オブジェクトは `.generate_url()` を使用して画像URLを取得する。`.url` プロパティは一部の画像タイプ（サムネイルなど）でのみ設定される。

## 動画生成

テキストプロンプトから短い動画クリップを生成する:

```python
video = coll.generate_video(
    prompt="a timelapse of a flower blooming in a garden",
    duration=5,
)

stream_url = video.generate_stream()
video.play()
```

### generate_video パラメータ

| パラメータ | 型 | デフォルト | 説明 |
|-----------|------|---------|-------------|
| `prompt` | `str` | 必須 | 生成する動画のテキスト説明 |
| `duration` | `int` | `5` | 秒単位の長さ（整数値、5〜8の範囲） |
| `callback_url` | `str\|None` | `None` | 非同期コールバックを受け取るURL |

`Video` オブジェクトを返す。生成された動画は自動的にコレクションに追加され、アップロードされた動画と同様にタイムライン、検索、コンパイルで使用できる。

## 音声生成

VideoDBは異なる音声タイプに対して3つの別々のメソッドを提供する。

### 音楽

テキスト説明からバックグラウンドミュージックを生成する:

```python
music = coll.generate_music(
    prompt="upbeat electronic music with a driving beat, suitable for a tech demo",
    duration=30,
)

print(music.id)
```

| パラメータ | 型 | デフォルト | 説明 |
|-----------|------|---------|-------------|
| `prompt` | `str` | 必須 | 音楽のテキスト説明 |
| `duration` | `int` | `5` | 秒単位の長さ |
| `callback_url` | `str\|None` | `None` | 非同期コールバックを受け取るURL |

### 効果音

特定の効果音を生成する:

```python
sfx = coll.generate_sound_effect(
    prompt="thunderstorm with heavy rain and distant thunder",
    duration=10,
)
```

| パラメータ | 型 | デフォルト | 説明 |
|-----------|------|---------|-------------|
| `prompt` | `str` | 必須 | 効果音のテキスト説明 |
| `duration` | `int` | `2` | 秒単位の長さ |
| `config` | `dict` | `{}` | 追加設定 |
| `callback_url` | `str\|None` | `None` | 非同期コールバックを受け取るURL |

### 音声（テキスト読み上げ）

テキストから音声を生成する:

```python
voice = coll.generate_voice(
    text="Welcome to our product demo. Today we'll walk through the key features.",
    voice_name="Default",
)
```

| パラメータ | 型 | デフォルト | 説明 |
|-----------|------|---------|-------------|
| `text` | `str` | 必須 | 音声に変換するテキスト |
| `voice_name` | `str` | `"Default"` | 使用する音声 |
| `config` | `dict` | `{}` | 追加設定 |
| `callback_url` | `str\|None` | `None` | 非同期コールバックを受け取るURL |

3つの音声メソッドはすべて `.id`、`.name`、`.length`、`.collection_id` を持つ `Audio` オブジェクトを返す。

## テキスト生成（LLM統合）

`coll.generate_text()` を使用してLLM分析を実行する。これは **Collectionレベル** のメソッドで、コンテキスト（トランスクリプト、説明など）はプロンプト文字列に直接渡す。

```python
# まず動画からトランスクリプトを取得
transcript_text = video.get_transcript_text()

# コレクションのLLMを使用して分析を生成
result = coll.generate_text(
    prompt=f"Summarize the key points discussed in this video:\n{transcript_text}",
    model_name="pro",
)

print(result["output"])
```

### generate_text パラメータ

| パラメータ | 型 | デフォルト | 説明 |
|-----------|------|---------|-------------|
| `prompt` | `str` | 必須 | LLM向けのコンテキスト付きプロンプト |
| `model_name` | `str` | `"basic"` | モデルティア: `"basic"`, `"pro"`, または `"ultra"` |
| `response_type` | `str` | `"text"` | レスポンス形式: `"text"` または `"json"` |

`output` キーを持つ `dict` を返す。`response_type="text"` の場合、`output` は `str` である。`response_type="json"` の場合、`output` は `dict` である。

```python
result = coll.generate_text(prompt="Summarize this", model_name="pro")
print(result["output"])  # 実際のテキスト/dictにアクセス
```

### LLMによるシーン分析

シーン抽出とテキスト生成を組み合わせる:

```python
from videodb import SceneExtractionType

# まずシーンにインデックスを付ける
scenes = video.index_scenes(
    extraction_type=SceneExtractionType.time_based,
    extraction_config={"time": 10},
    prompt="Describe the visual content in this scene.",
)

# 音声コンテキスト用のトランスクリプトを取得
transcript_text = video.get_transcript_text()
scene_descriptions = []
for scene in scenes:
    if isinstance(scene, dict):
        description = scene.get("description") or scene.get("summary")
    else:
        description = getattr(scene, "description", None) or getattr(scene, "summary", None)
    scene_descriptions.append(description or str(scene))

scenes_text = "\n".join(scene_descriptions)

# コレクションのLLMで分析
result = coll.generate_text(
    prompt=(
        f"Given this video transcript:\n{transcript_text}\n\n"
        f"And these visual scene descriptions:\n{scenes_text}\n\n"
        "Based on the spoken and visual content, describe the main topics covered."
    ),
    model_name="pro",
)
print(result["output"])
```

## 吹き替えと翻訳

### 動画の吹き替え

コレクションメソッドを使用して動画を別の言語に吹き替える:

```python
dubbed_video = coll.dub_video(
    video_id=video.id,
    language_code="es",  # スペイン語
)

dubbed_video.play()
```

### dub_video パラメータ

| パラメータ | 型 | デフォルト | 説明 |
|-----------|------|---------|-------------|
| `video_id` | `str` | 必須 | 吹き替える動画のID |
| `language_code` | `str` | 必須 | ターゲット言語コード（例: `"es"`, `"fr"`, `"de"`） |
| `callback_url` | `str\|None` | `None` | 非同期コールバックを受け取るURL |

吹き替えコンテンツを含む `Video` オブジェクトを返す。

### トランスクリプトの翻訳

吹き替えなしで動画のトランスクリプトを翻訳する:

```python
translated = video.translate_transcript(
    language="Spanish",
    additional_notes="Use formal tone",
)

for entry in translated:
    print(entry)
```

**対応言語**には以下が含まれる: `en`, `es`, `fr`, `de`, `it`, `pt`, `ja`, `ko`, `zh`, `hi`, `ar` など。

## 完全なワークフロー例

### 動画のナレーション生成

```python
import videodb

conn = videodb.connect()
coll = conn.get_collection()
video = coll.get_video("your-video-id")

# トランスクリプトを取得
transcript_text = video.get_transcript_text()

# コレクションのLLMを使用してナレーションスクリプトを生成
result = coll.generate_text(
    prompt=(
        f"Write a professional narration script for this video content:\n"
        f"{transcript_text[:2000]}"
    ),
    model_name="pro",
)
script = result["output"]

# スクリプトを音声に変換
narration = coll.generate_voice(text=script)
print(f"Narration audio: {narration.id}")
```

### プロンプトからサムネイルを生成

```python
thumbnail = coll.generate_image(
    prompt="professional video thumbnail showing data analytics dashboard, modern design",
    aspect_ratio="16:9",
)
print(f"Thumbnail URL: {thumbnail.generate_url()}")
```

### 生成した音楽を動画に追加

```python
import videodb
from videodb.timeline import Timeline
from videodb.asset import VideoAsset, AudioAsset

conn = videodb.connect()
coll = conn.get_collection()
video = coll.get_video("your-video-id")

# バックグラウンドミュージックを生成
music = coll.generate_music(
    prompt="calm ambient background music for a tutorial video",
    duration=60,
)

# 動画 + 音楽オーバーレイのタイムラインを構築
timeline = Timeline(conn)
timeline.add_inline(VideoAsset(asset_id=video.id))
timeline.add_overlay(0, AudioAsset(asset_id=music.id, disable_other_tracks=False))

stream_url = timeline.generate_stream()
print(f"Video with music: {stream_url}")
```

### 構造化JSONの出力

```python
transcript_text = video.get_transcript_text()

result = coll.generate_text(
    prompt=(
        f"Given this transcript:\n{transcript_text}\n\n"
        "Return a JSON object with keys: summary, topics (array), action_items (array)."
    ),
    model_name="pro",
    response_type="json",
)

# response_type="json" の場合、result["output"] は dict
print(result["output"]["summary"])
print(result["output"]["topics"])
```

## ヒント

- **生成されたメディアは永続的**: 生成されたコンテンツはすべてコレクションに保存され、再利用できる。
- **3つの音声メソッド**: バックグラウンドミュージックには `generate_music()`、効果音には `generate_sound_effect()`、テキスト読み上げには `generate_voice()` を使用する。統一された `generate_audio()` メソッドは存在しない。
- **テキスト生成はコレクションレベル**: `coll.generate_text()` は動画コンテンツに自動的にアクセスできない。`video.get_transcript_text()` でトランスクリプトを取得してプロンプトに渡すこと。
- **モデルティア**: `"basic"` が最も高速で、`"pro"` はバランスが取れており、`"ultra"` は最高品質。ほとんどの分析タスクには `"pro"` を使用する。
- **生成タイプを組み合わせる**: オーバーレイ用の画像、バックグラウンド用の音楽、ナレーション用の音声を生成し、タイムラインを使って合成する（[editor.md](editor.md) を参照）。
- **プロンプトの品質が重要**: 詳細で具体的なプロンプトは、すべての生成タイプでより良い結果を生む。
- **画像のアスペクト比**: `"1:1"`, `"9:16"`, `"16:9"`, `"4:3"`, `"3:4"` から選択する。
