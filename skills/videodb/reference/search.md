# 検索＆インデックスガイド

検索を使用すると、自然言語クエリ、キーワード完全一致、または視覚的なシーン説明を使ってビデオ内の特定の瞬間を見つけることができます。

## 前提条件

ビデオは検索前に**インデックスされている必要があります**。インデックス化はビデオごと、インデックスタイプごとに一度だけ行う操作です。

## インデックス化

### 音声ワードインデックス

セマンティック検索とキーワード検索のために、ビデオの書き起こされた音声コンテンツをインデックス化します:

```python
video = coll.get_video(video_id)

# force=True により冪等なインデックス化が可能 — 既にインデックス済みの場合はスキップ
video.index_spoken_words(force=True)
```

音声トラックを書き起こし、話されたコンテンツに対して検索可能なインデックスを構築します。セマンティック検索とキーワード検索に必要です。

**パラメーター:**

| パラメーター | 型 | デフォルト | 説明 |
|-----------|------|---------|-------------|
| `language_code` | `str\|None` | `None` | ビデオの言語コード |
| `segmentation_type` | `SegmentationType` | `SegmentationType.sentence` | セグメンテーションタイプ（`sentence` または `llm`） |
| `force` | `bool` | `False` | 既にインデックス済みの場合はスキップするには `True` に設定（「already exists」エラーを回避） |
| `callback_url` | `str\|None` | `None` | 非同期通知用のWebhook URL |

### シーンインデックス

シーンのAI説明を生成してビジュアルコンテンツをインデックス化します。音声ワードインデックスと同様に、シーンインデックスが既に存在する場合はエラーが発生します。エラーメッセージから既存の `scene_index_id` を抽出してください。

```python
import re
from videodb import SceneExtractionType

try:
    scene_index_id = video.index_scenes(
        extraction_type=SceneExtractionType.shot_based,
        prompt="Describe the visual content, objects, actions, and setting in this scene.",
    )
except Exception as e:
    match = re.search(r"id\s+([a-f0-9]+)", str(e))
    if match:
        scene_index_id = match.group(1)
    else:
        raise
```

**抽出タイプ:**

| タイプ | 説明 | 最適な用途 |
|------|-------------|----------|
| `SceneExtractionType.shot_based` | 映像のショット境界で分割 | 汎用、アクションコンテンツ |
| `SceneExtractionType.time_based` | 固定間隔で分割 | 均一サンプリング、長い静的コンテンツ |
| `SceneExtractionType.transcript` | トランスクリプトセグメントに基づいて分割 | 音声駆動のシーン境界 |

**`time_based` のパラメーター:**

```python
video.index_scenes(
    extraction_type=SceneExtractionType.time_based,
    extraction_config={"time": 5, "select_frames": ["first", "last"]},
    prompt="Describe what is happening in this scene.",
)
```

## 検索タイプ

### セマンティック検索

音声コンテンツに対してマッチングされる自然言語クエリ:

```python
from videodb import SearchType

results = video.search(
    query="explaining the benefits of machine learning",
    search_type=SearchType.semantic,
)
```

クエリとセマンティックに一致する音声コンテンツのセグメントをランキングして返します。

### キーワード検索

書き起こされた音声での完全一致:

```python
results = video.search(
    query="artificial intelligence",
    search_type=SearchType.keyword,
)
```

キーワードまたはフレーズを含むセグメントを返します。

### シーン検索

インデックス化されたシーン説明に対してマッチングされる視覚コンテンツクエリ。事前の `index_scenes()` 呼び出しが必要です。

`index_scenes()` は `scene_index_id` を返します。特定のシーンインデックスを対象にするには（ビデオに複数のシーンインデックスがある場合に特に重要）、`video.search()` にそれを渡します:

```python
from videodb import SearchType, IndexType
from videodb.exceptions import InvalidRequestError

# シーンインデックスに対してセマンティック検索を使用して検索する。
# 低関連度のノイズをフィルタリングするためにscore_thresholdを使用する（推奨: 0.3以上）。
try:
    results = video.search(
        query="person writing on a whiteboard",
        search_type=SearchType.semantic,
        index_type=IndexType.scene,
        scene_index_id=scene_index_id,
        score_threshold=0.3,
    )
    shots = results.get_shots()
except InvalidRequestError as e:
    if "No results found" in str(e):
        shots = []
    else:
        raise
```

**重要な注意事項:**

- `SearchType.semantic` と `index_type=IndexType.scene` の組み合わせを使用する — これが最も信頼性が高く、すべてのプランで動作します。
- `SearchType.scene` は存在しますが、すべてのプランで利用できるわけではありません（例: Freeティア）。`SearchType.semantic` と `IndexType.scene` を優先してください。
- `scene_index_id` パラメーターはオプションです。省略した場合、検索はビデオ上のすべてのシーンインデックスに対して実行されます。特定のインデックスを対象にするには渡してください。
- ビデオごとに複数のシーンインデックスを作成し（異なるプロンプトまたは抽出タイプで）、`scene_index_id` を使用して独立して検索することができます。

### メタデータフィルタリング付きシーン検索

カスタムメタデータでシーンをインデックス化する場合、セマンティック検索とメタデータフィルターを組み合わせることができます:

```python
from videodb import SearchType, IndexType

results = video.search(
    query="a skillful chasing scene",
    search_type=SearchType.semantic,
    index_type=IndexType.scene,
    scene_index_id=scene_index_id,
    filter=[{"camera_view": "road_ahead"}, {"action_type": "chasing"}],
)
```

カスタムメタデータインデックス化とフィルタリング検索の完全な例については、[scene_level_metadata_indexing cookbook](https://github.com/video-db/videodb-cookbook/blob/main/quickstart/scene_level_metadata_indexing.ipynb) を参照してください。

## 結果の操作

### ショットの取得

個別の結果セグメントにアクセスする:

```python
results = video.search("your query")

for shot in results.get_shots():
    print(f"Video: {shot.video_id}")
    print(f"Start: {shot.start:.2f}s")
    print(f"End: {shot.end:.2f}s")
    print(f"Text: {shot.text}")
    print("---")
```

### コンパイル済み結果の再生

すべての一致するセグメントを単一のコンパイル済みビデオとしてストリーム配信する:

```python
results = video.search("your query")
stream_url = results.compile()
results.play()  # コンパイル済みストリームをブラウザで開く
```

### クリップの抽出

特定の結果セグメントをダウンロードまたはストリーム配信する:

```python
for shot in results.get_shots():
    stream_url = shot.generate_stream()
    print(f"Clip: {stream_url}")
```

## コレクションをまたいだ検索

コレクション内のすべてのビデオにわたって検索する:

```python
coll = conn.get_collection()

# コレクション内のすべてのビデオにわたって検索
results = coll.search(
    query="product demo",
    search_type=SearchType.semantic,
)

for shot in results.get_shots():
    print(f"Video: {shot.video_id} [{shot.start:.1f}s - {shot.end:.1f}s]")
```

> **注意:** コレクションレベルの検索は `SearchType.semantic` のみをサポートします。`SearchType.keyword` または `SearchType.scene` を `coll.search()` で使用すると `NotImplementedError` が発生します。キーワードまたはシーン検索には、個々のビデオに対して `video.search()` を使用してください。

## 検索＋コンパイル

一致するセグメントをインデックス化・検索・コンパイルして単一の再生可能なストリームにする:

```python
video.index_spoken_words(force=True)
results = video.search(query="your query", search_type=SearchType.semantic)
stream_url = results.compile()
print(stream_url)
```

## ヒント

- **一度インデックス化、何度も検索**: インデックス化はコストのかかる操作です。一度インデックス化されると、検索は高速になります。
- **インデックスタイプを組み合わせる**: 音声ワードとシーンの両方をインデックス化して、同じビデオですべての検索タイプを利用できるようにします。
- **クエリを洗練させる**: セマンティック検索は単一のキーワードよりも説明的な自然言語フレーズで最もうまく機能します。
- **精度にはキーワード検索を使用する**: 完全な用語マッチングが必要な場合は、キーワード検索でセマンティックなドリフトを避けます。
- **「No results found」を処理する**: `video.search()` は一致する結果がない場合に `InvalidRequestError` を発生させます。常に検索呼び出しをtry/exceptで囲み、`"No results found"` を空の結果セットとして扱います。
- **シーン検索のノイズをフィルタリングする**: セマンティックシーン検索は曖昧なクエリに対して低関連度の結果を返す可能性があります。ノイズをフィルタリングするには `score_threshold=0.3`（またはそれ以上）を使用します。
- **冪等なインデックス化**: 安全に再インデックス化するには `index_spoken_words(force=True)` を使用します。`index_scenes()` には `force` パラメーターがありません — try/exceptで囲み、`re.search(r"id\s+([a-f0-9]+)", str(e))` でエラーメッセージから既存の `scene_index_id` を抽出します。
