# Expense Tracker — Full Build Context V6

## What We Built
Automated personal expense tracking for Jupiter (Federal Bank), Kotak 811, and Bank of Baroda transactions, with a Google Sheets dashboard and Splitwise expense splitting via a Telegram Mini App.

### Core Flow
```
Bank SMS arrives on iPhone
→ iMessage syncs to Mac (same Apple ID)
→ Python script polls ~/Library/Messages/chat.db every 10 seconds
→ Parses SMS: extracts amount, credit/debit type, date, bank
→ POSTs to n8n webhook (hosted on Railway)
→ n8n sends Telegram message with inline category buttons
→ User taps a button
→ n8n asks for description via Telegram
→ User types description (or sends - to skip)
→ n8n appends row to Google Sheet
```

### Splitwise Split Flow (new)
```
Same Telegram message also has a "💸 Split this" button
→ User taps it
→ Telegram Mini App opens inside Telegram (hosted on Railway)
→ URL carries amount, description, bank, date as params
→ Mini App fetches Splitwise groups from Flask backend
→ User picks a group → members shown with amount inputs
→ Amounts pre-filled equally, user adjusts as needed
→ Running total shows Balanced/Unbalanced in real time
→ Submit button disabled until balanced
→ User edits description if needed, taps Submit
→ Flask backend calls Splitwise API → expense created
→ Mini App closes with success message
```

---

## Current Status
- ✅ Jupiter (Federal Bank) — fully operational
- ✅ Kotak 811 — fully operational
- ✅ Bank of Baroda — fully operational (BOBSMS-S and BOBTXN-S)
- ✅ n8n hosted on Railway (always-on)
- ✅ Telegram inline button flow working (updated categories)
- ✅ Google Sheet logging (category, description, amount, date, bank, type)
- ✅ Credits stored as negative amounts (handled in n8n Node 5)
- ✅ Dates written as MM/DD/YYYY (real date format, not text)
- ✅ Dashboard with month dropdown — switch months instantly
- ✅ Opening balance table with auto-cascading formulas
- ✅ Category breakdown table with SUMIFS per category per month
- ✅ Pie chart showing spending by category for selected month
- ✅ Splitwise Mini App — fully operational and live on Railway
- ✅ "💸 Split this" button wired into n8n Workflow A

---

## Infrastructure

### Mac Setup
- Python script: `~/Codex-and-other-things/Expense-tracker-agent/scripts/jupiter_tracker.py`
- State file: `~/.jupiter_tracker_state.json`
- Mac username: `hemantchhabria`
- Python path: `/opt/anaconda3/bin/python3.12`
- Script auto-starts on login via launchd plist: `~/Library/LaunchAgents/com.hemant.jupytertracker.plist`
- Logs: `~/Library/Logs/jupytertracker.log` and `~/Library/Logs/jupytertracker.error.log`
- Full Disk Access granted to `/opt/anaconda3/bin/python3.12` in System Settings → Privacy & Security
- Mac set to never sleep (Battery → Prevent Mac from sleeping automatically)

### n8n (Railway)
- Hosted at: `https://n8n-production-0e8d.up.railway.app`
- Two workflows:

#### Workflow A — Expense Tracker - Send
- Webhook URL: `https://n8n-production-0e8d.up.railway.app/webhook/4027de78-94e3-4172-8722-e151fc229832`
- Nodes: Webhook → Telegram Send Message (with inline keyboard)
- Inline keyboard has category buttons (rows 1–4) + "💸 Split this" URL button (row 5)
- Split this button URL format:
  `https://splitwise-miniapp-production.up.railway.app/?amount={{ $json.body.amount }}&description={{ $json.body.message }}&bank={{ $json.body.sender }}&date={{ $json.body.date }}`
- Note: date param with `/` must be URL-encoded as `%2F` for correct parsing

#### Workflow B — Expense Tracker - Receive
- Telegram webhook: `https://n8n-production-0e8d.up.railway.app/webhook/43fb3b1a-0e95-4501-8765-6da143bce113/webhook`
- Nodes: Telegram Trigger → Switch → [Node 3: Save transaction] → [Node 4: Ask for description] / [Node 5: Retrieve transaction] → [Node 6: Google Sheets append]

### Telegram
- Bot: @Expensetally_bot
- Chat ID: `1269390790`

### Google Sheets
- Document: Finance tally
- Data sheet: `2026 Consolidated`
- Dashboard sheet: `Dashboard`
- Google Cloud Project ID: 1075000089385
- OAuth Client ID: `1075000089385-bpec7lh0r5murl4oduentd6talfp9m82.apps.googleusercontent.com`

### Splitwise Mini App (new)
- GitHub repo: `https://github.com/Riskiestelm/splitwise-miniapp` (private)
- Railway URL: `https://splitwise-miniapp-production.up.railway.app`
- Local path: `~/Codex-and-other-things/Expense-tracker-agent/splitwise-miniapp/`
- Stack: Python Flask + gunicorn + plain HTML/JS frontend
- Railway env variable: `SPLITWISE_API_KEY`
- Auto-deploys from GitHub on every push to main

---

## Splitwise Mini App — Architecture

### File Structure
```
splitwise-miniapp/
├── app.py              # Flask backend
├── static/
│   └── index.html      # Telegram Mini App UI
├── requirements.txt    # flask, requests, flask-cors, gunicorn
├── Procfile            # web: gunicorn app:app --bind 0.0.0.0:$PORT
└── .env.example        # SPLITWISE_API_KEY=your_key_here
```

### Flask Endpoints
- `GET /` — serves static/index.html
- `GET /groups` — fetches Splitwise groups, returns id + name + all members (including Hemant)
- `POST /create-expense` — creates Splitwise expense

### POST /create-expense payload
```json
{
  "group_id": 66008700,
  "description": "Dinner",
  "total": 1200.00,
  "splits": {
    "self_amount": 400.00,
    "others": [
      {"user_id": 45606672, "amount": 400.00},
      {"user_id": 62176651, "amount": 400.00}
    ]
  }
}
```

### Splitwise API
- Base URL: `https://secure.splitwise.com/api/v3.0`
- Auth: `Authorization: Bearer YOUR_API_KEY`
- Key registered at: `https://secure.splitwise.com/oauth_clients`
- Hemant's Splitwise user ID: `46271821`
- Currency: INR

### Mini App UI behaviour
- Reads URL params: `amount`, `description`, `bank`, `date`
- Step 1: Shows transaction card + group list
- Step 2: Shows members with editable amount inputs, pre-filled equally
- Running total vs transaction total shown — "Balanced ✅" or "Unbalanced ❌"
- Submit button disabled (greyed out, opacity 0.4) when unbalanced
- Description is editable inline before submitting
- On success: shows "Expense Added!" confirmation, then closes

### Active Splitwise Groups
| Group | ID | Key Members |
|---|---|---|
| Reliable Pride | 66008700 | Hemant, Ayush, Harsh |
| BANG-A-LORE | 70649501 | Hemant, Kunal, Ayush, Kriti, Shubh |
| H2k2 | 92517925 | Hemant, Kriti, Kunal, Harsh |
| H@-BSDK | 81408874 | Hemant, Kunal, Ayush, Disha, Sarmi, Ankit, Harsh, Kriti |
| YouTube Premium | 74648390 | Hemant, Kriti, Tanishka, Kunal, Ayush |

(Old/inactive groups archived in Splitwise app — API returns only active groups)

### Key Member IDs
| Name | Splitwise ID |
|---|---|
| Hemant (you) | 46271821 |
| Ayush | 45606672 |
| Shubh | 45186417 |
| Kunal | 58911127 |
| Kriti | 67346118 |
| Tanishka | 49666031 |
| Harsh Popat | 62176651 |
| Harsh Kashyap | 62761467 |

### Deploying updates to Mini App
```bash
cd ~/Codex-and-other-things/Expense-tracker-agent/splitwise-miniapp
# make changes
git add .
git commit -m "description of change"
git push
# Railway auto-redeploys in ~1 minute
```

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

TRACKED_SENDERS = {"FEDBNK-S", "FEDBNK-T", "KOTAKB-S", "BOBSMS-S", "BOBTXN-S"}

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

def normalise_date(raw):
    m = re.match(r'(\d{2})/(\d{2})/(\d{4})', raw)
    if m:
        return f"{m.group(1)}-{m.group(2)}-{m.group(3)}"
    m = re.match(r'(\d{2})-(\d{2})-(\d{2})$', raw)
    if m:
        return f"{m.group(1)}-{m.group(2)}-20{m.group(3)}"
    return raw

def parse_sms(sender, text):
    normalised = next((s for s in TRACKED_SENDERS if sender.endswith(s)), None)
    if normalised is None:
        return None

    amount = extract_amount(text)
    if not amount:
        return None

    text_lower = text.lower()

    if normalised == "FEDBNK-S":
        return {"sender": "Jupiter", "amount": amount, "type": "debit", "date": "", "message": text}

    elif normalised == "FEDBNK-T":
        if "debited" in text_lower:
            txn_type = "debit"
        elif "credited" in text_lower:
            txn_type = "credit"
        else:
            print(f"[SKIP] Jupiter: cannot determine debit/credit: {text[:60]}")
            return None
        return {"sender": "Jupiter", "amount": amount, "type": txn_type, "date": "", "message": text}

    elif normalised == "KOTAKB-S":
        if text_lower.startswith("received"):
            txn_type = "credit"
        elif "spent via kotak debit card" in text_lower or text_lower.startswith("sent"):
            txn_type = "debit"
        else:
            print(f"[SKIP] Kotak: cannot determine type: {text[:60]}")
            return None
        date_match = re.search(r'(\d{2}[/-]\d{2}[/-]\d{2,4})', text)
        date = normalise_date(date_match.group(1)) if date_match else ""
        return {"sender": "Kotak", "amount": amount, "type": txn_type, "date": date, "message": text}

    elif normalised in ("BOBSMS-S", "BOBTXN-S"):
        if "credited" in text_lower:
            txn_type = "credit"
        elif "dr." in text_lower:
            txn_type = "debit"
        else:
            print(f"[SKIP] BOB: cannot determine type: {text[:60]}")
            return None
        date_match = re.search(r'(\d{2}[/-]\d{2}[/-]\d{2,4})', text)
        date = normalise_date(date_match.group(1)) if date_match else ""
        return {"sender": "BOB", "amount": amount, "type": txn_type, "date": date, "message": text}

    return None

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
    print("Expense Tracker started. Polling every 10 seconds...")
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

## launchd Plist

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
        <string>/opt/anaconda3/bin/python3.12</string>
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
# Check if running
launchctl list | grep jupytertracker

# Restart
launchctl unload ~/Library/LaunchAgents/com.hemant.jupytertracker.plist
launchctl load ~/Library/LaunchAgents/com.hemant.jupytertracker.plist

# View logs
tail -f ~/Library/Logs/jupytertracker.log
```

---

## n8n Workflow A — Expense Tracker - Send

### Node 1: Webhook
- Method: POST
- Path: `4027de78-94e3-4172-8722-e151fc229832`
- Respond: When Last Node Finishes

### Node 2: Telegram — Send Message
- Chat ID: `1269390790`
- Parse Mode: HTML
- Message:
```
💳 New Transaction Detected

Bank: {{ $json.body.sender }}
Amount: ₹{{ $json.body.amount }}
Type: {{ $json.body.type }}
Date: {{ $json.body.date }}
SMS: {{ $json.body.message }}

Select a category:
```
- Inline Keyboard:
  - Row 1: 🛒 Groceries (GR), 🍔 Food (FD), 🏢 Flat expenses (FE)
  - Row 2: 🛍 Shopping (SH), 🥤 Beverages (BV), 🎬 Entertainment (EN)
  - Row 3: 🚗 Transport (TP), 💳 CC + previous dues (CC), 💰 Investment (IN)
  - Row 4: 💈 Haircut (HC), 🔄 Refund (RF), ⛽ Petrol (PE), 📝 Other (RD)
  - Row 5: 💸 Split this (URL button → Mini App)

### Row 5 button configuration
- Type: URL (not callback)
- Text: `💸 Split this`
- URL: `https://splitwise-miniapp-production.up.railway.app/?amount={{ $json.body.amount }}&description={{ $json.body.message }}&bank={{ $json.body.sender }}&date={{ $json.body.date }}`
- Note: if date contains `/` characters, they should be URL-encoded as `%2F`

---

## n8n Workflow B — Expense Tracker - Receive

### Node 1: Telegram Trigger
- Trigger On: Callback Query, Message

### Node 2: Switch
- Rule 1: `{{ $json.callback_query ? 'callback' : 'message' }}` equals `callback` → output 0
- Rule 2: same expression equals `message` → output 1

### Node 3: Code (Save transaction — Switch output 0)
```javascript
const data = $input.all()[0].json;
const staticData = $getWorkflowStaticData('global');
staticData.pendingTransaction = {
  category: data.callback_query.data,
  amount: data.callback_query.message.text.match(/Amount: ₹([\d.]+)/)?.[1],
  type: data.callback_query.message.text.match(/Type: (\w+)/)?.[1],
  date: data.callback_query.message.text.match(/Date: (\d{2}-\d{2}-\d{4})/)?.[1] || '',
  rawDate: data.callback_query.message.text.match(/on (\d{2}-\d{2}-\d{4}|\d{2}[A-Z]{3}\d{2})/i)?.[1] || '',
  bank: data.callback_query.message.text.match(/Bank: (\w+)/)?.[1],
  chat_id: data.callback_query.message.chat.id
};
return $input.all();
```

### Node 4: Telegram — Send Message (connected to Node 3)
- Chat ID: `{{ $json.callback_query.message.chat.id }}`
- Text: `Got it! Now send a description for this expense, or send - to skip.`

### Node 5: Code (Retrieve transaction — Switch output 1)
```javascript
const staticData = $getWorkflowStaticData('global');
const pending = staticData.pendingTransaction;
const description = $input.all()[0].json.message.text;

const categoryMap = {
  FD: 'Food', GR: 'Groceries', FE: 'Flat expenses', SH: 'Shopping',
  BV: 'Beverages', EN: 'Entertainment', TP: 'Transport',
  CC: 'CC + previous dues', IN: 'Investment', HC: 'Haircut',
  RF: 'Refund', PE: 'Petrol & other expenses', RD: 'Random'
};

function toSheetDate(raw) {
  if (!raw) return '';
  let m = raw.match(/^(\d{2})-(\d{2})-(\d{4})$/);
  if (m) return `${m[2]}/${m[1]}/${m[3]}`;
  m = raw.match(/^(\d{2})-(\d{2})-(\d{2})$/);
  if (m) return `${m[2]}/${m[1]}/20${m[3]}`;
  m = raw.match(/(\d{2})(JAN|FEB|MAR|APR|MAY|JUN|JUL|AUG|SEP|OCT|NOV|DEC)(\d{2})/i);
  if (m) {
    const months = {JAN:'01',FEB:'02',MAR:'03',APR:'04',MAY:'05',JUN:'06',JUL:'07',AUG:'08',SEP:'09',OCT:'10',NOV:'11',DEC:'12'};
    return `${months[m[2].toUpperCase()]}/${m[1]}/20${m[3]}`;
  }
  return raw;
}

const rawDate = pending.date || pending.rawDate;
const date = toSheetDate(rawDate);
const amount = pending.type === 'credit' ? `-${pending.amount}` : pending.amount;

return [{
  json: {
    category: categoryMap[pending.category] || pending.category,
    description: description === '-' ? '' : description,
    amount: amount,
    date: date,
    bank: pending.bank,
    type: pending.type
  }
}];
```

### Node 6: Google Sheets — Append Row (connected to Node 5)
- Document: Finance tally
- Sheet: `2026 Consolidated`
- Column mapping: A=category, B=description, C=amount, D=date, E=bank, F=type

---

## Google Sheet Structure

### Data Sheet: `2026 Consolidated`

| Column | Content | Notes |
|---|---|---|
| A | Category | Full name e.g. Food, Groceries |
| B | Description | Free text or blank |
| C | Amount | Positive = debit, Negative = credit |
| D | Date | MM/DD/YYYY format (real date, not text) |
| E | Bank | Jupiter / Kotak / BOB |
| F | Type | debit / credit (lowercase) |

---

## Dashboard Sheet

Sheet: `Dashboard` in the same Google Sheets file.

### Month Selector (with dropdown)
- `A1` — Label `Month:`
- `B1` — Dropdown (Data validation → Dropdown from range → `H2:H13`)
- `A3` / `B3` — Start Date formula:
  `=DATE(VALUE(RIGHT(B1,4)),MATCH(LEFT(B1,FIND(" ",B1)-1),{"January","February","March","April","May","June","July","August","September","October","November","December"},0),1)`
- `A4` / `B4` — End Date formula: `=EOMONTH(B3,0)`

### Per-Bank Rows
**Jupiter:**
- `B7` — Spent: SUMIFS on debit
- `B8` — Received: ABS SUMIFS on credit
- `B9` — Op Bal: `=INDEX(I:I,MATCH(B1,H:H,0))`
- `B10` — Current Bal: `=B9-B7+B8`

**Kotak:** rows 12–15 (same pattern with "Kotak")
**BOB:** rows 17–20 (same pattern with "BOB")

### Opening Balance Table (columns H–M)
- H: Month names (March 2026 onward)
- I: Jupiter opening balance
- J: Kotak opening balance
- K: BOB opening balance
- L: Start date helper
- M: End date helper
- Row 2 (March 2026): manually entered
- Row 3+: auto-cascading SUMIFS formula from previous month

### Category Breakdown Table (columns D–E)
13 categories with SUMIFS formulas. Refund row omits "debit" filter to pick up credits.

### Pie Chart
- Data range: D1:E14
- Auto-updates when month changes

---

## Category Codes

| Code | Full Name | Telegram Button |
|---|---|---|
| GR | Groceries | 🛒 Groceries |
| FD | Food | 🍔 Food |
| FE | Flat expenses | 🏢 Flat expenses |
| SH | Shopping | 🛍 Shopping |
| BV | Beverages | 🥤 Beverages |
| EN | Entertainment | 🎬 Entertainment |
| TP | Transport | 🚗 Transport |
| CC | CC + previous dues | 💳 CC + previous dues |
| IN | Investment | 💰 Investment |
| HC | Haircut | 💈 Haircut |
| RF | Refund | 🔄 Refund |
| PE | Petrol & other expenses | ⛽ Petrol |
| RD | Random | 📝 Other |

---

## Bank SMS Senders
| Bank | Sender IDs | Status |
|---|---|---|
| Jupiter (Federal Bank) | FEDBNK-S, FEDBNK-T | ✅ Live |
| Kotak 811 | KOTAKB-S, VK-KOTAKB-S, AD-KOTAKB-S | ✅ Live |
| Bank of Baroda | BOBSMS-S, BOBTXN-S + prefixed variants | ✅ Live |

---

## Known Issues / Quirks
- SMS text in `chat.db` is stored in `attributedBody` binary blob, not plain `text` column
- Federal Bank SMS always starts with "Rs" — decoder uses `\x00Rs` marker
- Sender normalisation uses `endswith` matching to handle prefixed IDs like `VK-KOTAKB-S`
- Credits stored as negative numbers in the sheet
- Dates written as MM/DD/YYYY so Google Sheets treats them as real dates (not text)
- n8n webhook data lands in a `body` wrapper: field paths are `$json.body.sender` etc.
- macOS Full Disk Access must be granted to `/opt/anaconda3/bin/python3.12`
- iCloud Messages must be enabled on Mac for SMS sync from iPhone
- Mac must not sleep — disable in System Settings → Battery
- Locking Mac is fine — sleep stops the script
- launchd manages the Python script — Terminal does not need to be open
- Category names in Dashboard column D must exactly match column A in consolidated sheet
- Month values in opening balance table must have a space: `March 2026` not `March2026`
- Refund formula must NOT have the "debit" filter
- Mini App URL date param with `/` must be URL-encoded as `%2F` for correct parsing
- Port 5000 on Mac is occupied by AirPlay Receiver — use port 5001 for local Flask dev
- Splitwise free tier API has conservative rate limits — fine for personal use

---

## How to Start / Manage

### Python script
```bash
launchctl list | grep jupytertracker        # check if running
launchctl unload ~/Library/LaunchAgents/com.hemant.jupytertracker.plist  # stop
launchctl load ~/Library/LaunchAgents/com.hemant.jupytertracker.plist    # start
tail -f ~/Library/Logs/jupytertracker.log   # view logs
```

### n8n
- Always on at Railway — no action needed
- Access at: `https://n8n-production-0e8d.up.railway.app`

### Splitwise Mini App
- Always on at Railway — no action needed
- Access at: `https://splitwise-miniapp-production.up.railway.app`
- To update: push to GitHub → Railway auto-redeploys
- Local dev: `cd ~/Codex-and-other-things/Expense-tracker-agent/splitwise-miniapp && source venv/bin/activate && SPLITWISE_API_KEY=xxx python app.py`

---

## What's Next
- Potential: Visual formatting/beautification of the Google Sheets dashboard
- Potential: Add more banks as needed
- Potential: Yearly rollover (new consolidated sheet for 2027)
- Potential: Add "not a split expense" quick dismiss to the Mini App
