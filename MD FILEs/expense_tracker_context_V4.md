# Expense Tracker — Full Build Context V5

## What We Built
Automated personal expense tracking for Jupiter (Federal Bank), Kotak 811, and Bank of Baroda transactions, with a Google Sheets dashboard.

### Flow
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

---

## Current Status
- ✅ Jupiter (Federal Bank) — fully operational
- ✅ Kotak 811 — fully operational
- ✅ Bank of Baroda — fully operational (BOBSMS-S and BOBTXN-S)
- ✅ n8n hosted on Railway (always-on)
- ✅ Telegram inline button flow working
- ✅ Google Sheet logging (category, description, amount, date, bank, type)
- ✅ Credits stored as negative amounts (handled in n8n Node 5)
- ✅ Dates written as MM/DD/YYYY (real date format, not text)
- ✅ Dashboard sheet built with SUMIFS formulas — running balances per bank
- ⏳ Next: Beautiful dashboard UI using Google Apps Script

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
- Mac set to never sleep (Battery → Prevent Mac from sleeping automatically) so script keeps polling

### n8n (Railway)
- Hosted at: `https://n8n-production-0e8d.up.railway.app`
- Two workflows:

#### Workflow A — Expense Tracker - Send
- Webhook URL: `https://n8n-production-0e8d.up.railway.app/webhook/4027de78-94e3-4172-8722-e151fc229832`
- Nodes: Webhook → Telegram Send Message (with inline keyboard)

#### Workflow B — Expense Tracker - Receive
- Telegram webhook: `https://n8n-production-0e8d.up.railway.app/webhook/43fb3b1a-0e95-4501-8765-6da143bce113/webhook`
- Nodes: Telegram Trigger → Switch → [Node 3: Save transaction] → [Node 4: Ask for description] / [Node 5: Retrieve transaction] → [Node 6: Google Sheets append]

### Telegram
- Bot: @Expensetally_bot
- Chat ID: `1269390790`

### Google Sheets
- Document: Finance tally
- Sheet: 2026 Consolidated Jupiter
- Google Cloud Project ID: 1075000089385
- OAuth Client ID: `1075000089385-bpec7lh0r5murl4oduentd6talfp9m82.apps.googleusercontent.com`

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
    # DD/MM/YYYY → DD-MM-YYYY
    m = re.match(r'(\d{2})/(\d{2})/(\d{4})', raw)
    if m:
        return f"{m.group(1)}-{m.group(2)}-{m.group(3)}"
    # DD-MM-YY → DD-MM-20YY
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
        return {
            "sender": "Jupiter",
            "amount": amount,
            "type": "debit",
            "date": "",
            "message": text
        }

    elif normalised == "FEDBNK-T":
        if "debited" in text_lower:
            txn_type = "debit"
        elif "credited" in text_lower:
            txn_type = "credit"
        else:
            print(f"[SKIP] Jupiter: cannot determine debit/credit: {text[:60]}")
            return None
        return {
            "sender": "Jupiter",
            "amount": amount,
            "type": txn_type,
            "date": "",
            "message": text
        }

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

        return {
            "sender": "Kotak",
            "amount": amount,
            "type": txn_type,
            "date": date,
            "message": text
        }

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

        return {
            "sender": "BOB",
            "amount": amount,
            "type": txn_type,
            "date": date,
            "message": text
        }

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
# Load (start)
launchctl load ~/Library/LaunchAgents/com.hemant.jupytertracker.plist

# Unload (stop)
launchctl unload ~/Library/LaunchAgents/com.hemant.jupytertracker.plist

# Check if running
launchctl list | grep jupytertracker

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
  - Row 1: 🍔 Food (FD), 🛒 Groceries (GR), ⛽ Petrol (PT)
  - Row 2: 🛍 Shopping (SH), 🥤 Beverages (BV), 🎬 Entertainment (EN)
  - Row 3: 🚗 Transport (TP), 💳 Credit Card (CC), 💰 Investment (IN)
  - Row 4: 🏠 Rent (RN), 🔌 Electricity (EL), 📶 Wifi (WF)
  - Row 5: 💈 Haircut (HC), 🔄 Refund (RF), 📝 Other (RD)

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
  FD:'Food',GR:'Groceries',RN:'Rent',CK:'Cook',MD:'Maid',
  PF:'Purifier',WF:'Wifi',RC:'Recharge',TR:'Trip',RF:'Refund',
  TP:'Transport',PT:'Petrol',BV:'Beverages',CC:'Credit Card',
  IN:'Investment',RD:'Random',SH:'Shopping',KT:'Kotak',GE:'Gaadi',
  EN:'Entertainment',PD:'Previous Dues',HC:'Haircut',EL:'Electricity'
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
- Sheet: 2026 Consolidated Jupiter
- Column mapping:

| Column | Expression |
|---|---|
| A - Expense and receivable name | `{{ $json.category }}` |
| B - Description | `{{ $json.description }}` |
| C - Amount | `{{ $json.amount }}` |
| D - Date | `{{ $json.date }}` |
| E - Bank | `{{ $json.bank }}` |
| F - Type | `{{ $json.type }}` |

---

## Google Sheet Structure

Sheet: `2026 Consolidated Jupiter`

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

### Current Layout
- `A1` / `B1` — Label "Month:" and value e.g. `March 2026` (typed manually)
- `A3` / `B3` — Label "Start Date:" and formula:
  `=DATE(VALUE(RIGHT(B1,4)),MATCH(LEFT(B1,FIND(" ",B1)-1),{"January","February","March","April","May","June","July","August","September","October","November","December"},0),1)`
- `A4` / `B4` — Label "End Date:" and formula: `=EOMONTH(B3,0)`

### Per-Bank Rows
For each bank: Spent, Received, Opening Balance, Closing Balance.

**Jupiter (rows 8–12):**
- `B8` — Jupiter Spent: `=SUMIFS('2026 Consolidated Jupiter'!C:C,'2026 Consolidated Jupiter'!E:E,"Jupiter",'2026 Consolidated Jupiter'!D:D,">="&B3,'2026 Consolidated Jupiter'!D:D,"<="&B4,'2026 Consolidated Jupiter'!F:F,"debit")`
- `B10` — Jupiter Received: `=ABS(SUMIFS('2026 Consolidated Jupiter'!C:C,'2026 Consolidated Jupiter'!E:E,"Jupiter",'2026 Consolidated Jupiter'!D:D,">="&B3,'2026 Consolidated Jupiter'!D:D,"<="&B4,'2026 Consolidated Jupiter'!F:F,"credit"))`
- `B11` — Jupiter Opening Balance: **entered manually each month**
- `B12` — Jupiter Closing Balance: `=B11-B8+B10`

**Kotak (rows 14–17):**
- `B14` — Kotak Spent: (same SUMIFS with "Kotak")
- `B15` — Kotak Received: (same ABS SUMIFS with "Kotak")
- `B16` — Kotak Opening Balance: **entered manually each month**
- `B17` — Kotak Closing Balance: `=B16-B14+B15`

**BOB (rows 19–22):**
- `B19` — BOB Spent: (same SUMIFS with "BOB")
- `B20` — BOB Received: (same ABS SUMIFS with "BOB")
- `B21` — BOB Opening Balance: **entered manually each month**
- `B22` — BOB Closing Balance: `=B21-B19+B20`

### Month-end Process (manual, ~30 seconds)
1. Type new month in `B1` e.g. `April 2026`
2. Copy closing balances from `B12`, `B17`, `B22` and paste as values into `B11`, `B16`, `B21`

### Important Notes
- Google Sheets pivot tables do NOT auto-refresh when new data is added — manual refresh required
- The SUMIFS-based dashboard updates automatically as new rows are added — preferred over pivot tables
- All SUMIFS formulas use real date comparisons, so dates must be in MM/DD/YYYY format (not text)

---

## Bank SMS Senders
| Bank | Sender IDs | Status |
|---|---|---|
| Jupiter (Federal Bank) | FEDBNK-S, FEDBNK-T | ✅ Live |
| Kotak 811 | KOTAKB-S, VK-KOTAKB-S, AD-KOTAKB-S | ✅ Live |
| Bank of Baroda | BOBSMS-S, BOBTXN-S + prefixed variants | ✅ Live |

### Kotak SMS Formats
1. `Received Rs.X in your Kotak Bank AC X5858 from ... on DD-MM-YY` → credit
2. `Rs.X spent via Kotak Debit Card XX9051 at ... on DD/MM/YYYY` → debit
3. `Sent Rs.X from Kotak Bank AC X5858 to ... on DD-MM-YY` → debit

### BOB SMS Formats
1. `Rs.X credited to A/C xx3092 on DD-MM-YY` → credit (BOBSMS-S)
2. `Rs.X Dr. from A/C XXXXXX3092 ...` → debit (BOBSMS-S)
3. `Rs.X Credited to A/C ...3092 thru NEFT ... DD-MM-YYYY` → credit (BOBTXN-S)

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

Note: "Other" button in Telegram has callback data `RD` → maps to Random.

---

## Known Issues / Quirks
- SMS text in `chat.db` is stored in `attributedBody` binary blob, not plain `text` column
- Federal Bank SMS always starts with "Rs" — decoder uses `\x00Rs` marker
- Sender normalisation uses `endswith` matching to handle prefixed IDs like `VK-KOTAKB-S`, `JK-BOBTXN-S`
- Credits stored as negative numbers in the sheet
- Dates written as MM/DD/YYYY so Google Sheets treats them as real dates (not text)
- n8n webhook data lands in a `body` wrapper: field paths are `$json.body.sender` etc.
- macOS Full Disk Access must be granted to `/opt/anaconda3/bin/python3.12`
- iCloud Messages must be enabled on Mac for SMS sync from iPhone
- Mac must not sleep — disable in System Settings → Battery → Prevent Mac from sleeping automatically
- Locking Mac (password screen) is fine — script keeps running. Sleep stops it.
- launchd manages the Python script — Terminal does not need to be open

---

## How to Start / Manage

### Python script
```bash
# Check if running
launchctl list | grep jupytertracker

# Restart
launchctl unload ~/Library/LaunchAgents/com.hemant.jupytertracker.plist
launchctl load ~/Library/LaunchAgents/com.hemant.jupytertracker.plist

# View logs
tail -f ~/Library/Logs/jupytertracker.log
```

### n8n
- Always on at Railway — no action needed
- Access at: `https://n8n-production-0e8d.up.railway.app`

---

## What's Next

### Dashboard UI
The current Dashboard sheet is functional but plain. The next focus is building a **beautiful, informative dashboard** directly in Google Sheets using **Google Apps Script**.

Rather than manually formatting cells, Apps Script will programmatically generate the entire dashboard layout — colors, fonts, borders, charts, and formulas — in one shot.

The dashboard should:
- Show the current month automatically at the top
- Display balances for Jupiter, Kotak, and BOB clearly
- Show spent vs received vs closing balance per bank
- Be visually clean and easy to read at a glance
- Allow switching to any past month via the `B1` month selector
- Show a category breakdown (where did the money go this month)
- Show a chart or visual indicator of spend vs opening balance per bank

All data lives in `2026 Consolidated Jupiter`. No external services — everything stays inside Google Sheets.

**The next chat session should focus entirely on writing the Google Apps Script to build this dashboard beautifully and automatically.**
