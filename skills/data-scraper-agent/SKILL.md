---
name: data-scraper-agent
description: あらゆる公開ソース（求人ボード、価格、ニュース、GitHub、スポーツ等）向けの完全自動化AIデータ収集エージェントを構築します。スケジュール実行でデータを収集し、無料LLM（Gemini Flash）でデータを強化し、結果をNotion/Sheets/Supabaseに保存して、ユーザーのフィードバックから学習します。GitHub Actions上で完全無料で動作します。ユーザーが何らかの公開データを自動的に監視・収集・追跡したい場合に使用してください。
origin: community
---

# データスクレイパーエージェント

あらゆる公開データソース向けの本番品質AIデータ収集エージェントを構築します。
スケジュール実行でデータを収集し、無料LLMで結果を強化し、データベースに保存し、時間とともに改善されます。

**スタック: Python · Gemini Flash（無料）· GitHub Actions（無料）· Notion / Sheets / Supabase**

## 起動タイミング

- ユーザーが公開ウェブサイトやAPIのスクレイピング・監視を希望している
- 「〇〇をチェックするボットを作って」「Xを監視して」「〇〇からデータを収集して」と言っている
- 求人・価格・ニュース・リポジトリ・スポーツスコア・イベント・リストの追跡を希望している
- ホスティング費用なしでデータ収集を自動化する方法を尋ねている
- 判断を積み重ねるごとに賢くなるエージェントを求めている

## コアコンセプト

### 3つのレイヤー

データスクレイパーエージェントはすべて3つのレイヤーで構成されます：

```
COLLECT → ENRICH → STORE
  │           │        │
Scraper    AI (LLM)  Database
runs on    scores/   Notion /
schedule   summarises Sheets /
           & classifies Supabase
```

### 無料スタック

| レイヤー | ツール | 理由 |
|---|---|---|
| **スクレイピング** | `requests` + `BeautifulSoup` | 無料、公開サイトの80%に対応 |
| **JSレンダリングサイト** | `playwright`（無料） | HTMLスクレイピングが失敗する場合 |
| **AI強化** | Gemini Flash（REST API経由） | 1日500リクエスト、100万トークン/日 — 無料 |
| **ストレージ** | Notion API | 無料枠あり、レビューに最適なUI |
| **スケジュール** | GitHub Actions cron | 公開リポジトリは無料 |
| **学習** | リポジトリ内JSONフィードバックファイル | インフラ不要、gitで永続化 |

### AIモデルフォールバックチェーン

クォータ枯渇時にGeminiモデルを自動フォールバックするようにエージェントを構築します：

```
gemini-2.0-flash-lite (30 RPM) →
gemini-2.0-flash (15 RPM) →
gemini-2.5-flash (10 RPM) →
gemini-flash-lite-latest (fallback)
```

### 効率のためのAPIバッチ呼び出し

LLMを1アイテムごとに1回呼び出してはいけません。必ずバッチ処理してください：

```python
# 悪い例: 33アイテムに33回のAPIコール
for item in items:
    result = call_ai(item)  # 33コール → レート制限に達する

# 良い例: 33アイテムに7回のAPIコール（バッチサイズ5）
for batch in chunks(items, size=5):
    results = call_ai(batch)  # 7コール → 無料枠内に収まる
```

---

## ワークフロー

### ステップ1: 目標の把握

ユーザーに確認する内容：

1. **収集対象:** 「どのデータソースですか？URL / API / RSS / 公開エンドポイント？」
2. **抽出フィールド:** 「何のフィールドが必要ですか？タイトル、価格、URL、日付、スコア？」
3. **保存先:** 「結果の保存先は？Notion、Google Sheets、Supabase、またはローカルファイル？」
4. **強化方法:** 「AIに各アイテムのスコアリング、要約、分類、マッチングをさせますか？」
5. **頻度:** 「どのくらいの頻度で実行しますか？毎時、毎日、毎週？」

プロンプトの参考例：
- 求人ボード → 履歴書との関連性スコアリング
- 商品価格 → 値下がり時にアラート
- GitHubリポジトリ → 新しいリリースの要約
- ニュースフィード → トピックとセンチメントで分類
- スポーツ結果 → トラッカーにスタッツを抽出
- イベントカレンダー → 興味でフィルタリング

---

### ステップ2: エージェントアーキテクチャの設計

ユーザー向けにこのディレクトリ構造を生成します：

```
my-agent/
├── config.yaml              # ユーザーがカスタマイズ（キーワード、フィルター、設定）
├── profile/
│   └── context.md           # AIが使用するユーザーコンテキスト（履歴書、興味、基準）
├── scraper/
│   ├── __init__.py
│   ├── main.py              # オーケストレーター: スクレイプ → 強化 → 保存
│   ├── filters.py           # ルールベースの事前フィルター（高速、AI前）
│   └── sources/
│       ├── __init__.py
│       └── source_name.py   # データソースごとに1ファイル
├── ai/
│   ├── __init__.py
│   ├── client.py            # モデルフォールバック付きGemini RESTクライアント
│   ├── pipeline.py          # バッチAI分析
│   ├── jd_fetcher.py        # URLからコンテンツ全文を取得（オプション）
│   └── memory.py            # ユーザーフィードバックから学習
├── storage/
│   ├── __init__.py
│   └── notion_sync.py       # または sheets_sync.py / supabase_sync.py
├── data/
│   └── feedback.json        # ユーザーの決定履歴（自動更新）
├── .env.example
├── setup.py                 # 初回DB/スキーマ作成
├── enrich_existing.py       # 古い行へのAIスコアバックフィル
├── requirements.txt
└── .github/
    └── workflows/
        └── scraper.yml      # GitHub Actionsスケジュール
```

---

### ステップ3: スクレイパーソースの構築

任意のデータソース用テンプレート：

```python
# scraper/sources/my_source.py
"""
[ソース名] — [どこ]から[何]をスクレイプする。
方式: [REST API / HTMLスクレイピング / RSSフィード]
"""
import requests
from bs4 import BeautifulSoup
from datetime import datetime, timezone
from scraper.filters import is_relevant

HEADERS = {
    "User-Agent": "Mozilla/5.0 (compatible; research-bot/1.0)",
}


def fetch() -> list[dict]:
    """
    一貫したスキーマのアイテムリストを返す。
    各アイテムには最低限: name, url, date_found が必要。
    """
    results = []

    # ---- REST APIソース ----
    resp = requests.get("https://api.example.com/items", headers=HEADERS, timeout=15)
    if resp.status_code == 200:
        for item in resp.json().get("results", []):
            if not is_relevant(item.get("title", "")):
                continue
            results.append(_normalise(item))

    return results


def _normalise(raw: dict) -> dict:
    """生のAPI/HTMLデータを標準スキーマに変換する。"""
    return {
        "name": raw.get("title", ""),
        "url": raw.get("link", ""),
        "source": "MySource",
        "date_found": datetime.now(timezone.utc).date().isoformat(),
        # ドメイン固有フィールドをここに追加
    }
```

**HTMLスクレイピングパターン:**
```python
soup = BeautifulSoup(resp.text, "lxml")
for card in soup.select("[class*='listing']"):
    title = card.select_one("h2, h3").get_text(strip=True)
    link = card.select_one("a")["href"]
    if not link.startswith("http"):
        link = f"https://example.com{link}"
```

**RSSフィードパターン:**
```python
import xml.etree.ElementTree as ET
root = ET.fromstring(resp.text)
for item in root.findall(".//item"):
    title = item.findtext("title", "")
    link = item.findtext("link", "")
```

---

### ステップ4: Gemini AIクライアントの構築

```python
# ai/client.py
import os, json, time, requests

_last_call = 0.0

MODEL_FALLBACK = [
    "gemini-2.0-flash-lite",
    "gemini-2.0-flash",
    "gemini-2.5-flash",
    "gemini-flash-lite-latest",
]


def generate(prompt: str, model: str = "", rate_limit: float = 7.0) -> dict:
    """429時に自動フォールバックしてGeminiを呼び出す。解析済みJSONまたは{}を返す。"""
    global _last_call

    api_key = os.environ.get("GEMINI_API_KEY", "")
    if not api_key:
        return {}

    elapsed = time.time() - _last_call
    if elapsed < rate_limit:
        time.sleep(rate_limit - elapsed)

    models = [model] + [m for m in MODEL_FALLBACK if m != model] if model else MODEL_FALLBACK
    _last_call = time.time()

    for m in models:
        url = f"https://generativelanguage.googleapis.com/v1beta/models/{m}:generateContent?key={api_key}"
        payload = {
            "contents": [{"parts": [{"text": prompt}]}],
            "generationConfig": {
                "responseMimeType": "application/json",
                "temperature": 0.3,
                "maxOutputTokens": 2048,
            },
        }
        try:
            resp = requests.post(url, json=payload, timeout=30)
            if resp.status_code == 200:
                return _parse(resp)
            if resp.status_code in (429, 404):
                time.sleep(1)
                continue
            return {}
        except requests.RequestException:
            return {}

    return {}


def _parse(resp) -> dict:
    try:
        text = (
            resp.json()
            .get("candidates", [{}])[0]
            .get("content", {})
            .get("parts", [{}])[0]
            .get("text", "")
            .strip()
        )
        if text.startswith("```"):
            text = text.split("\n", 1)[-1].rsplit("```", 1)[0]
        return json.loads(text)
    except (json.JSONDecodeError, KeyError):
        return {}
```

---

### ステップ5: AIパイプラインの構築（バッチ処理）

```python
# ai/pipeline.py
import json
import yaml
from pathlib import Path
from ai.client import generate

def analyse_batch(items: list[dict], context: str = "", preference_prompt: str = "") -> list[dict]:
    """アイテムをバッチで分析する。AIフィールドで強化されたアイテムを返す。"""
    config = yaml.safe_load((Path(__file__).parent.parent / "config.yaml").read_text())
    model = config.get("ai", {}).get("model", "gemini-2.5-flash")
    rate_limit = config.get("ai", {}).get("rate_limit_seconds", 7.0)
    min_score = config.get("ai", {}).get("min_score", 0)
    batch_size = config.get("ai", {}).get("batch_size", 5)

    batches = [items[i:i + batch_size] for i in range(0, len(items), batch_size)]
    print(f"  [AI] {len(items)} items → {len(batches)} API calls")

    enriched = []
    for i, batch in enumerate(batches):
        print(f"  [AI] Batch {i + 1}/{len(batches)}...")
        prompt = _build_prompt(batch, context, preference_prompt, config)
        result = generate(prompt, model=model, rate_limit=rate_limit)

        analyses = result.get("analyses", [])
        for j, item in enumerate(batch):
            ai = analyses[j] if j < len(analyses) else {}
            if ai:
                score = max(0, min(100, int(ai.get("score", 0))))
                if min_score and score < min_score:
                    continue
                enriched.append({**item, "ai_score": score, "ai_summary": ai.get("summary", ""), "ai_notes": ai.get("notes", "")})
            else:
                enriched.append(item)

    return enriched


def _build_prompt(batch, context, preference_prompt, config):
    priorities = config.get("priorities", [])
    items_text = "\n\n".join(
        f"Item {i+1}: {json.dumps({k: v for k, v in item.items() if not k.startswith('_')})}"
        for i, item in enumerate(batch)
    )

    return f"""Analyse these {len(batch)} items and return a JSON object.

# Items
{items_text}

# User Context
{context[:800] if context else "Not provided"}

# User Priorities
{chr(10).join(f"- {p}" for p in priorities)}

{preference_prompt}

# Instructions
Return: {{"analyses": [{{"score": <0-100>, "summary": "<2 sentences>", "notes": "<why this matches or doesn't>"}} for each item in order]}}
Be concise. Score 90+=excellent match, 70-89=good, 50-69=ok, <50=weak."""
```

---

### ステップ6: フィードバック学習システムの構築

```python
# ai/memory.py
"""ユーザーの判断から学習して将来のスコアリングを改善する。"""
import json
from pathlib import Path

FEEDBACK_PATH = Path(__file__).parent.parent / "data" / "feedback.json"


def load_feedback() -> dict:
    if FEEDBACK_PATH.exists():
        try:
            return json.loads(FEEDBACK_PATH.read_text())
        except (json.JSONDecodeError, OSError):
            pass
    return {"positive": [], "negative": []}


def save_feedback(fb: dict):
    FEEDBACK_PATH.parent.mkdir(parents=True, exist_ok=True)
    FEEDBACK_PATH.write_text(json.dumps(fb, indent=2))


def build_preference_prompt(feedback: dict, max_examples: int = 15) -> str:
    """フィードバック履歴をプロンプトのバイアスセクションに変換する。"""
    lines = []
    if feedback.get("positive"):
        lines.append("# Items the user LIKED (positive signal):")
        for e in feedback["positive"][-max_examples:]:
            lines.append(f"- {e}")
    if feedback.get("negative"):
        lines.append("\n# Items the user SKIPPED/REJECTED (negative signal):")
        for e in feedback["negative"][-max_examples:]:
            lines.append(f"- {e}")
    if lines:
        lines.append("\nUse these patterns to bias scoring on new items.")
    return "\n".join(lines)
```

**ストレージレイヤーとの統合:** 各実行後、ポジティブ/ネガティブステータスのアイテムをDBから取得し、抽出したパターンで `save_feedback()` を呼び出します。

---

### ステップ7: ストレージの構築（Notionの例）

```python
# storage/notion_sync.py
import os
from notion_client import Client
from notion_client.errors import APIResponseError

_client = None

def get_client():
    global _client
    if _client is None:
        _client = Client(auth=os.environ["NOTION_TOKEN"])
    return _client

def get_existing_urls(db_id: str) -> set[str]:
    """保存済みURLをすべて取得 — 重複排除に使用。"""
    client, seen, cursor = get_client(), set(), None
    while True:
        resp = client.databases.query(database_id=db_id, page_size=100, **{"start_cursor": cursor} if cursor else {})
        for page in resp["results"]:
            url = page["properties"].get("URL", {}).get("url", "")
            if url: seen.add(url)
        if not resp["has_more"]: break
        cursor = resp["next_cursor"]
    return seen

def push_item(db_id: str, item: dict) -> bool:
    """1つのアイテムをNotionにプッシュする。成功した場合はTrueを返す。"""
    props = {
        "Name": {"title": [{"text": {"content": item.get("name", "")[:100]}}]},
        "URL": {"url": item.get("url")},
        "Source": {"select": {"name": item.get("source", "Unknown")}},
        "Date Found": {"date": {"start": item.get("date_found")}},
        "Status": {"select": {"name": "New"}},
    }
    # AIフィールド
    if item.get("ai_score") is not None:
        props["AI Score"] = {"number": item["ai_score"]}
    if item.get("ai_summary"):
        props["Summary"] = {"rich_text": [{"text": {"content": item["ai_summary"][:2000]}}]}
    if item.get("ai_notes"):
        props["Notes"] = {"rich_text": [{"text": {"content": item["ai_notes"][:2000]}}]}

    try:
        get_client().pages.create(parent={"database_id": db_id}, properties=props)
        return True
    except APIResponseError as e:
        print(f"[notion] Push failed: {e}")
        return False

def sync(db_id: str, items: list[dict]) -> tuple[int, int]:
    existing = get_existing_urls(db_id)
    added = skipped = 0
    for item in items:
        if item.get("url") in existing:
            skipped += 1; continue
        if push_item(db_id, item):
            added += 1; existing.add(item["url"])
        else:
            skipped += 1
    return added, skipped
```

---

### ステップ8: main.pyでのオーケストレーション

```python
# scraper/main.py
import os, sys, yaml
from pathlib import Path
from dotenv import load_dotenv

load_dotenv()

from scraper.sources import my_source          # ソースを追加

# 注意: この例ではNotionを使用しています。storage.providerが"sheets"または"supabase"の場合、
# このimportをstorage.sheets_syncまたはstorage.supabase_syncに置き換え、
# 環境変数とsync()の呼び出しを適宜更新してください。
from storage.notion_sync import sync

SOURCES = [
    ("My Source", my_source.fetch),
]

def ai_enabled():
    return bool(os.environ.get("GEMINI_API_KEY"))

def main():
    config = yaml.safe_load((Path(__file__).parent.parent / "config.yaml").read_text())
    provider = config.get("storage", {}).get("provider", "notion")

    # プロバイダーに基づいて環境変数からストレージ識別子を解決
    if provider == "notion":
        db_id = os.environ.get("NOTION_DATABASE_ID")
        if not db_id:
            print("ERROR: NOTION_DATABASE_ID not set"); sys.exit(1)
    else:
        # sheets (SHEET_ID) や supabase (SUPABASE_TABLE) 等はここで拡張
        print(f"ERROR: provider '{provider}' not yet wired in main.py"); sys.exit(1)

    config = yaml.safe_load((Path(__file__).parent.parent / "config.yaml").read_text())
    all_items = []

    for name, fetch_fn in SOURCES:
        try:
            items = fetch_fn()
            print(f"[{name}] {len(items)} items")
            all_items.extend(items)
        except Exception as e:
            print(f"[{name}] FAILED: {e}")

    # URLで重複排除
    seen, deduped = set(), []
    for item in all_items:
        if (url := item.get("url", "")) and url not in seen:
            seen.add(url); deduped.append(item)

    print(f"Unique items: {len(deduped)}")

    if ai_enabled() and deduped:
        from ai.memory import load_feedback, build_preference_prompt
        from ai.pipeline import analyse_batch

        # load_feedback() はフィードバック同期スクリプトが書き込む data/feedback.json を読み込む。
        # 最新の状態を保つため、ストレージプロバイダーからポジティブ/ネガティブ
        # ステータスのアイテムを取得してsave_feedback()を呼び出す
        # 別のfeedback_sync.pyを実装してください。
        feedback = load_feedback()
        preference = build_preference_prompt(feedback)
        context_path = Path(__file__).parent.parent / "profile" / "context.md"
        context = context_path.read_text() if context_path.exists() else ""
        deduped = analyse_batch(deduped, context=context, preference_prompt=preference)
    else:
        print("[AI] Skipped — GEMINI_API_KEY not set")

    added, skipped = sync(db_id, deduped)
    print(f"Done — {added} new, {skipped} existing")

if __name__ == "__main__":
    main()
```

---

### ステップ9: GitHub Actionsワークフロー

```yaml
# .github/workflows/scraper.yml
name: Data Scraper Agent

on:
  schedule:
    - cron: "0 */3 * * *"  # 3時間ごと — 必要に応じて調整
  workflow_dispatch:        # 手動トリガーを許可

permissions:
  contents: write   # フィードバック履歴のコミットステップに必要

jobs:
  scrape:
    runs-on: ubuntu-latest
    timeout-minutes: 20

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"
          cache: "pip"

      - run: pip install -r requirements.txt

      # requirements.txtでPlaywrightが有効な場合はコメントを外す
      # - name: Install Playwright browsers
      #   run: python -m playwright install chromium --with-deps

      - name: Run agent
        env:
          NOTION_TOKEN: ${{ secrets.NOTION_TOKEN }}
          NOTION_DATABASE_ID: ${{ secrets.NOTION_DATABASE_ID }}
          GEMINI_API_KEY: ${{ secrets.GEMINI_API_KEY }}
        run: python -m scraper.main

      - name: Commit feedback history
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add data/feedback.json || true
          git diff --cached --quiet || git commit -m "chore: update feedback history"
          git push
```

---

### ステップ10: config.yamlテンプレート

```yaml
# このファイルをカスタマイズ — コード変更不要

# 収集対象（AI前の事前フィルター）
filters:
  required_keywords: []      # アイテムに少なくとも1つ含まれている必要がある
  blocked_keywords: []       # アイテムに含まれてはいけない

# 優先事項 — AIがスコアリングに使用
priorities:
  - "優先事項の例1"
  - "優先事項の例2"

# ストレージ
storage:
  provider: "notion"         # notion | sheets | supabase | sqlite

# フィードバック学習
feedback:
  positive_statuses: ["Saved", "Applied", "Interested"]
  negative_statuses: ["Skip", "Rejected", "Not relevant"]

# AI設定
ai:
  enabled: true
  model: "gemini-2.5-flash"
  min_score: 0               # このスコア未満のアイテムをフィルターアウト
  rate_limit_seconds: 7      # APIコール間の秒数
  batch_size: 5              # APIコールごとのアイテム数
```

---

## よくあるスクレイピングパターン

### パターン1: REST API（最も簡単）
```python
resp = requests.get(url, params={"q": query}, headers=HEADERS, timeout=15)
items = resp.json().get("results", [])
```

### パターン2: HTMLスクレイピング
```python
soup = BeautifulSoup(resp.text, "lxml")
for card in soup.select(".listing-card"):
    title = card.select_one("h2").get_text(strip=True)
    href = card.select_one("a")["href"]
```

### パターン3: RSSフィード
```python
import xml.etree.ElementTree as ET
root = ET.fromstring(resp.text)
for item in root.findall(".//item"):
    title = item.findtext("title", "")
    link = item.findtext("link", "")
    pub_date = item.findtext("pubDate", "")
```

### パターン4: ページネーション付きAPI
```python
page = 1
while True:
    resp = requests.get(url, params={"page": page, "limit": 50}, timeout=15)
    data = resp.json()
    items = data.get("results", [])
    if not items:
        break
    for item in items:
        results.append(_normalise(item))
    if not data.get("has_more"):
        break
    page += 1
```

### パターン5: JSレンダリングページ（Playwright）
```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.launch()
    page = browser.new_page()
    page.goto(url)
    page.wait_for_selector(".listing")
    html = page.content()
    browser.close()

soup = BeautifulSoup(html, "lxml")
```

---

## 避けるべきアンチパターン

| アンチパターン | 問題点 | 修正方法 |
|---|---|---|
| アイテムごとに1回LLMを呼び出す | レート制限に即座に達する | 1回の呼び出しで5アイテムをバッチ処理 |
| コードにキーワードをハードコード | 再利用不可 | すべての設定を `config.yaml` に移動 |
| レート制限なしでスクレイピング | IPアドレス禁止 | リクエスト間に `time.sleep(1)` を追加 |
| コードにシークレットを保存 | セキュリティリスク | 必ず `.env` とGitHub Secretsを使用 |
| 重複排除なし | 重複行が蓄積される | プッシュ前に必ずURLを確認 |
| `robots.txt` を無視 | 法的・倫理的リスク | クロールルールを尊重し、可能な限り公開APIを使用 |
| `requests` でJSレンダリングサイトをスクレイプ | 空のレスポンス | Playwrightを使用するか基盤となるAPIを探す |
| `maxOutputTokens` が低すぎる | JSONが途切れ、パースエラー | バッチレスポンスには2048以上を使用 |

---

## 無料枠の制限参考値

| サービス | 無料枠 | 典型的な使用量 |
|---|---|---|
| Gemini Flash Lite | 30 RPM、1500 RPD | 3時間ごとで約56リクエスト/日 |
| Gemini 2.0 Flash | 15 RPM、1500 RPD | 優れたフォールバック |
| Gemini 2.5 Flash | 10 RPM、500 RPD | 控えめに使用 |
| GitHub Actions | 無制限（公開リポジトリ） | 約20分/日 |
| Notion API | 無制限 | 約200書き込み/日 |
| Supabase | 500MB DB、2GB転送 | ほとんどのエージェントで十分 |
| Google Sheets API | 300リクエスト/分 | 小規模エージェントに対応 |

---

## requirementsテンプレート

```
requests==2.31.0
beautifulsoup4==4.12.3
lxml==5.1.0
python-dotenv==1.0.1
pyyaml==6.0.2
notion-client==2.2.1   # Notionを使用する場合
# playwright==1.40.0   # JSレンダリングサイトの場合はコメントを外す
```

---

## 品質チェックリスト

エージェントを完成とする前に確認：

- [ ] `config.yaml` がユーザー向け設定をすべて制御 — ハードコード値なし
- [ ] `profile/context.md` にAIマッチング用のユーザー固有コンテキストが含まれている
- [ ] ストレージプッシュ前にURLで重複排除している
- [ ] GeminiクライアントにモデルフォールバックチェーンがあるP（4モデル）
- [ ] バッチサイズが1回のAPIコールあたり5アイテム以下
- [ ] `maxOutputTokens` が2048以上
- [ ] `.env` が `.gitignore` に含まれている
- [ ] オンボーディング用に `.env.example` が提供されている
- [ ] `setup.py` が初回実行時にDBスキーマを作成する
- [ ] `enrich_existing.py` が古い行のAIスコアをバックフィルする
- [ ] GitHub Actionsワークフローが各実行後に `feedback.json` をコミットする
- [ ] READMEに5分以内のセットアップ、必要なシークレット、カスタマイズ方法が記載されている

---

## 実世界の使用例

```
「Hacker NewsのAIスタートアップ資金調達ニュースを監視するエージェントを作って」
「3つのECサイトの商品価格をスクレイプして、下がったらアラートを出して」
「'llm'や'agents'でタグ付けされた新しいGitHubリポジトリを追跡して、それぞれを要約して」
「LinkedInとCutshortからChief of Staffの求人リストを収集してNotionに入れて」
「自社についてのRedditの投稿を監視して、センチメントを分類して」
「興味のあるトピックに関するarXivの新しい学術論文を毎日スクレイプして」
「スポーツの試合結果を追跡して、Google Sheetsに順位表を維持して」
「不動産リストのウォッチャーを作って — ₹1 Cr以下の新しい物件にアラートを出して」
```

---

## リファレンス実装

このアーキテクチャで構築された完全に動作するエージェントは、4つ以上のソースをスクレイプし、
Geminiのコールをバッチ処理し、Notionに保存されたApplied/Rejectedの判断から学習し、
GitHub Actions上で完全無料で動作します。ステップ1〜9に従って独自のものを構築してください。
