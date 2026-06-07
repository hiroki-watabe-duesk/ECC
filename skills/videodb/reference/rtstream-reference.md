# RTStreamリファレンス

RTStream操作のコードレベルの詳細。ワークフローガイドは[rtstream.md](rtstream.md)を参照。
使用ガイダンスとワークフロー選択については、[../SKILL.md](../SKILL.md)から始めること。

[docs.videodb.io](https://docs.videodb.io/pages/ingest/live-streams/realtime-apis.md)に基づく。

---

## CollectionのRTStreamメソッド

RTStreamを管理する `Collection` 上のメソッド:

| メソッド | 戻り値 | 説明 |
|--------|---------|-------------|
| `coll.connect_rtstream(url, name, ...)` | `RTStream` | RTSP/RTMP URLから新しいRTStreamを作成 |
| `coll.get_rtstream(id)` | `RTStream` | IDで既存のRTStreamを取得 |
| `coll.list_rtstreams(limit, offset, status, name, ordering)` | `List[RTStream]` | コレクション内のすべてのRTStreamをリスト |
| `coll.search(query, namespace="rtstream")` | `RTStreamSearchResult` | すべてのRTStreamを横断して検索 |

### RTStreamの接続

```python
import videodb

conn = videodb.connect()
coll = conn.get_collection()

rtstream = coll.connect_rtstream(
    url="rtmp://your-stream-server/live/stream-key",
    name="My Live Stream",
    media_types=["video"],  # または ["audio", "video"]
    sample_rate=30,         # 任意
    store=True,             # エクスポート用の録画ストレージを有効化
    enable_transcript=True, # 任意
    ws_connection_id=ws_id, # 任意、リアルタイムイベント用
)
```

### 既存のRTStreamを取得

```python
rtstream = coll.get_rtstream("rts-xxx")
```

### RTStreamのリスト表示

```python
rtstreams = coll.list_rtstreams(
    limit=10,
    offset=0,
    status="connected",  # 任意のフィルター
    name="meeting",      # 任意のフィルター
    ordering="-created_at",
)

for rts in rtstreams:
    print(f"{rts.id}: {rts.name} - {rts.status}")
```

### キャプチャセッションから

キャプチャセッションがアクティブになった後、RTStreamオブジェクトを取得する:

```python
session = conn.get_capture_session(session_id)

mics = session.get_rtstream("mic")
displays = session.get_rtstream("screen")
system_audios = session.get_rtstream("system_audio")
```

または `capture_session.active` WebSocketイベントから `rtstreams` データを使用する:

```python
for rts in rtstreams:
    rtstream = coll.get_rtstream(rts["rtstream_id"])
```

---

## RTStreamメソッド

| メソッド | 戻り値 | 説明 |
|--------|---------|-------------|
| `rtstream.start()` | `None` | インジェストを開始 |
| `rtstream.stop()` | `None` | インジェストを停止 |
| `rtstream.generate_stream(start, end)` | `str` | 録画済みセグメントをストリーム（Unixタイムスタンプ） |
| `rtstream.export(name=None)` | `RTStreamExportResult` | 永続的な動画にエクスポート |
| `rtstream.index_visuals(prompt, ...)` | `RTStreamSceneIndex` | AI分析でビジュアルインデックスを作成 |
| `rtstream.index_audio(prompt, ...)` | `RTStreamSceneIndex` | LLM要約でオーディオインデックスを作成 |
| `rtstream.list_scene_indexes()` | `List[RTStreamSceneIndex]` | ストリームのすべてのシーンインデックスをリスト |
| `rtstream.get_scene_index(index_id)` | `RTStreamSceneIndex` | 特定のシーンインデックスを取得 |
| `rtstream.search(query, ...)` | `RTStreamSearchResult` | インデックス済みコンテンツを検索 |
| `rtstream.start_transcript(ws_connection_id, engine)` | `dict` | ライブ文字起こしを開始 |
| `rtstream.get_transcript(page, page_size, start, end, since)` | `dict` | 文字起こしページを取得 |
| `rtstream.stop_transcript(engine)` | `dict` | 文字起こしを停止 |

---

## 開始と停止

```python
# インジェストを開始
rtstream.start()

# ... ストリームが録画中 ...

# インジェストを停止
rtstream.stop()
```

---

## ストリームの生成

録画済みコンテンツから再生ストリームを生成するには、Unixタイムスタンプ（秒オフセットではない）を使用する:

```python
import time

start_ts = time.time()
rtstream.start()

# しばらく録画する...
time.sleep(60)

end_ts = time.time()
rtstream.stop()

# 録画済みセグメントのストリームURLを生成する
stream_url = rtstream.generate_stream(start=start_ts, end=end_ts)
print(f"Recorded stream: {stream_url}")
```

---

## 動画へのエクスポート

録画済みストリームをコレクション内の永続的な動画にエクスポートする:

```python
export_result = rtstream.export(name="Meeting Recording 2024-01-15")

print(f"Video ID: {export_result.video_id}")
print(f"Stream URL: {export_result.stream_url}")
print(f"Player URL: {export_result.player_url}")
print(f"Duration: {export_result.duration}s")
```

### RTStreamExportResultのプロパティ

| プロパティ | 型 | 説明 |
|----------|------|-------------|
| `video_id` | `str` | エクスポートされた動画のID |
| `stream_url` | `str` | HLSストリームURL |
| `player_url` | `str` | ウェブプレイヤーURL |
| `name` | `str` | 動画名 |
| `duration` | `float` | 秒単位の再生時間 |

---

## AIパイプライン

AIパイプラインはライブストリームを処理し、WebSocket経由で結果を送信する。

### RTStream AIパイプラインメソッド

| メソッド | 戻り値 | 説明 |
|--------|---------|-------------|
| `rtstream.index_audio(prompt, batch_config, ...)` | `RTStreamSceneIndex` | LLM要約でオーディオインデックス作成を開始 |
| `rtstream.index_visuals(prompt, batch_config, ...)` | `RTStreamSceneIndex` | 画面コンテンツのビジュアルインデックス作成を開始 |

### オーディオインデックス作成

一定間隔でオーディオコンテンツのLLMサマリーを生成する:

```python
audio_index = rtstream.index_audio(
    prompt="Summarize what is being discussed",
    batch_config={"type": "word", "value": 50},
    model_name=None,       # 任意
    name="meeting_audio",  # 任意
    ws_connection_id=ws_id,
)
```

**オーディオ batch_config オプション:**

| タイプ | 値 | 説明 |
|------|-------|-------------|
| `"word"` | 数 | N単語ごとにセグメント化 |
| `"sentence"` | 数 | N文ごとにセグメント化 |
| `"time"` | 秒 | N秒ごとにセグメント化 |

例:
```python
{"type": "word", "value": 50}      # 50単語ごと
{"type": "sentence", "value": 5}   # 5文ごと
{"type": "time", "value": 30}      # 30秒ごと
```

結果は `audio_index` WebSocketチャンネルで受信される。

### ビジュアルインデックス作成

ビジュアルコンテンツのAI説明を生成する:

```python
scene_index = rtstream.index_visuals(
    prompt="Describe what is happening on screen",
    batch_config={"type": "time", "value": 2, "frame_count": 5},
    model_name="basic",
    name="screen_monitor",  # 任意
    ws_connection_id=ws_id,
)
```

**パラメーター:**

| パラメーター | 型 | 説明 |
|-----------|------|-------------|
| `prompt` | `str` | AIモデルへの指示（構造化JSON出力をサポート） |
| `batch_config` | `dict` | フレームサンプリングを制御する（以下参照） |
| `model_name` | `str` | モデルティア: `"mini"`、`"basic"`、`"pro"`、`"ultra"` |
| `name` | `str` | インデックスの名前（任意） |
| `ws_connection_id` | `str` | 結果を受信するためのWebSocket接続ID |

**ビジュアル batch_config:**

| キー | 型 | 説明 |
|-----|------|-------------|
| `type` | `str` | ビジュアルでサポートされるのは `"time"` のみ |
| `value` | `int` | ウィンドウサイズ（秒） |
| `frame_count` | `int` | ウィンドウごとに抽出するフレーム数 |

例: `{"type": "time", "value": 2, "frame_count": 5}` は2秒ごとに5フレームをサンプリングしてモデルに送信する。

**構造化JSON出力:**

構造化されたレスポンスのためにJSON形式を要求するプロンプトを使用する:

```python
scene_index = rtstream.index_visuals(
    prompt="""Analyze the screen and return a JSON object with:
{
  "app_name": "name of the active application",
  "activity": "what the user is doing",
  "ui_elements": ["list of visible UI elements"],
  "contains_text": true/false,
  "dominant_colors": ["list of main colors"]
}
Return only valid JSON.""",
    batch_config={"type": "time", "value": 3, "frame_count": 3},
    model_name="pro",
    ws_connection_id=ws_id,
)
```

結果は `scene_index` WebSocketチャンネルで受信される。

---

## バッチ設定の概要

| インデックスタイプ | `type` オプション | `value` | 追加キー |
|---------------|----------------|---------|------------|
| **オーディオ** | `"word"`、`"sentence"`、`"time"` | 単語数/文数/秒 | - |
| **ビジュアル** | `"time"` のみ | 秒 | `frame_count` |

例:
```python
# オーディオ: 50単語ごと
{"type": "word", "value": 50}

# オーディオ: 30秒ごと
{"type": "time", "value": 30}

# ビジュアル: 2秒ごとに5フレーム
{"type": "time", "value": 2, "frame_count": 5}
```

---

## 文字起こし

WebSocket経由のリアルタイム文字起こし:

```python
# ライブ文字起こしを開始
rtstream.start_transcript(
    ws_connection_id=ws_id,
    engine=None,  # 任意、デフォルトは "assemblyai"
)

# 文字起こしページを取得（任意のフィルター付き）
transcript = rtstream.get_transcript(
    page=1,
    page_size=100,
    start=None,   # 任意: 開始タイムスタンプフィルター
    end=None,     # 任意: 終了タイムスタンプフィルター
    since=None,   # 任意: ポーリング用、このタイムスタンプ以降の文字起こしを取得
    engine=None,
)

# 文字起こしを停止
rtstream.stop_transcript(engine=None)
```

文字起こし結果は `transcript` WebSocketチャンネルで受信される。

---

## RTStreamSceneIndex

`index_audio()` または `index_visuals()` を呼び出すと、メソッドは `RTStreamSceneIndex` オブジェクトを返す。このオブジェクトは実行中のインデックスを表し、シーンとアラートを管理するためのメソッドを提供する。

```python
# index_visualsはRTStreamSceneIndexを返す
scene_index = rtstream.index_visuals(
    prompt="Describe what is on screen",
    ws_connection_id=ws_id,
)

# index_audioもRTStreamSceneIndexを返す
audio_index = rtstream.index_audio(
    prompt="Summarize the discussion",
    ws_connection_id=ws_id,
)
```

### RTStreamSceneIndexのプロパティ

| プロパティ | 型 | 説明 |
|----------|------|-------------|
| `rtstream_index_id` | `str` | インデックスの一意のID |
| `rtstream_id` | `str` | 親RTStreamのID |
| `extraction_type` | `str` | 抽出タイプ（`time` または `transcript`） |
| `extraction_config` | `dict` | 抽出設定 |
| `prompt` | `str` | 分析に使用されたプロンプト |
| `name` | `str` | インデックスの名前 |
| `status` | `str` | ステータス（`connected`、`stopped`） |

### RTStreamSceneIndexのメソッド

| メソッド | 戻り値 | 説明 |
|--------|---------|-------------|
| `index.get_scenes(start, end, page, page_size)` | `dict` | インデックス済みシーンを取得 |
| `index.start()` | `None` | インデックスを開始／再開 |
| `index.stop()` | `None` | インデックスを停止 |
| `index.create_alert(event_id, callback_url, ws_connection_id)` | `str` | イベント検出のアラートを作成 |
| `index.list_alerts()` | `list` | このインデックスのすべてのアラートをリスト |
| `index.enable_alert(alert_id)` | `None` | アラートを有効化 |
| `index.disable_alert(alert_id)` | `None` | アラートを無効化 |

### シーンの取得

インデックスからインデックス済みシーンをポーリングする:

```python
result = scene_index.get_scenes(
    start=None,      # 任意: 開始タイムスタンプ
    end=None,        # 任意: 終了タイムスタンプ
    page=1,
    page_size=100,
)

for scene in result["scenes"]:
    print(f"[{scene['start']}-{scene['end']}] {scene['text']}")

if result["next_page"]:
    # 次のページを取得
    pass
```

### シーンインデックスの管理

```python
# ストリームのすべてのインデックスをリスト
indexes = rtstream.list_scene_indexes()

# IDで特定のインデックスを取得
scene_index = rtstream.get_scene_index(index_id)

# インデックスを停止
scene_index.stop()

# インデックスを再開
scene_index.start()
```

---

## イベント

イベントは再利用可能な検出ルール。一度作成して、アラート経由で任意のインデックスにアタッチする。

### 接続イベントメソッド

| メソッド | 戻り値 | 説明 |
|--------|---------|-------------|
| `conn.create_event(event_prompt, label)` | `str`（event_id） | 検出イベントを作成 |
| `conn.list_events()` | `list` | すべてのイベントをリスト |

### イベントの作成

```python
event_id = conn.create_event(
    event_prompt="User opened Slack application",
    label="slack_opened",
)
```

### イベントのリスト表示

```python
events = conn.list_events()
for event in events:
    print(f"{event['event_id']}: {event['label']}")
```

---

## アラート

アラートはイベントをインデックスに接続してリアルタイム通知を行う。AIがイベントの説明に一致するコンテンツを検出すると、アラートが送信される。

### アラートの作成

```python
# index_visualsからRTStreamSceneIndexを取得
scene_index = rtstream.index_visuals(
    prompt="Describe what application is open on screen",
    ws_connection_id=ws_id,
)

# インデックスにアラートを作成
alert_id = scene_index.create_alert(
    event_id=event_id,
    callback_url="https://your-backend.com/alerts",  # Webhook配信用
    ws_connection_id=ws_id,  # WebSocket配信用（任意）
)
```

**注意:** `callback_url` は必須。WebSocket配信のみを使用する場合は空文字列 `""` を渡す。

### アラートの管理

```python
# インデックスのすべてのアラートをリスト
alerts = scene_index.list_alerts()

# アラートの有効化／無効化
scene_index.disable_alert(alert_id)
scene_index.enable_alert(alert_id)
```

### アラート配信

| 方法 | レイテンシ | ユースケース |
|--------|---------|----------|
| WebSocket | リアルタイム | ダッシュボード、ライブUI |
| Webhook | 1秒未満 | サーバー間、自動化 |

### WebSocketアラートイベント

```json
{
  "channel": "alert",
  "rtstream_id": "rts-xxx",
  "data": {
    "event_label": "slack_opened",
    "timestamp": 1710000012340,
    "text": "User opened Slack application"
  }
}
```

### WebhookペイロードWebhook Payload

```json
{
  "event_id": "event-xxx",
  "label": "slack_opened",
  "confidence": 0.95,
  "explanation": "User opened the Slack application",
  "timestamp": "2024-01-15T10:30:45Z",
  "start_time": 1234.5,
  "end_time": 1238.0,
  "stream_url": "https://stream.videodb.io/v3/...",
  "player_url": "https://console.videodb.io/player?url=..."
}
```

---

## WebSocket統合

すべてのリアルタイムAI結果はWebSocket経由で配信される。`ws_connection_id` を以下に渡す:
- `rtstream.start_transcript()`
- `rtstream.index_audio()`
- `rtstream.index_visuals()`
- `scene_index.create_alert()`

### WebSocketチャンネル

| チャンネル | ソース | コンテンツ |
|---------|--------|---------|
| `transcript` | `start_transcript()` | リアルタイム音声テキスト変換 |
| `scene_index` | `index_visuals()` | ビジュアル分析結果 |
| `audio_index` | `index_audio()` | オーディオ分析結果 |
| `alert` | `create_alert()` | アラート通知 |

WebSocketイベント構造とws_listenerの使用方法については、[capture-reference.md](capture-reference.md)を参照。

---

## 完全なワークフロー

```python
import time
import videodb
from videodb.exceptions import InvalidRequestError

conn = videodb.connect()
coll = conn.get_collection()

# 1. 接続して録画を開始
rtstream = coll.connect_rtstream(
    url="rtmp://your-stream-server/live/stream-key",
    name="Weekly Standup",
    store=True,
)
rtstream.start()

# 2. ミーティングの終了まで録画
start_ts = time.time()
time.sleep(1800)  # 30分
end_ts = time.time()
rtstream.stop()

# キャプチャしたウィンドウの即時再生URLを生成
stream_url = rtstream.generate_stream(start=start_ts, end=end_ts)
print(f"Recorded stream: {stream_url}")

# 3. 永続的な動画にエクスポート
export_result = rtstream.export(name="Weekly Standup Recording")
print(f"Exported video: {export_result.video_id}")

# 4. 検索のためにエクスポートした動画をインデックス化
video = coll.get_video(export_result.video_id)
video.index_spoken_words(force=True)

# 5. アクションアイテムを検索
try:
    results = video.search("action items and next steps")
    stream_url = results.compile()
    print(f"Action items clip: {stream_url}")
except InvalidRequestError as exc:
    if "No results found" in str(exc):
        print("No action items were detected in the recording.")
    else:
        raise
```
