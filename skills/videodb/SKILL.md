---
name: videodb
description: 映像・音声の知覚・記憶・アクション。知覚 - ローカルファイル・URL・RTSP/ライブフィード・デスクトップのライブ録画から取り込み、リアルタイムコンテキストと再生可能なストリームリンクを返す。理解 - フレーム抽出、視覚的/意味的/時間的インデックスの構築、タイムスタンプと自動クリップ付きのモーメント検索。アクション - トランスコードと正規化（コーデック、fps、解像度、アスペクト比）、タイムライン編集（字幕、テキスト/画像オーバーレイ、ブランディング、音声オーバーレイ、吹き替え、翻訳）、メディアアセットの生成（画像、音声、動画）、ライブストリームまたはデスクトップキャプチャからのイベントのリアルタイムアラートの作成。
origin: ECC
allowed-tools: Read Grep Glob Bash(python:*)
argument-hint: "[task description]"
---

# VideoDB スキル

**映像・ライブストリーム・デスクトップセッションのための知覚＋記憶＋アクション。**

## 使用するタイミング

### デスクトップ知覚
- **スクリーン、マイク、システム音声**をキャプチャする**デスクトップセッション**の開始・停止
- **ライブコンテキスト**のストリーミングと**エピソード的なセッションメモリ**の保存
- 画面上で話されていることと起きていることに対する**リアルタイムアラート/トリガー**の実行
- **セッションサマリー**、検索可能なタイムライン、**再生可能なエビデンスリンク**の生成

### 動画の取り込みとストリーミング
- **ファイルまたはURL**を取り込み、**再生可能なウェブストリームリンク**を返す
- トランスコード/正規化: **コーデック、ビットレート、fps、解像度、アスペクト比**

### インデックス＋検索（タイムスタンプ＋エビデンス）
- **視覚的**、**音声**、**キーワード**インデックスの構築
- **タイムスタンプ**と**再生可能なエビデンス**付きで正確なモーメントを検索・返却
- 検索結果から**クリップ**を自動作成

### タイムライン編集＋生成
- 字幕: **生成**、**翻訳**、**バーンイン**
- オーバーレイ: **テキスト/画像/ブランディング**、モーションキャプション
- 音声: **BGM**、**ボイスオーバー**、**吹き替え**
- **タイムライン操作**によるプログラマティックな合成とエクスポート

### ライブストリーム（RTSP）＋モニタリング
- **RTSP/ライブフィード**への接続
- **リアルタイムの視覚・音声理解**の実行と監視ワークフロー向けの**イベント/アラート**の発行

## 動作の仕組み

### 一般的な入力
- ローカル**ファイルパス**、公開**URL**、または**RTSP URL**
- デスクトップキャプチャリクエスト: **セッションの開始/停止/サマリー**
- 目的の操作: 理解のためのコンテキスト取得、トランスコード仕様、インデックス仕様、検索クエリ、クリップ範囲、タイムライン編集、アラートルール

### 一般的な出力
- **ストリームURL**
- **タイムスタンプ**と**エビデンスリンク**付きの検索結果
- 生成されたアセット: 字幕、音声、画像、クリップ
- ライブストリーム向けの**イベント/アラートペイロード**
- デスクトップの**セッションサマリー**とメモリエントリ

### Pythonコードの実行

VideoDB コードを実行する前に、プロジェクトディレクトリに移動して環境変数を読み込んでください:

```python
from dotenv import load_dotenv
load_dotenv(".env")

import videodb
conn = videodb.connect()
```

これにより以下から `VIDEO_DB_API_KEY` が読み込まれます:
1. 環境変数（すでにエクスポートされている場合）
2. カレントディレクトリのプロジェクトの `.env` ファイル

キーが見つからない場合、`videodb.connect()` は自動的に `AuthenticationError` を発生させます。

短いインラインコマンドで済む場合はスクリプトファイルを作成しないでください。

インラインPython（`python -c "..."`）を記述する際は、適切にフォーマットされたコードを使用してください。文は必ずセミコロンで区切り、読みやすく保ってください。3文を超える場合は、代わりにヒアドキュメントを使用してください:

```bash
python << 'EOF'
from dotenv import load_dotenv
load_dotenv(".env")

import videodb
conn = videodb.connect()
coll = conn.get_collection()
print(f"Videos: {len(coll.get_videos())}")
EOF
```

### セットアップ

ユーザーが「videodb のセットアップ」などと求めた場合:

### 1. SDKのインストール

```bash
pip install "videodb[capture]" python-dotenv
```

Linux で `videodb[capture]` が失敗した場合は、captureエクストラなしでインストールしてください:

```bash
pip install videodb python-dotenv
```

### 2. APIキーの設定

ユーザーは **いずれかの** 方法で `VIDEO_DB_API_KEY` を設定する必要があります:

- **ターミナルでのエクスポート**（Claudeの起動前）: `export VIDEO_DB_API_KEY=your-key`
- **プロジェクトの `.env` ファイル**: プロジェクトの `.env` ファイルに `VIDEO_DB_API_KEY=your-key` を保存

無料APIキーは [console.videodb.io](https://console.videodb.io) で取得できます（50回の無料アップロード、クレジットカード不要）。

APIキーを自分で読み取り、書き込み、または処理**しないでください**。常にユーザーが設定するようにしてください。

### クイックリファレンス

### メディアのアップロード

```python
# URL
video = coll.upload(url="https://example.com/video.mp4")

# YouTube
video = coll.upload(url="https://www.youtube.com/watch?v=VIDEO_ID")

# ローカルファイル
video = coll.upload(file_path="/path/to/video.mp4")
```

### トランスクリプト＋字幕

```python
# force=True を指定すると、動画がすでにインデックス済みの場合のエラーをスキップ
video.index_spoken_words(force=True)
text = video.get_transcript_text()
stream_url = video.add_subtitle()
```

### 動画内の検索

```python
from videodb.exceptions import InvalidRequestError

video.index_spoken_words(force=True)

# search() は結果が見つからない場合に InvalidRequestError を発生させます。
# 常に try/except でラップし、"No results found" は空として扱ってください。
try:
    results = video.search("product demo")
    shots = results.get_shots()
    stream_url = results.compile()
except InvalidRequestError as e:
    if "No results found" in str(e):
        shots = []
    else:
        raise
```

### シーン検索

```python
import re
from videodb import SearchType, IndexType, SceneExtractionType
from videodb.exceptions import InvalidRequestError

# index_scenes() には force パラメータがありません — シーンインデックスが
# すでに存在する場合はエラーが発生します。エラーから既存のインデックスIDを取得してください。
try:
    scene_index_id = video.index_scenes(
        extraction_type=SceneExtractionType.shot_based,
        prompt="Describe the visual content in this scene.",
    )
except Exception as e:
    match = re.search(r"id\s+([a-f0-9]+)", str(e))
    if match:
        scene_index_id = match.group(1)
    else:
        raise

# score_threshold を使って低関連性のノイズをフィルタリング（推奨値: 0.3以上）
try:
    results = video.search(
        query="person writing on a whiteboard",
        search_type=SearchType.semantic,
        index_type=IndexType.scene,
        scene_index_id=scene_index_id,
        score_threshold=0.3,
    )
    shots = results.get_shots()
    stream_url = results.compile()
except InvalidRequestError as e:
    if "No results found" in str(e):
        shots = []
    else:
        raise
```

### タイムライン編集

**重要:** タイムラインを構築する前に必ずタイムスタンプを検証してください:
- `start` は >= 0 でなければなりません（負の値は静かに受け入れられますが、壊れた出力を生成します）
- `start` は `end` より小さくなければなりません
- `end` は `video.length` 以下でなければなりません

```python
from videodb.timeline import Timeline
from videodb.asset import VideoAsset, TextAsset, TextStyle

timeline = Timeline(conn)
timeline.add_inline(VideoAsset(asset_id=video.id, start=10, end=30))
timeline.add_overlay(0, TextAsset(text="The End", duration=3, style=TextStyle(fontsize=36)))
stream_url = timeline.generate_stream()
```

### 動画のトランスコード（解像度/品質変更）

```python
from videodb import TranscodeMode, VideoConfig, AudioConfig

# 解像度、品質、またはアスペクト比をサーバーサイドで変更
job_id = conn.transcode(
    source="https://example.com/video.mp4",
    callback_url="https://example.com/webhook",
    mode=TranscodeMode.economy,
    video_config=VideoConfig(resolution=720, quality=23, aspect_ratio="16:9"),
    audio_config=AudioConfig(mute=False),
)
```

### アスペクト比のリフレーム（ソーシャルプラットフォーム向け）

**注意:** `reframe()` はサーバーサイドの処理が遅い操作です。長い動画では数分かかり、タイムアウトする場合があります。ベストプラクティス:
- 可能な限り `start`/`end` を使って短いセグメントに限定する
- フル長の動画では非同期処理のために `callback_url` を渡す
- まず `Timeline` で動画をトリミングし、その短い結果をリフレームする

```python
from videodb import ReframeMode

# 常に短いセグメントのリフレームを優先:
reframed = video.reframe(start=0, end=60, target="vertical", mode=ReframeMode.smart)

# フル長の動画の非同期リフレーム（Noneを返し、結果はWebhookで取得）:
video.reframe(target="vertical", callback_url="https://example.com/webhook")

# プリセット: "vertical" (9:16)、"square" (1:1)、"landscape" (16:9)
reframed = video.reframe(start=0, end=60, target="square")

# カスタムサイズ
reframed = video.reframe(start=0, end=60, target={"width": 1280, "height": 720})
```

### 生成メディア

```python
image = coll.generate_image(
    prompt="a sunset over mountains",
    aspect_ratio="16:9",
)
```

## エラー処理

```python
from videodb.exceptions import AuthenticationError, InvalidRequestError

try:
    conn = videodb.connect()
except AuthenticationError:
    print("Check your VIDEO_DB_API_KEY")

try:
    video = coll.upload(url="https://example.com/video.mp4")
except InvalidRequestError as e:
    print(f"Upload failed: {e}")
```

### よくある落とし穴

| シナリオ | エラーメッセージ | 解決策 |
|----------|--------------|----------|
| すでにインデックス済みの動画をインデックス | `Spoken word index for video already exists` | `video.index_spoken_words(force=True)` を使ってインデックス済みの場合はスキップ |
| シーンインデックスがすでに存在 | `Scene index with id XXXX already exists` | `re.search(r"id\s+([a-f0-9]+)", str(e))` でエラーから既存の `scene_index_id` を取得 |
| 検索結果が見つからない | `InvalidRequestError: No results found` | 例外をキャッチして空の結果として扱う（`shots = []`） |
| リフレームがタイムアウト | 長い動画で無限にブロック | `start`/`end` でセグメントを限定するか、非同期処理のために `callback_url` を渡す |
| タイムライン上の負のタイムスタンプ | 壊れたストリームを静かに生成 | `VideoAsset` を作成する前に必ず `start >= 0` を検証する |
| `generate_video()` / `create_collection()` が失敗 | `Operation not allowed` または `maximum limit` | プラン制限の機能 — プランの制限についてユーザーに伝える |

## 使用例

### 標準的なプロンプト
- 「デスクトップキャプチャを開始し、パスワードフィールドが表示されたらアラートを送信してください。」
- 「セッションを録画し、終了時に実行可能なサマリーを作成してください。」
- 「このファイルを取り込み、再生可能なストリームリンクを返してください。」
- 「このフォルダをインデックスし、人物が映っているすべてのシーンをタイムスタンプ付きで見つけてください。」
- 「字幕を生成し、バーンインして、軽いBGMを追加してください。」
- 「このRTSP URLに接続し、人物がゾーンに入ったらアラートを送信してください。」

### スクリーン録画（デスクトップキャプチャ）

録画セッション中のWebSocketイベントのキャプチャには `ws_listener.py` を使用してください。デスクトップキャプチャは **macOSのみ** に対応しています。

#### クイックスタート

1. **状態ディレクトリの選択**: `STATE_DIR="${VIDEODB_EVENTS_DIR:-$HOME/.local/state/videodb}"`
2. **リスナーの開始**: `VIDEODB_EVENTS_DIR="$STATE_DIR" python scripts/ws_listener.py --clear "$STATE_DIR" &`
3. **WebSocket IDの取得**: `cat "$STATE_DIR/videodb_ws_id"`
4. **キャプチャコードの実行**（完全なワークフローは reference/capture.md を参照）
5. **イベントの書き込み先**: `$STATE_DIR/videodb_events.jsonl`

新しいキャプチャ実行を開始する際は必ず `--clear` を使用してください。古いトランスクリプトや視覚イベントが新しいセッションに混入しないようにするためです。

#### イベントのクエリ

```python
import json
import os
import time
from pathlib import Path

events_dir = Path(os.environ.get("VIDEODB_EVENTS_DIR", Path.home() / ".local" / "state" / "videodb"))
events_file = events_dir / "videodb_events.jsonl"
events = []

if events_file.exists():
    with events_file.open(encoding="utf-8") as handle:
        for line in handle:
            try:
                events.append(json.loads(line))
            except json.JSONDecodeError:
                continue

transcripts = [e["data"]["text"] for e in events if e.get("channel") == "transcript"]
cutoff = time.time() - 300
recent_visual = [
    e for e in events
    if e.get("channel") == "visual_index" and e["unix_ts"] > cutoff
]
```

## 追加ドキュメント

リファレンスドキュメントはこの SKILL.md ファイルに隣接する `reference/` ディレクトリにあります。必要に応じて Glob ツールで場所を確認してください。

- [reference/api-reference.md](reference/api-reference.md) - 完全な VideoDB Python SDK APIリファレンス
- [reference/search.md](reference/search.md) - 動画検索の詳細ガイド（音声およびシーンベース）
- [reference/editor.md](reference/editor.md) - タイムライン編集、アセット、コンポジション
- [reference/streaming.md](reference/streaming.md) - HLSストリーミングとインスタント再生
- [reference/generative.md](reference/generative.md) - AI駆動のメディア生成（画像、動画、音声）
- [reference/rtstream.md](reference/rtstream.md) - ライブストリーム取り込みワークフロー（RTSP/RTMP）
- [reference/rtstream-reference.md](reference/rtstream-reference.md) - RTStream SDKメソッドとAIパイプライン
- [reference/capture.md](reference/capture.md) - デスクトップキャプチャワークフロー
- [reference/capture-reference.md](reference/capture-reference.md) - キャプチャSDKとWebSocketイベント
- [reference/use-cases.md](reference/use-cases.md) - 一般的な動画処理パターンと使用例

VideoDBが操作をサポートしている場合は **ffmpeg、moviepy、またはローカルエンコーディングツールを使用しないでください**。以下はすべてVideoDBがサーバーサイドで処理します — トリミング、クリップの結合、音声やBGMのオーバーレイ、字幕の追加、テキスト/画像オーバーレイ、トランスコード、解像度変更、アスペクト比変換、プラットフォーム要件に合わせたリサイズ、トランスクリプション、メディア生成。reference/editor.md の制限事項に記載されている操作（トランジション、速度変更、クロップ/ズーム、カラーグレーディング、ボリュームミキシング）のみローカルツールにフォールバックしてください。

### どちらを使うか

| 問題 | VideoDBの解決策 |
|---------|-----------------|
| プラットフォームが動画のアスペクト比や解像度を拒否 | `video.reframe()` または `VideoConfig` 付きの `conn.transcode()` |
| Twitter/Instagram/TikTok向けに動画をリサイズ | `video.reframe(target="vertical")` または `target="square"` |
| 解像度を変更（例: 1080p → 720p） | `VideoConfig(resolution=720)` 付きの `conn.transcode()` |
| 動画に音声/BGMをオーバーレイ | `Timeline` 上の `AudioAsset` |
| 字幕を追加 | `video.add_subtitle()` または `CaptionAsset` |
| クリップを結合/トリミング | `Timeline` 上の `VideoAsset` |
| ボイスオーバー、BGM、効果音を生成 | `coll.generate_voice()`、`generate_music()`、`generate_sound_effect()` |

## 出典

このスキルのリファレンス資料は `skills/videodb/reference/` 配下にローカルでベンダリングされています。
実行時に外部リポジトリのリンクを辿るのではなく、上記のローカルコピーを使用してください。
