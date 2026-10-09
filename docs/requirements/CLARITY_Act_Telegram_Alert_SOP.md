# SOP: Real-Time Telegram Alert Pipeline

**Target Event:** US Senate CLARITY Act Procedural Cloture Vote (September 15, 2026)

**Critical Operational Context:** The September 15th vote is a procedural cloture vote on the motion to proceed (requiring a 60-vote threshold to break a filibuster), not final bill passage. Instant notification on the outcome (pass/fail) is vital for opening or closing open perp and altcoin positions instantly.

## 1. Telegram Bot & Channel Setup

- Open Telegram and message `@BotFather` to create a new bot via the `/newbot` command.
- Copy the HTTP API Token provided.
- Create a dedicated Telegram channel or private group for alerts and add the new bot as an Administrator with post permissions.
- Obtain the target Chat ID (numeric ID for private groups or `@channelname` for public channels).

## 2. Data Pipeline Options

### Option A: RSS Webhook via Make.com / Zapier (Low-Code)

- **Trigger Module:** RSS / Watch RSS Feed targeting high-speed financial and political news wires (e.g., Google News custom feed: `https://news.google.com/rss/search?q=CLARITY+Act+Senate+when:1d`).
- **Filter Module:** Set text matching conditions to require keywords: `CLARITY Act`, `cloture`, `senate`, or `crypto vote`.
- **Action Module:** Telegram → Send a Message using your Bot API Token and Chat ID. Ensure message notifications are active.

### Option B: Python Worker Script (Custom Infrastructure)

Deploy the following worker script to a low-latency server (e.g., AWS EC2 or DigitalOcean) to poll breaking feeds every 30 seconds:

```python
import os
import time
import feedparser
import requests

TELEGRAM_BOT_TOKEN = os.getenv("TELEGRAM_BOT_TOKEN")
CHAT_ID = os.getenv("TELEGRAM_CHAT_ID")
RSS_URL = "https://news.google.com/rss/search?q=CLARITY+Act+Senate+when:1d"

KEYWORDS = ["clarity act", "cloture", "senate vote", "crypto bill"]
seen_entries = set()

def send_telegram_alert(text):
    url = f"https://api.telegram.org/bot{TELEGRAM_BOT_TOKEN}/sendMessage"
    payload = {
        "chat_id": CHAT_ID,
        "text": text,
        "parse_mode": "Markdown",
        "disable_notification": False
    }
    requests.post(url, json=payload)

def check_feed():
    feed = feedparser.parse(RSS_URL)
    for entry in feed.entries:
        if entry.id not in seen_entries:
            seen_entries.add(entry.id)
            content_lower = (entry.title + " " + entry.get("summary", "")).lower()
            if any(kw in content_lower for kw in KEYWORDS):
                msg = f"🚨 **CLARITY ACT CLOTURE VOTE ALERT** 🚨\n\n{entry.title}\n\n[Read Wire]({entry.link})"
                send_telegram_alert(msg)

if __name__ == "__main__":
    while True:
        try:
            check_feed()
        except Exception as e:
            print(f"Error: {e}")
        time.sleep(30)
```

## 3. Deployment & Mobile Verification

- **Uptime:** Ensure 24/7 server execution leading up to and through the September 15th session.
- **Device Alert Override:** Configure mobile Telegram app notification settings for the alert channel to "Always On" with custom high-priority sound tones to bypass Do Not Disturb mode.

---
*Prepared for Trading Operations & Execution Infrastructure*
