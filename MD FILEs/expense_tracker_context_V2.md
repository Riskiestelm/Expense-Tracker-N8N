# Expense Tracker — Full Build Context

## What We Built
Automated personal expense tracking for Jupiter (Federal Bank) transactions.

### Flow
```
Bank SMS arrives on iPhone
→ iMessage syncs to Mac (same Apple ID)
→ Python script polls ~/Library/Messages/chat.db every 10 seconds
→ Parses SMS: extracts amount, credit/debit type
→ POSTs to n8n webhook
→ n8n sends Telegram message with resume link
→ User responds with category and description
→ n8n appends row to Google Sheet
```

---

## Current Status
- ✅ Python script working — detects FEDBNK-S and FEDBNK-T messages
- ✅ n8n webhook receiving data
- ✅ Telegram message sending
- ✅ Google Sheet logging (category, description, amount, date from SMS)
- ❌ Telegram UX — currently requires clicking a resume link in browser, not ideal
- ❌ ngrok URL changes on restart — Telegram webhook needs re-registering each time
- ⏳ Next: rebuild Telegram interaction as inline button flow with permanent URL

---

## Infrastructure

### Mac Setup
- n8n runs locally: `npx n8n` at `http://localhost:5678`
- ngrok exposes n8n publicly: `ngrok http 5678`
- Python script runs from: `~/Codex-and-other-things/Expense-tracker-agent/scripts/jupiter_tracker.py`
- State file: `~/.jupiter_tracker_state.json`
- Mac username: `hemantchhabria`

### n8n
- Version: 2.8.4
- Workflow name: "My workflow" (ID: 2WNipyht6h3ynW2g) — OLD, keep as backup
- Webhook URL: `http://localhost:5678/webhook/f981d699-e125-448a-87d5-7cbf2f1c8c63`
- New workflow to be built for button-based Telegram UX

### ngrok
- Free account, authtoken configured
- Config fixed with: `sudo chown -R hemantchhabria ~/.config/ngrok`
- Start with: `ngrok http 5678`
- **Problem: URL changes on every restart** — needs permanent solution
- Options: ngrok paid plan (static domain), or use Cloudflare Tunnel (free, permanent)

---

## Python Script (current working version)

File: `~/Codex-and-other-things/Expense-tracker-agent/scripts/jupiter_tracker.py`

```python
import sqlite3
import json
import time
import re
import requests
from pathlib import Path

WEBHOOK_URL = "http://localhost:5678/webhook/f981d699-e125-448a-87d5-7cbf2f1c8c63"
DB_PATH = Path.home() / "Library/Messages/chat.db"
STATE_FILE = Path.home() / ".jupiter_tracker_state.json"
POLL_INTERVAL = 10

TRACKED_SENDERS = {"FEDBNK-S", "FEDBNK-T"}

def load_state():
    if STATE_FILE.exists():
        with open(STATE_FILE) as f:
            return set(json.load(f))
    return set()

def save_state(processed_ids):
    with open(STATE_FILE, "w") as f:
        json.dump(list(processed_ids), f)

def decode_attributed_body(blob):
    """
    Federal Bank SMS text is stored in attributedBody blob.
    Text always starts with 'Rs' — we find \x00Rs and read from there.
    """
    if blob is None:
        return None
    try:
        idx = blob.find(b'\x00Rs')
        if idx != -1:
            chunk = blob[idx + 1:]
            result = ""
            for byte in chunk:
                ch = chr(byte)
                if ch.isprintable() or ch in ("\n", "\r", "\t"):
                    result += ch
                elif result:
                    break
            result = result.strip()
            if len(result) > 10:
                return result

        # Fallback
        anchor = b'NSString\x01'
        idx = blob.find(anchor)
        if idx == -1:
            return None
        chunk = blob[idx + len(anchor):]
        for i in range(len(chunk) - 1):
            if chunk[i] == 0x00 and chr(chunk[i + 1]).isprintable():
                result = ""
                for byte in chunk[i + 1:]:
                    ch = chr(byte)
                    if ch.isprintable() or ch in ("\n", "\r", "\t"):
                        result += ch
                    elif result:
                        break
                result = result.strip()
                if len(result) > 10:
                    return result
        return None
    except Exception:
        return None

def extract_amount(text):
    match = re.search(r'Rs\.?\s*([\d,]+(?:\.\d{1,2})?)', text, re.IGNORECASE)
    if match:
        return match.group(1).replace(",", "")
    return None

def parse_sms(sender, text):
    normalised = next((s for s in TRACKED_SENDERS if sender.endswith(s)), None)
    if normalised is None:
        return None

    amount = extract_amount(text)
    if not amount:
        return None

    text_lower = text.lower()

    if normalised == "FEDBNK-S":
        txn_type = "debit"
    elif normalised == "FEDBNK-T":
        if "debited" in text_lower:
            txn_type = "debit"
        elif "credited" in text_lower:
            txn_type = "credit"
        else:
            print(f"[SKIP] Cannot determine debit/credit: {text[:60]}")
            return None
    else:
        return None

    return {
        "sender": "Jupiter",
        "amount": amount,
        "type": txn_type,
        "message": text
    }

def fetch_new_messages(processed_ids):
    if not DB_PATH.exists():
        print(f"[ERROR] chat.db not found at {DB_PATH}")
        return []

    try:
        conn = sqlite3.connect(f"file:{DB_PATH}?mode=ro", uri=True)
        cursor = conn.cursor()
        cursor.execute("""
            SELECT message.ROWID, handle.id, message.text, message.attributedBody
            FROM message
            JOIN handle ON message.handle_id = handle.ROWID
            WHERE message.is_from_me = 0
            ORDER BY message.date DESC
            LIMIT 200
        """)
        rows = cursor.fetchall()
        conn.close()

        new_messages = []
        for msg_id, sender, text, attributed_body in rows:
            if msg_id in processed_ids:
                continue
            if not any(sender.endswith(s) for s in TRACKED_SENDERS):
                continue
            resolved_text = text or decode_attributed_body(attributed_body)
            if resolved_text:
                new_messages.append((msg_id, sender, resolved_text))

        return new_messages

    except Exception as e:
        print(f"[ERROR] Failed to read chat.db: {e}")
        return []

def send_to_webhook(payload):
    try:
        response = requests.post(WEBHOOK_URL, json=payload, timeout=10)
        if response.status_code == 200:
            print(f"[OK] Sent: ₹{payload['amount']} {payload['type']} via {payload['sender']}")
            return True
        else:
            print(f"[ERROR] n8n returned {response.status_code}: {response.text}")
            return False
    except requests.exceptions.ConnectionError:
        print("[ERROR] Cannot connect to n8n. Is it running at localhost:5678?")
        return False
    except Exception as e:
        print(f"[ERROR] Webhook failed: {e}")
        return False

def main():
    print("Jupiter Expense Tracker started. Polling every 10 seconds...")
    print(f"Watching: {', '.join(TRACKED_SENDERS)}")
    print("Press Ctrl+C to stop.\n")

    processed_ids = load_state()

    while True:
        new_messages = fetch_new_messages(processed_ids)
        for msg_id, sender, text in new_messages:
            parsed = parse_sms(sender, text)
            if parsed:
                success = send_to_webhook(parsed)
                if success:
                    processed_ids.add(msg_id)
                    save_state(processed_ids)
            else:
                processed_ids.add(msg_id)
                save_state(processed_ids)

        time.sleep(POLL_INTERVAL)

if __name__ == "__main__":
    main()
```

---

## n8n Workflow — Current (Old, keep as backup)

### Node 1: Webhook
- Method: POST
- Path: `f981d699-e125-448a-87d5-7cbf2f1c8c63`
- Respond: When Last Node Finishes

### Node 2: Telegram — Send message and wait for response
- Credential: Telegram account (Hemant's bot)
- Chat ID: `1269390790`
- Operation: Send and Wait for Response
- Response Type: Free Text
- Message template:
```
💳 New Transaction Detected

Bank: {{ $json.body.sender }}
Amount: ₹{{ $json.body.amount }}
Type: {{ $json.body.type }}
SMS: {{ $json.body.message }}

Reply in this exact format:
CATEGORY | description

Example: FD | lunch with friends

Categories:
GR-Groceries | RN-Rent | CK-Cook | MD-Maid
PF-Purifier | WF-Wifi | RC-Recharge | TR-Trip
RF-Refund | TP-Transport | PT-Petrol | FD-Food
BV-Beverages | CC-Credit Card | IN-Investment
RD-Random | SH-Shopping | KT-Kotak | GE-Gaadi
EN-Entertainment | PD-Previous Dues | HC-Haircut | EL-Electricity
```

### Node 3: Google Sheets — Append Row
- Connected to Hemant's Google account via OAuth2
- Google Cloud project ID: 1075000089385

| Column | Expression |
|---|---|
| Expense and receivable name | `{{ {FD:'Food',GR:'Groceries',RN:'Rent',CK:'Cook',MD:'Maid',PF:'Purifier',WF:'Wifi',RC:'Recharge',TR:'Trip',RF:'Refund',TP:'Transport',PT:'Petrol',BV:'Beverages',CC:'Credit Card',IN:'Investment',RD:'Random',SH:'Shopping',KT:'Kotak',GE:'Gaadi',EN:'Entertainment',PD:'Previous Dues',HC:'Haircut',EL:'Electricity'}[$json.data.text.split('|')[0].trim().toUpperCase()] || $json.data.text.split('|')[0].trim() }}` |
| Description | `{{ $json.data.text.split('|')[1].trim() }}` |
| Amount | `{{ $('Webhook').item.json.body.type === 'credit' ? '-' + $('Webhook').item.json.body.amount : $('Webhook').item.json.body.amount }}` |
| Date | `{{ $('Webhook').item.json.body.message.match(/on (\d{2}-\d{2}-\d{4})/)?.[1] }}` |
| Bank | `{{ $('Webhook').item.json.body.sender }}` |

---

## Google Sheet Structure
Sheet columns (in order):
1. Expense and receivable name (category full name)
2. Description of the expense
3. Amount (negative for credit, positive for debit)
4. Date (from SMS text, format DD-MM-YYYY)
5. Bank (sender name e.g. "Jupiter")

---

## Category Codes
| Code | Full Name |
|---|---|
| FD | Food |
| GR | Groceries |
| PT | Petrol |
| SH | Shopping |
| BV | Beverages |
| EN | Entertainment |
| RN | Rent |
| CK | Cook |
| MD | Maid |
| PF | Purifier |
| WF | Wifi |
| RC | Recharge |
| TR | Trip |
| RF | Refund |
| TP | Transport |
| CC | Credit Card |
| IN | Investment |
| RD | Random |
| KT | Kotak |
| GE | Gaadi |
| PD | Previous Dues |
| HC | Haircut |
| EL | Electricity |

---

## Bank SMS Senders (for future banks)
| Bank | Sender IDs | Notes |
|---|---|---|
| Jupiter (Federal Bank) | FEDBNK-S, FEDBNK-T | Currently tracked |
| Kotak 811 | KOTAKB-S | Not yet built |
| Bank of Baroda | BOBTXN-S, BOBSMS-S | Not yet built |

---

## What Needs to Be Built Next

### 1. Permanent public URL (Priority — needed for Telegram button flow)
**Problem:** ngrok free tier gives a new URL every restart. Telegram bot webhook needs a stable URL.

**Solution: Cloudflare Tunnel (free, permanent)**
- Install cloudflared on Mac
- Create a tunnel pointing to localhost:5678
- Get a permanent `*.trycloudflare.com` or custom domain URL
- Set this as n8n's WEBHOOK_URL env variable
- Register this URL as Telegram bot webhook

**Commands to set up:**
```bash
# Install
brew install cloudflare/cloudflare/cloudflared

# Login
cloudflared tunnel login

# Create tunnel
cloudflared tunnel create expense-tracker

# Route to n8n
cloudflared tunnel route dns expense-tracker your-subdomain.yourdomain.com

# Start
cloudflared tunnel run expense-tracker
```

### 2. New Telegram Button Flow (build after permanent URL is set)
**Goal:** Replace the resume-link UX with inline keyboard buttons in Telegram.

**Flow:**
```
Transaction detected
→ Telegram: sends message with 7 inline buttons:
  [Food] [Groceries] [Petrol] [Shopping] [Beverages] [Entertainment] [Other]
→ User taps a button
→ Telegram asks: "Add description? (send - to skip)"
→ User types description
→ Google Sheet row appended
```

**Requires two n8n workflows:**
- Workflow A: Webhook → Telegram Send Message (with inline keyboard JSON)
- Workflow B: Telegram Trigger (On callback query + On message) → Switch → Google Sheets

**Inline keyboard JSON format for Telegram:**
```json
{
  "inline_keyboard": [
    [
      {"text": "🍔 Food", "callback_data": "FD"},
      {"text": "🛒 Groceries", "callback_data": "GR"},
      {"text": "⛽ Petrol", "callback_data": "PT"}
    ],
    [
      {"text": "🛍 Shopping", "callback_data": "SH"},
      {"text": "🥤 Beverages", "callback_data": "BV"},
      {"text": "🎬 Entertainment", "callback_data": "EN"}
    ],
    [
      {"text": "📝 Other", "callback_data": "OTHER"}
    ]
  ]
}
```

**State management problem:**
When user taps a button, n8n needs to know which transaction it belongs to. Solution: store transaction data in n8n static data or pass it via callback_data.

### 3. Auto-start on Mac Login
After permanent URL and button flow are working:
- Create a launchd plist for the Python script
- Create a launchd plist for cloudflared tunnel
- Both auto-start on login without Terminal

### 4. Add More Banks
Once Jupiter flow is stable, replicate for:
- Kotak 811 (KOTAKB-S) — separate Google Sheet tab
- Bank of Baroda (BOBTXN-S, BOBSMS-S) — separate Google Sheet tab

---

## Known Issues / Quirks
- iMessage SMS sync to Mac has variable delay (seconds to minutes)
- attributedBody blob must be decoded — plain `text` column is always NULL for SMS
- Federal Bank SMS always starts with "Rs" — decoder uses `\x00Rs` marker
- n8n reverts to draft after saving — must explicitly click Publish after every change
- ngrok URL changes on restart — currently must manually update n8n WEBHOOK_URL env var

---

## How to Start Everything (current setup)
Open 3 Terminal windows:

**Terminal 1 — n8n:**
```bash
WEBHOOK_URL=https://your-ngrok-url.ngrok-free.app npx n8n
```

**Terminal 2 — ngrok:**
```bash
ngrok http 5678
```
(copy the URL and use it in Terminal 1)

**Terminal 3 — Python script:**
```bash
python3 ~/Codex-and-other-things/Expense-tracker-agent/scripts/jupiter_tracker.py
```
