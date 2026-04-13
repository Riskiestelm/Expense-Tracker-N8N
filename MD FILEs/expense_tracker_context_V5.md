# Expense Tracker — Full Build Context V6

## What We Built
Automated personal expense tracking for Jupiter (Federal Bank), Kotak 811, and Bank of Baroda transactions, with a Google Sheets dashboard that auto-switches between months.

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
- ✅ Telegram inline button flow working (updated categories — see below)
- ✅ Google Sheet logging (category, description, amount, date, bank, type)
- ✅ Credits stored as negative amounts (handled in n8n Node 5)
- ✅ Dates written as MM/DD/YYYY (real date format, not text)
- ✅ Dashboard with month dropdown — switch months instantly
- ✅ Opening balance table with auto-cascading formulas (only March entered manually)
- ✅ Category breakdown table with SUMIFS per category per month
- ✅ Pie chart showing spending by category for selected month
- ✅ Complete SETUP_GUIDE.md written for GitHub/Reddit (public-facing, step-by-step for anyone)

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
- Data sheet: `2026 Consolidated` (renamed from `2026 Consolidated Jupiter`)
- Dashboard sheet: `Dashboard`
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
- Inline Keyboard (UPDATED — reduced and renamed categories):
  - Row 1: 🛒 Groceries (GR), 🍔 Food (FD), 🏢 Flat expenses (FE)
  - Row 2: 🛍 Shopping (SH), 🥤 Beverages (BV), 🎬 Entertainment (EN)
  - Row 3: 🚗 Transport (TP), 💳 CC + previous dues (CC), 💰 Investment (IN)
  - Row 4: 💈 Haircut (HC), 🔄 Refund (RF), ⛽ Petrol (PE), 📝 Other (RD)

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

// UPDATED category map — reduced set
const categoryMap = {
  FD: 'Food',
  GR: 'Groceries',
  FE: 'Flat expenses',
  SH: 'Shopping',
  BV: 'Beverages',
  EN: 'Entertainment',
  TP: 'Transport',
  CC: 'CC + previous dues',
  IN: 'Investment',
  HC: 'Haircut',
  RF: 'Refund',
  PE: 'Petrol & other expenses',
  RD: 'Random'
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
- `A3` / `B3` — Label "Start Date:" and formula:
  `=DATE(VALUE(RIGHT(B1,4)),MATCH(LEFT(B1,FIND(" ",B1)-1),{"January","February","March","April","May","June","July","August","September","October","November","December"},0),1)`
- `A4` / `B4` — Label "End Date:" and formula: `=EOMONTH(B3,0)`

Changing the dropdown instantly updates all balances, spending, categories, and the pie chart.

### Per-Bank Rows
For each bank: Spent, Received, Opening Balance (auto from table), Closing Balance.

**Jupiter:**
- `B7` — Jupiter Spent: `=SUMIFS('2026 Consolidated'!C:C,'2026 Consolidated'!E:E,"Jupiter",'2026 Consolidated'!D:D,">="&$B$3,'2026 Consolidated'!D:D,"<="&$B$4,'2026 Consolidated'!F:F,"debit")`
- `B8` — Jupiter Received: `=ABS(SUMIFS('2026 Consolidated'!C:C,'2026 Consolidated'!E:E,"Jupiter",'2026 Consolidated'!D:D,">="&$B$3,'2026 Consolidated'!D:D,"<="&$B$4,'2026 Consolidated'!F:F,"credit"))`
- `B9` — Jupiter Op Bal: `=INDEX(I:I,MATCH(B1,H:H,0))`
- `B10` — Jupiter Current Bal: `=B9-B7+B8`

**Kotak:**
- `B12` — Kotak Spent: (same SUMIFS with "Kotak")
- `B13` — Kotak Received: (same ABS SUMIFS with "Kotak")
- `B14` — Kotak Op Bal: `=INDEX(J:J,MATCH(B1,H:H,0))`
- `B15` — Kotak Current Bal: `=B14-B12+B13`

**BOB:**
- `B17` — BOB Spent: (same SUMIFS with "BOB")
- `B18` — BOB Received: (same ABS SUMIFS with "BOB")
- `B19` — BOB Op Bal: `=INDEX(K:K,MATCH(B1,H:H,0))`
- `B20` — BOB Current Bal: `=B19-B17+B18`

### Opening Balance Table (columns H–M)

Auto-cascading table — only the first month (March 2026) is entered manually. Every subsequent month's opening balance is calculated from the previous month's closing balance.

| Column | Content |
|---|---|
| H | Month (March 2026 through Feb 2027+) |
| I | Jupiter opening balance |
| J | Kotak opening balance |
| K | BOB opening balance |
| L | Start date (helper): `=DATE(VALUE(RIGHT(H2,4)),MATCH(LEFT(H2,FIND(" ",H2)-1),{"January",...},0),1)` |
| M | End date (helper): `=EOMONTH(L2,0)` |

**Row 2 (March 2026):** I2, J2, K2 are manually entered opening balances.

**Row 3 onward (April 2026+):** Each cell uses formula:
```
=I2-SUMIFS('2026 Consolidated'!C:C,'2026 Consolidated'!E:E,"Jupiter",'2026 Consolidated'!D:D,">="&L2,'2026 Consolidated'!D:D,"<="&M2,'2026 Consolidated'!F:F,"debit")+ABS(SUMIFS('2026 Consolidated'!C:C,'2026 Consolidated'!E:E,"Jupiter",'2026 Consolidated'!D:D,">="&L2,'2026 Consolidated'!D:D,"<="&M2,'2026 Consolidated'!F:F,"credit"))
```
(Same pattern for Kotak column J and BOB column K, changing the bank name filter.)

Each row references the **row above** for opening balance and its **own row's L/M** for date range. The chain cascades from March through all future months.

**Important:** Month values in column H must have a space between month name and year (e.g., `March 2026` not `March2026`). The date formula uses `FIND(" ",...)` to parse the month name.

### Category Breakdown Table (columns D–E)

| D | E |
|---|---|
| Category | Amount |
| Groceries | `=SUMIFS('2026 Consolidated'!C:C,'2026 Consolidated'!A:A,D2,'2026 Consolidated'!D:D,">="&$B$3,'2026 Consolidated'!D:D,"<="&$B$4,'2026 Consolidated'!F:F,"debit")` |
| Flat expenses | *(same formula, drag down)* |
| Food | ... |
| Beverages | ... |
| Shopping | ... |
| Petrol & other expenses | ... |
| Entertainment | ... |
| Transport | ... |
| CC + previous dues | ... |
| Investment | ... |
| Haircut | ... |
| Refund | `=SUMIFS('2026 Consolidated'!C:C,'2026 Consolidated'!A:A,D13,'2026 Consolidated'!D:D,">="&$B$3,'2026 Consolidated'!D:D,"<="&$B$4)` ← NO "debit" filter, picks up credits too |
| Random | *(same as other categories with debit filter)* |
| Total | `=SUM(E2:E14)` |

**Important:** Category names in column D must exactly match what's stored in column A of `2026 Consolidated`. Any mismatch (extra space, different casing) breaks the SUMIFS.

**Refund exception:** The Refund row formula omits the `F:F,"debit"` filter so it picks up negative (credit) amounts. This makes refunds reduce the total spending.

### Pie Chart
- Data range: D1:E14 (excludes Total row)
- Type: Pie chart
- Slice labels: Percentage
- Auto-updates when month changes or new transactions land
- Google Sheets automatically hides zero-value slices

### Month-End Process
**None.** Just switch the dropdown. Everything auto-updates.

---

## Category Codes (UPDATED — reduced set)

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

**Removed categories** (consolidated into above): Rent, Cook, Maid, Purifier, Wifi, Recharge, Electricity → merged into "Flat expenses". Petrol, Trip, Gaadi → merged into "Petrol & other expenses". Credit Card, Previous Dues → merged into "CC + previous dues". Kotak category removed.

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
- Category names in Dashboard column D must **exactly match** column A in consolidated sheet (case-sensitive, no extra spaces)
- Month values in opening balance table must have a space between month and year (`March 2026` not `March2026`)
- Refund formula in Dashboard must NOT have the "debit" filter — it needs to pick up negative credit amounts

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

## Deliverables Created

### SETUP_GUIDE.md
A comprehensive public-facing setup guide written for GitHub/Reddit. Contains every step needed for anyone on the internet to replicate this entire system on their own device:
- Google Sheets setup (structure, APIs, OAuth)
- Telegram bot creation
- n8n deployment on Railway (both workflows with full code)
- Python script (full code with comments)
- launchd auto-start
- Dashboard with dropdown, opening balance table, category breakdown, pie chart
- Customization guide (adding banks, changing categories, non-Indian users)
- Troubleshooting section
- Architecture diagram

---

## What's Next
- Potential: Visual formatting/beautification of the dashboard (colors, fonts, borders) — possibly via Google Apps Script
- Potential: Add more banks as needed
- Potential: Yearly rollover (new consolidated sheet for 2027, extend opening balance table)
