# Grid-twitter-pulse
# Grid Twitter Pulse

A small, lightweight Python project that monitors Twitter (X) for specified queries and builds a daily "pulse" of new tweets. It saves seen tweets, posts alerts to Telegram, and can render a simple `grid.html` showing the latest tweets.

---

## File structure

```
grid-twitter-pulse/
├─ README.md
├─ requirements.txt
├─ .gitignore
├─ .env.example
├─ src/
│  ├─ main.py
│  ├─ notifier.py
│  ├─ storage.py
│  └─ renderer.py
└─ grid.html (generated)
```

---

## README.md

````markdown
# Grid Twitter Pulse

Grid Twitter Pulse is a lightweight Twitter/X monitoring tool that:

- Polls Twitter for your chosen search queries.
- Stores seen tweet IDs (so you only get new alerts).
- Sends new tweet alerts to a Telegram chat (or prints to console).
- Generates a simple `grid.html` showing recent tweets in a responsive grid.

> Note: This project uses the Twitter API v2 (Bearer token) via `tweepy.Client`. Make sure your developer account has the required access.

## Features

- Configurable search queries
- Deduplication (stores seen tweet IDs locally)
- Telegram alerts (optional)
- HTML grid renderer for quick visual pulse

## Requirements

- Python 3.10+
- Twitter Developer Bearer token (v2)
- (Optional) Telegram bot token and chat ID

## Installation

1. Clone the repo:

```bash
git clone https://github.com/YOUR_USERNAME/grid-twitter-pulse.git
cd grid-twitter-pulse
````

2. Create and activate a virtual environment:

```bash
python -m venv venv
source venv/bin/activate  # macOS / Linux
venv\Scripts\activate     # Windows
```

3. Install dependencies:

```bash
pip install -r requirements.txt
```

4. Copy `.env.example` to `.env` and fill in your tokens:

```bash
cp .env.example .env
# then edit .env
```

## .env (example)

```
TWITTER_BEARER_TOKEN=YOUR_TWITTER_BEARER_TOKEN
TELEGRAM_BOT_TOKEN=YOUR_TELEGRAM_BOT_TOKEN  # optional
TELEGRAM_CHAT_ID=YOUR_TELEGRAM_CHAT_ID      # optional
QUERIES=base protocol,solana nft,web3 game
POLL_INTERVAL_SECONDS=300
MAX_TWEETS_PER_QUERY=5
```

## Usage

Run the monitor:

```bash
python src/main.py
```

This will poll Twitter for the queries listed in `QUERIES`, post new tweets to Telegram (if configured), save seen tweets to `seen_tweets.json`, and update `grid.html` with the latest tweets.

## Deploying

You can deploy this as a small long-running worker on platforms like:

* Replit / Deta / Railway
* Heroku (worker dyno)
* Docker container on any VPS

For a cron-style approach, run `python src/main.py` via a scheduler (cron) or cloud schedule.

## License

MIT

```
```

---

## requirements.txt

```
python-dotenv
tweepy>=4.0.0
requests
Jinja2
```

---

## .gitignore

```
venv/
__pycache__/
.env
seen_tweets.json
grid.html
```

---

## .env.example

```
TWITTER_BEARER_TOKEN=YOUR_TWITTER_BEARER_TOKEN
TELEGRAM_BOT_TOKEN=
TELEGRAM_CHAT_ID=
QUERIES=base protocol,solana nft,web3 game
POLL_INTERVAL_SECONDS=300
MAX_TWEETS_PER_QUERY=5
```

---

## src/main.py

```python
"""
Grid Twitter Pulse - main runner
"""
import os
import time
import json
from dotenv import load_dotenv
import tweepy
from notifier import TelegramNotifier, ConsoleNotifier
from storage import Storage
from renderer import Renderer

load_dotenv()

BEARER = os.getenv('TWITTER_BEARER_TOKEN')
QUERIES = os.getenv('QUERIES', 'web3').split(',')
POLL_INTERVAL = int(os.getenv('POLL_INTERVAL_SECONDS', '300'))
MAX_TWEETS_PER_QUERY = int(os.getenv('MAX_TWEETS_PER_QUERY', '5'))

if not BEARER:
    raise SystemExit('TWITTER_BEARER_TOKEN not set in .env')

client = tweepy.Client(bearer_token=BEARER, wait_on_rate_limit=True)

storage = Storage('seen_tweets.json')

# choose notifier: Telegram if configured, else console
if os.getenv('TELEGRAM_BOT_TOKEN') and os.getenv('TELEGRAM_CHAT_ID'):
    notifier = TelegramNotifier(os.getenv('TELEGRAM_BOT_TOKEN'), os.getenv('TELEGRAM_CHAT_ID'))
else:
    notifier = ConsoleNotifier()

renderer = Renderer('grid.html')

print('Starting Grid Twitter Pulse...')

while True:
    new_items = []
    for q in QUERIES:
        query = q.strip()
        if not query:
            continue
        try:
            resp = client.search_recent_tweets(query=query, max_results=MAX_TWEETS_PER_QUERY, tweet_fields=['created_at','author_id','text'])
        except Exception as e:
            print('Twitter API error for query', query, e)
            continue

        tweets = resp.data or []
        for t in tweets:
            tid = str(t.id)
            if storage.is_seen(tid):
                continue
            # mark seen and collect
            storage.mark_seen(tid, {'id': tid, 'text': t.text, 'author_id': t.author_id, 'created_at': t.created_at.isoformat() if getattr(t, 'created_at', None) else None, 'query': query})
            new_items.append({'id': tid, 'text': t.text, 'author_id': t.author_id, 'created_at': t.created_at.isoformat() if getattr(t, 'created_at', None) else None, 'query': query})

    if new_items:
        for item in new_items:
            notifier.notify(item)
        # update grid.html with latest N tweets
        all_seen = storage.get_all_items()
        renderer.render(all_seen[-100:])

    time.sleep(POLL_INTERVAL)
```

---

## src/storage.py

```python
import json
import os

class Storage:
    def __init__(self, path='seen_tweets.json'):
        self.path = path
        self._data = []
        if os.path.exists(self.path):
            try:
                with open(self.path, 'r', encoding='utf-8') as f:
                    self._data = json.load(f)
            except Exception:
                self._data = []

    def is_seen(self, tweet_id: str) -> bool:
        return any(item['id'] == tweet_id for item in self._data)

    def mark_seen(self, tweet_id: str, meta: dict):
        self._data.append({'id': tweet_id, 'meta': meta})
        self._save()

    def _save(self):
        with open(self.path, 'w', encoding='utf-8') as f:
            json.dump(self._data, f, ensure_ascii=False, indent=2)

    def get_all_items(self):
        return [item['meta'] for item in self._data]
```

---

## src/notifier.py

```python
import requests
import os

class ConsoleNotifier:
    def notify(self, item: dict):
        print('\n--- NEW TWEET ---')
        print(f"Query: {item.get('query')}")
        print(f"Author ID: {item.get('author_id')}")
        print(f"Text: {item.get('text')}")
        print('-----------------\n')

class TelegramNotifier:
    def __init__(self, bot_token: str, chat_id: str):
        self.bot_token = bot_token
        self.chat_id = chat_id
        self.url = f'https://api.telegram.org/bot{self.bot_token}/sendMessage'

    def notify(self, item: dict):
        text = f"*New Tweet*\nQuery: {item.get('query')}\n{item.get('text')}"
        payload = {
            'chat_id': self.chat_id,
            'text': text,
            'parse_mode': 'Markdown'
        }
        try:
            resp = requests.post(self.url, data=payload, timeout=10)
            resp.raise_for_status()
        except Exception as e:
            # fallback to console if telegram fails
            print('Telegram notify failed:', e)
            print(text)
```

---

## src/renderer.py

```python
from jinja2 import Template

GRID_TEMPLATE = """
<!doctype html>
<html>
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width,initial-scale=1">
  <title>Grid Twitter Pulse</title>
  <style>
    body { font-family: -apple-system,BlinkMacSystemFont,Segoe UI,Roboto,Helvetica,Arial,sans-serif; padding: 16px; }
    .grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(240px, 1fr)); gap: 12px; }
    .card { border-radius: 8px; padding: 12px; box-shadow: 0 4px 10px rgba(0,0,0,0.05); background: #fff }
    .meta { font-size: 12px; color: #666 }
    pre { white-space: pre-wrap; word-wrap: break-word }
  </style>
</head>
<body>
  <h1>Grid Twitter Pulse</h1>
  <div class="grid">
    {% for t in tweets %}
      <div class="card">
        <div class="meta">{{ t.created_at or "" }} &middot; {{ t.query }}</div>
        <pre>{{ t.text }}</pre>
      </div>
    {% endfor %}
  </div>
</body>
</html>
"""

class Renderer:
    def __init__(self, outpath='grid.html'):
        self.outpath = outpath
        self.template = Template(GRID_TEMPLATE)

    def render(self, tweets):
        html = self.template.render(tweets=tweets[::-1])
        with open(self.outpath, 'w', encoding='utf-8') as f:
            f.write(html)
```

---


