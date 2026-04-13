# Expense Tracker — Full Build Context

## What We Built
Automated personal expense tracking for Jupiter (Federal Bank) transactions.

### Flow
```
Bank SMS arrives on iPhone
→ iMessage syncs to Mac (same Apple ID)
→ Python script polls ~/Library/Messages/chat.db every 10 seconds
→ Parses SMS: extracts amount, credit/debit type, date
→ POSTs to n8n webhook (hosted on Railway)
→ n8n sends Telegram message with inline category buttons
→ User taps a button
→ n8n asks for description via Telegram
→ User types description (or sends - to skip)
→ n8n appends row to Google Sheet
```

---

## Current Status
- ✅ Python script working — detects FEDBNK-S and FEDBNK-T messages
- ✅ n8n hosted on Railway (always-on, no Mac dependency for n8n)
- ✅ Telegram inline button flow working
- ✅ Google Sheet logging (category, description, amount, date, bank)
- ✅ Python script auto-starts on Mac login via launchd
- ⏳ Next: Add Kotak 811 and Bank of Baroda support

---

## Infrastructure

### Mac Setup
- Python script runs from: `~/Codex-and-other-things/Expense-tracker-agent/scripts/jupiter_tracker.py`
- State file: `~/.jupiter_tracker_state.json`
- Mac username: `hemantchhabria`
- Python path: `/opt/anaconda3/bin/python3`
- Script auto-starts on login via launchd plist: `~/Library/LaunchAgents/com.hemant.jupytertracker.plist`
- Logs: `~/Library/Logs/jupytertracker.log` and `~/Library/Logs/jupytertracker.error.log`

### n8n (Railway)
- Hosted at: `https://n8n-production-0e8d.up.railway.app`
- Login: credentials set during Railway deployment
- Two workflows:

#### Workflow A — Expense Tracker - Send
- Triggered by Python script POSTing transaction data
- Sends Telegram message with inline category buttons
- Webhook URL (Production): `https://n8n-production-0e8d.up.railway.app/webhook/4027de78-94e3-4172-8722-e151fc229832`
- Nodes: Webhook → Telegram Send Message (with inline keyboard)

#### Workflow B — Expense Tracker - Receive
- Triggered by Telegram (button taps + text messages)
- Telegram webhook registered at: `https://n8n-production-0e8d.up.railway.app/webhook/43fb3b1a-0e95-4501-8765-6da143bce113/webhook`
- Nodes: Telegram Trigger → Switch → [Code → Telegram Send Message] / [Code → Google Sheets]

### Telegram
- Bot: @Expensetally_bot
- Chat ID: `1269390790`
- Webhook registered via:
```bash
curl "https://api.telegram.org/bot{TOKEN}/setWebhook?url=https://n8n-production-0e8d.up.railway.app/webhook/43fb3b1a-0e95-4501-8765-6da143bce113/webhook"
```

### Google Sheets
- Document: Finance tally
- Sheet: 2026 Consolidated Jupiter
- Google Cloud Project ID: 1075000089385
- OAuth Client ID: `1075000089385-bpec7lh0r5murl4oduentd6talfp9m82.apps.googleusercontent.com`
- Authorized redirect URI: `https://n8n-production-0e8d.up.railway.app/rest/oauth2-credential/callback`

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

WEBHOOK_URL = "https://n8n-production-0e8d.up.railway.app/webhook/4027de78-94e3-4172-8722-e151fc229832"
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
        print("[ERROR] Cannot connect to n8n.")
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

## launchd Plist (auto-start on Mac login)

File: `~/Library/LaunchAgents/com.hemant.jupytertracker.plist`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.hemant.jupytertracker</string>
    <key>ProgramArguments</key>
    <array>
        <string>/opt/anaconda3/bin/python3</string>
        <string>/Users/hemantchhabria/Codex-and-other-things/Expense-tracker-agent/scripts/jupiter_tracker.py</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>StandardOutPath</key>
    <string>/Users/hemantchhabria/Library/Logs/jupytertracker.log</string>
    <key>StandardErrorPath</key>
    <string>/Users/hemantchhabria/Library/Logs/jupytertracker.error.log</string>
</dict>
</plist>
```

Commands:
```bash
# Load (start)
launchctl load ~/Library/LaunchAgents/com.hemant.jupytertracker.plist

# Unload (stop)
launchctl unload ~/Library/LaunchAgents/com.hemant.jupytertracker.plist

# Check if running
launchctl list | grep jupytertracker
```

---

## n8n Workflow A — Expense Tracker - Send

### Node 1: Webhook
- Method: POST
- Path: `4027de78-94e3-4172-8722-e151fc229832`
- Respond: When Last Node Finishes

### Node 2: Telegram — Send Message
- Credential: Telegram account (@Expensetally_bot)
- Chat ID: `1269390790`
- Operation: Send Message
- Parse Mode: HTML
- Message:
```
💳 New Transaction Detected

Bank: {{ $json.body.sender }}
Amount: ₹{{ $json.body.amount }}
Type: {{ $json.body.type }}
SMS: {{ $json.body.message }}

Select a category:
```
- Reply Markup: Inline Keyboard
  - Row 1: 🍔 Food (FD), 🛒 Groceries (GR), ⛽ Petrol (PT)
  - Row 2: 🛍 Shopping (SH), 🥤 Beverages (BV), 🎬 Entertainment (EN)
  - Row 3: 🚗 Transport (TP), 💳 Credit Card (CC), 💰 Investment (IN)
  - Row 4: 🏠 Rent (RN), 🔌 Electricity (EL), 📶 Wifi (WF)
  - Row 5: 💈 Haircut (HC), 🎲 Random (RD), 📝 Other (OTHER)

---

## n8n Workflow B — Expense Tracker - Receive

### Node 1: Telegram Trigger
- Credential: Telegram account (@Expensetally_bot)
- Trigger On: Callback Query, Message

### Node 2: Switch
- Rule 1: `{{ $json.callback_query ? 'callback' : 'message' }}` is equal to `callback` → output 0
- Rule 2: same expression is equal to `message` → output 1

### Node 3: Code (Save transaction — connected to Switch output 0)
```javascript
const data = $input.all()[0].json;

const staticData = $getWorkflowStaticData('global');
staticData.pendingTransaction = {
  category: data.callback_query.data,
  amount: data.callback_query.message.text.match(/Amount: ₹([\d.]+)/)?.[1],
  date: data.callback_query.message.text.match(/on (\d{2}-\d{2}-\d{4}|\d{2}[A-Z]{3}\d{2})/i)?.[1],
  bank: data.callback_query.message.text.match(/Bank: (\w+)/)?.[1],
  chat_id: data.callback_query.message.chat.id
};

return $input.all();
```

### Node 4: Telegram — Send Message (connected to Node 3)
- Chat ID: `{{ $json.callback_query.message.chat.id }}`
- Text: `Got it! Now send a description for this expense, or send - to skip.`

### Node 5: Code (Retrieve transaction — connected to Switch output 1)
```javascript
const staticData = $getWorkflowStaticData('global');
const pending = staticData.pendingTransaction;
const description = $input.all()[0].json.message.text;

const categoryMap = {
  FD:'Food',GR:'Groceries',RN:'Rent',CK:'Cook',MD:'Maid',
  PF:'Purifier',WF:'Wifi',RC:'Recharge',TR:'Trip',RF:'Refund',
  TP:'Transport',PT:'Petrol',BV:'Beverages',CC:'Credit Card',
  IN:'Investment',RD:'Random',SH:'Shopping',KT:'Kotak',GE:'Gaadi',
  EN:'Entertainment',PD:'Previous Dues',HC:'Haircut',EL:'Electricity'
};

function parseDate(raw) {
  if (!raw) return '';
  if (/\d{2}-\d{2}-\d{4}/.test(raw)) return raw;
  const match = raw.match(/(\d{2})(JAN|FEB|MAR|APR|MAY|JUN|JUL|AUG|SEP|OCT|NOV|DEC)(\d{2})/i);
  if (match) {
    const months = {JAN:'01',FEB:'02',MAR:'03',APR:'04',MAY:'05',JUN:'06',JUL:'07',AUG:'08',SEP:'09',OCT:'10',NOV:'11',DEC:'12'};
    return `${match[1]}-${months[match[2].toUpperCase()]}-20${match[3]}`;
  }
  return raw;
}

return [{
  json: {
    category: categoryMap[pending.category] || pending.category,
    description: description === '-' ? '' : description,
    amount: pending.amount,
    date: parseDate(pending.date),
    bank: pending.bank
  }
}];
```

### Node 6: Google Sheets — Append Row (connected to Node 5)
- Document: Finance tally
- Sheet: 2026 Consolidated Jupiter
- Mapping:

| Column | Expression |
|---|---|
| Expense and receivable name | `{{ $json.category }}` |
| Description of the expense | `{{ $json.description }}` |
| Amount | `{{ $json.amount }}` |
| Date | `{{ $json.date }}` |
| Bank | `{{ $json.bank }}` |

---

## Google Sheet Structure
1. Expense and receivable name (category full name)
2. Description of the expense
3. Amount (negative for credit, positive for debit)
4. Date (format DD-MM-YYYY)
5. Bank (e.g. "Jupiter")

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

## Bank SMS Senders
| Bank | Sender IDs | Status |
|---|---|---|
| Jupiter (Federal Bank) | FEDBNK-S, FEDBNK-T | ✅ Live |
| Kotak 811 | KOTAKB-S | ⏳ Not built |
| Bank of Baroda | BOBTXN-S, BOBSMS-S | ⏳ Not built |

---

## Known Issues / Quirks
- iMessage SMS sync to Mac has variable delay (seconds to minutes)
- attributedBody blob must be decoded — plain `text` column is always NULL for SMS
- Federal Bank SMS always starts with "Rs" — decoder uses `\x00Rs` marker
- Date format varies: UPI transactions use `DD-MM-YYYY`, debit card transactions use `DDMMMYY` (e.g. 11MAR26) — both handled
- n8n must be Published (not just Saved) for workflows to be active
- Mac must not be set to sleep — disable in System Settings → Energy Saver

## How to Start / Manage

### Python script (managed by launchd, auto-starts on login)
```bash
# Check if running
launchctl list | grep jupytertracker

# Restart
launchctl unload ~/Library/LaunchAgents/com.hemant.jupytertracker.plist
launchctl load ~/Library/LaunchAgents/com.hemant.jupytertracker.plist

# View logs
tail -f ~/Library/Logs/jupytertracker.log
tail -f ~/Library/Logs/jupytertracker.error.log
```

### n8n
- Always on at Railway — no action needed
- Access at: `https://n8n-production-0e8d.up.railway.app`
