# Expense Tracker — Setup Instructions

Automated personal expense tracking. Every bank transaction triggers a Telegram notification with category buttons. One tap categorizes it. Everything logs to Google Sheets automatically.

**~3 hour setup. Free to run (Railway Hobby plan ~$5/month).**

---

## How It Works

```
Bank SMS → parsed → Telegram notification with category buttons
→ tap a category → type a description → logged to Google Sheets
```

Two ways to get the SMS into the system:

| Path | Requirements | Autonomy |
|---|---|---|
| **Mac** (preferred) | Mac on same Apple ID as iPhone | Fully automatic, zero interaction |
| **iPhone only** | Just your iPhone | Fully automatic via iOS Shortcuts |

Pick your path. Everything after the SMS entry point is identical.

---

## Prerequisites

- iPhone with bank accounts that send SMS transaction alerts
- Telegram account
- Google account
- Railway account (Hobby plan ~$5/mo to keep n8n running 24/7)
- Mac (optional — removes need for iOS Shortcuts entirely)

---

## Part 1 — Google Sheets

### 1.1 Create the sheet
1. Create a new Google Sheet (e.g. name it `Finance tally`)
2. Create a tab called `2026 Consolidated`
3. Add these headers in row 1:

| A | B | C | D | E | F |
|---|---|---|---|---|---|
| Expense and receivable name | Description of the expense | Amount | Date | Bank | Type |

### 1.2 Enable Google APIs
1. Go to [console.cloud.google.com](https://console.cloud.google.com)
2. Create a new project
3. Enable **Google Sheets API** and **Google Drive API**
4. Go to **APIs & Services → OAuth consent screen**
   - App name: anything
   - Under **Audience**, click **Publish App** — this prevents OAuth tokens from expiring every 7 days
5. Go to **Credentials → Create Credentials → OAuth Client ID**
   - Application type: Web application
   - Authorized redirect URI: `https://your-n8n-url.up.railway.app/rest/oauth2-credential/callback`
   - Save the Client ID and Client Secret

---

## Part 2 — Telegram Bot

1. Open Telegram → search `@BotFather` → send `/newbot`
2. Follow the prompts, save the **bot token**
3. Start a chat with your new bot, send any message
4. Visit `https://api.telegram.org/bot<YOUR_TOKEN>/getUpdates` — find the `id` field under `chat`. That's your chat ID.

---

## Part 3 — n8n on Railway

### 3.1 Deploy n8n
1. [railway.app](https://railway.app) → New Project → Deploy a Template → search **n8n** → select n8n + Postgres → Deploy
2. Once deployed, open Postgres service → Settings → reduce to 0.5 vCPU / 512MB RAM (saves cost)
3. Note your n8n URL (e.g. `https://n8n-production-xxxx.up.railway.app`)
4. Open n8n, create an account

### 3.2 Connect Google Sheets
1. n8n → Credentials → New → Google Sheets OAuth2
2. Paste your Client ID and Client Secret
3. Connect and authorize

### 3.3 Create Workflow A — Send

**Node 1: Webhook**
- Method: POST
- Respond: When Last Node Finishes
- Save the webhook URL — you'll need it for the Shortcut or Python script

**Node 2: Code (Parse SMS)**
```javascript
const body = $json.body;

let msg, amount, type, bank, date;

if (body.amount) {
  // Mac path — Python script already parsed
  msg = body.message;
  amount = body.amount;
  type = body.type;
  bank = body.sender;
  date = body.date || '';
} else {
  // iOS path — raw message, parse here
  msg = body.message || '';
  const senderRaw = body.sender || '';

  // Amount — handles Rs, Rs., INR
  const amountMatch = msg.match(/(?:Rs\.?\s*|INR\s*)([\d,]+(?:\.\d{1,2})?)/i);
  amount = amountMatch ? amountMatch[1].replace(/,/g, '') : null;

  // Bank — specific matches must come before generic ones
  if (msg.toLowerCase().includes('kotak credit') || msg.toLowerCase().includes('kotak cc')) {
    bank = 'Kotak CC';
  } else if (senderRaw.includes('KOTAK') || msg.toLowerCase().includes('kotak')) {
    bank = 'Kotak';
  } else if (senderRaw.includes('FEDBNK') || msg.includes('Federal Bank')) {
    bank = 'Jupiter';
  } else if (senderRaw.includes('BOB') || msg.includes('Baroda')) {
    bank = 'BOB';
  } else {
    bank = 'Unknown';
  }

  // Type
  type = /received|credited/i.test(msg) ? 'credit' : 'debit';

  // Date — handles DD-MM-YYYY, DD/MM/YYYY, DD-MM-YY, DD-Mon-YYYY (e.g. 12-Apr-2026)
  const monthMap = {jan:'01',feb:'02',mar:'03',apr:'04',may:'05',jun:'06',jul:'07',aug:'08',sep:'09',oct:'10',nov:'11',dec:'12'};
  let dateMatch = msg.match(/(\d{2})[\/\-]([A-Za-z]{3})[\/\-](\d{4})/);
  if (dateMatch) {
    date = `${dateMatch[1]}-${monthMap[dateMatch[2].toLowerCase()]}-${dateMatch[3]}`;
  } else {
    dateMatch = msg.match(/(\d{2}[\/\-]\d{2}[\/\-]\d{2,4})/);
    date = dateMatch ? dateMatch[1] : '';
  }
}

return [{ json: { amount, type, bank, date, message: msg } }];
```

**Node 3: Telegram — Send Message**
- Credential: your Telegram bot
- Chat ID: your chat ID
- Parse Mode: HTML
- Message:
```
💳 New Transaction Detected

Bank: {{ $json.bank }}
Amount: ₹{{ $json.amount }}
Type: {{ $json.type }}
Date: {{ $json.date }}
SMS: {{ $json.message }}

Select a category:
```
- Inline Keyboard:
  - Row 1: 🛒 Groceries → `GR` | 🍔 Food → `FD` | 🏢 Flat expenses → `FE`
  - Row 2: 🛍 Shopping → `SH` | 🥤 Beverages → `BV` | 🎬 Entertainment → `EN`
  - Row 3: 🚗 Transport → `TP` | 💳 CC + previous dues → `CC` | 💰 Investment → `IN`
  - Row 4: 💈 Haircut → `HC` | 🔄 Refund → `RF` | ⛽ Petrol → `PE` | 📝 Other → `RD`

Publish the workflow.

### 3.4 Create Workflow B — Receive

**Node 1: Telegram Trigger**
- Trigger On: Callback Query, Message

Register the Telegram webhook:
```
https://api.telegram.org/bot<YOUR_TOKEN>/setWebhook?url=<YOUR_N8N_URL>/webhook/<WORKFLOW_B_PATH>/webhook
```

**Node 2: Switch**
- Condition: `{{ $json.callback_query ? 'callback' : 'message' }}`
- Output 0: equals `callback`
- Output 1: equals `message`

**Node 3: Code — Save transaction** (Switch output 0)
```javascript
const data = $input.all()[0].json;
const staticData = $getWorkflowStaticData('global');
staticData.pendingTransaction = {
  category: data.callback_query.data,
  amount: data.callback_query.message.text.match(/Amount: ₹([\d.]+)/)?.[1],
  type: data.callback_query.message.text.match(/Type: (\w+)/)?.[1],
  date: data.callback_query.message.text.match(/Date: (\d{2}-\d{2}-\d{4})/)?.[1] || '',
  rawDate: data.callback_query.message.text.match(/on (\d{2}-\d{2}-\d{4}|\d{2}[A-Z]{3}\d{2})/i)?.[1] || '',
  bank: data.callback_query.message.text.match(/Bank: (.+)/)?.[1],
  chat_id: data.callback_query.message.chat.id
};
return $input.all();
```

**Node 4: Telegram — Send Message** (connected to Node 3)
- Chat ID: `{{ $json.callback_query.message.chat.id }}`
- Text: `Got it! Now send a description, or send - to skip.`

**Node 5: Code — Retrieve transaction** (Switch output 1)
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

**Node 6: Google Sheets — Append Row** (connected to Node 5)
- Sheet: `2026 Consolidated`
- A: `{{ $json.category }}`, B: `{{ $json.description }}`, C: `{{ $json.amount }}`, D: `{{ $json.date }}`, E: `{{ $json.bank }}`, F: `{{ $json.type }}`

Publish the workflow.

---

## Part 4 — SMS Entry Point

### Option A: iPhone Only (iOS Shortcuts)

You need **two shortcuts** — one for `Rs` format banks, one for `INR` format banks.

**For each shortcut:**
1. Shortcuts app → Automation → New Automation
2. Trigger: **Message** → contains `Rs` (Shortcut 1) or `INR` (Shortcut 2)
3. Add action: **Get Contents of URL**
   - URL: your n8n Workflow A webhook URL
   - Method: POST
   - Request Body: JSON
     - `message` → Shortcut Input (Content)
     - `sender` → Shortcut Input (Sender)
4. Enable **"Run after confirmation immediately"** in automation settings — makes it fully automatic, no tap required
5. Save

**Bank-specific notes:**
- Jupiter uses `Rs` without a dot — Shortcut 1 trigger must be `Rs` not `Rs.`
- Kotak Credit Card uses `INR` — covered by Shortcut 2
- The n8n Code node detects which bank sent the SMS from message content automatically
- If your bank uses a different amount format (e.g. `USD`, `EUR`), add a third shortcut with that trigger

### Option B: Mac (fully autonomous, no Shortcuts needed)

Requirements: Mac on the same Apple ID as your iPhone, iCloud Messages enabled, Mac never sleeps.

**Enable iCloud Messages:** Messages app → Settings → iMessage → Enable Messages in iCloud

**Grant Full Disk Access to Python:** System Settings → Privacy & Security → Full Disk Access → add your Python binary (use `which python3.12` to find the path)

**Disable sleep:** System Settings → Battery → Prevent Mac from sleeping automatically

**Python script** — save as `~/expense-tracker/tracker.py`, replace `WEBHOOK_URL` with your n8n webhook URL:

```python
import sqlite3, json, time, re, requests
from pathlib import Path

WEBHOOK_URL = "https://your-n8n-url.up.railway.app/webhook/YOUR_PATH"
DB_PATH = Path.home() / "Library/Messages/chat.db"
STATE_FILE = Path.home() / ".expense_tracker_state.json"
POLL_INTERVAL = 10

TRACKED_SENDERS = {"FEDBNK-S", "FEDBNK-T", "KOTAKB-S", "BOBSMS-S", "BOBTXN-S"}

def load_state():
    if STATE_FILE.exists():
        with open(STATE_FILE) as f:
            return set(json.load(f))
    return set()

def save_state(ids):
    with open(STATE_FILE, "w") as f:
        json.dump(list(ids), f)

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
        txn_type = "debit" if "debited" in text_lower else "credit" if "credited" in text_lower else None
        if not txn_type:
            return None
        return {"sender": "Jupiter", "amount": amount, "type": txn_type, "date": "", "message": text}
    elif normalised == "KOTAKB-S":
        if text_lower.startswith("received"):
            txn_type = "credit"
        elif "spent via kotak debit card" in text_lower or text_lower.startswith("sent"):
            txn_type = "debit"
        else:
            return None
        date_match = re.search(r'(\d{2}[/-]\d{2}[/-]\d{2,4})', text)
        date = normalise_date(date_match.group(1)) if date_match else ""
        return {"sender": "Kotak", "amount": amount, "type": txn_type, "date": date, "message": text}
    elif normalised in ("BOBSMS-S", "BOBTXN-S"):
        txn_type = "credit" if "credited" in text_lower else "debit" if "dr." in text_lower else None
        if not txn_type:
            return None
        date_match = re.search(r'(\d{2}[/-]\d{2}[/-]\d{2,4})', text)
        date = normalise_date(date_match.group(1)) if date_match else ""
        return {"sender": "BOB", "amount": amount, "type": txn_type, "date": date, "message": text}
    return None

def fetch_new_messages(processed_ids):
    if not DB_PATH.exists():
        return []
    try:
        conn = sqlite3.connect(f"file:{DB_PATH}?mode=ro", uri=True)
        cursor = conn.cursor()
        cursor.execute("""
            SELECT message.ROWID, handle.id, message.text, message.attributedBody
            FROM message JOIN handle ON message.handle_id = handle.ROWID
            WHERE message.is_from_me = 0
            ORDER BY message.date DESC LIMIT 200
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
        print(f"[ERROR] {e}")
        return []

def main():
    print("Expense Tracker started...")
    processed_ids = load_state()
    while True:
        for msg_id, sender, text in fetch_new_messages(processed_ids):
            parsed = parse_sms(sender, text)
            if parsed:
                try:
                    r = requests.post(WEBHOOK_URL, json=parsed, timeout=10)
                    if r.status_code == 200:
                        print(f"[OK] ₹{parsed['amount']} {parsed['type']} via {parsed['sender']}")
                        processed_ids.add(msg_id)
                        save_state(processed_ids)
                except Exception as e:
                    print(f"[ERROR] {e}")
            else:
                processed_ids.add(msg_id)
                save_state(processed_ids)
        time.sleep(POLL_INTERVAL)

if __name__ == "__main__":
    main()
```

**Auto-start on login** — save as `~/Library/LaunchAgents/com.expense.tracker.plist`:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key><string>com.expense.tracker</string>
    <key>ProgramArguments</key>
    <array>
        <string>/opt/anaconda3/bin/python3.12</string>
        <string>/Users/YOUR_USERNAME/expense-tracker/tracker.py</string>
    </array>
    <key>RunAtLoad</key><true/>
    <key>KeepAlive</key><true/>
    <key>StandardOutPath</key><string>/Users/YOUR_USERNAME/Library/Logs/expense-tracker.log</string>
    <key>StandardErrorPath</key><string>/Users/YOUR_USERNAME/Library/Logs/expense-tracker.error.log</string>
</dict>
</plist>
```

Replace `YOUR_USERNAME` and the Python path. Then:
```bash
launchctl load ~/Library/LaunchAgents/com.expense.tracker.plist
```

**Important:** Use the exact Python binary that has Full Disk Access. Generic `python3` often fails with launchd.

---

## Part 5 — Dashboard (optional)

### Month selector
- `B1`: Data validation dropdown from range `H2:H13`
- `B3`: `=DATE(VALUE(RIGHT(B1,4)),MATCH(LEFT(B1,FIND(" ",B1)-1),{"January","February","March","April","May","June","July","August","September","October","November","December"},0),1)`
- `B4`: `=EOMONTH(B3,0)`

### Per-bank balance rows (example for Jupiter)
- Spent: `=SUMIFS('2026 Consolidated'!C:C,'2026 Consolidated'!E:E,"Jupiter",'2026 Consolidated'!D:D,">="&$B$3,'2026 Consolidated'!D:D,"<="&$B$4,'2026 Consolidated'!F:F,"debit")`
- Received: `=ABS(SUMIFS('2026 Consolidated'!C:C,'2026 Consolidated'!E:E,"Jupiter",'2026 Consolidated'!D:D,">="&$B$3,'2026 Consolidated'!D:D,"<="&$B$4,'2026 Consolidated'!F:F,"credit"))`
- Opening Balance: `=INDEX(I:I,MATCH(B1,H:H,0))`
- Current Balance: `=Opening - Spent + Received`

Replicate for Kotak, Kotak CC, and BOB.

### Opening balance cascade table (columns H–M)
- H: Month names (`March 2026`, `April 2026` etc. — must have a space between month and year)
- I/J/K: Opening balances per bank — first month entered manually, rest use formula cascading from previous month
- L/M: Start and end date helpers

### Category breakdown
- Column D: category names — must exactly match column A of data sheet
- Column E: `=SUMIFS('2026 Consolidated'!C:C,'2026 Consolidated'!A:A,D2,'2026 Consolidated'!D:D,">="&$B$3,'2026 Consolidated'!D:D,"<="&$B$4,'2026 Consolidated'!F:F,"debit")`
- Refund row: omit `F:F,"debit"` filter

Add a pie chart with data range covering your category table.

---

## Troubleshooting

**Telegram message not arriving**
- Check Workflow A is Published in n8n
- Verify the webhook URL in your Shortcut matches Workflow A's production URL

**Bank shows as Unknown**
- Message content doesn't contain a recognized keyword
- Add your bank's identifying phrase to the bank detection block in the Code node

**Amount is null**
- Your bank uses a different currency prefix — add it to the amount regex: `(?:Rs\.?\s*|INR\s*|YOUR_PREFIX\s*)`

**Wrong type (credit/debit)**
- Your bank uses different keywords — add them: `/received|credited|YOUR_KEYWORD/i`

**Date showing blank**
- Check the raw SMS in n8n executions and add the date pattern to the date block

**Sheet row not written**
- Check Workflow B is Published
- Re-run the Telegram setWebhook curl command
- Reconnect Google Sheets OAuth if token expired

**n8n Google Sheets token keeps expiring**
- Google Cloud Console → OAuth consent screen → Audience → Publish App

**iOS Shortcut not triggering**
- Confirm "Run after confirmation immediately" is enabled
- Jupiter uses `Rs` without dot — trigger must be `Rs` not `Rs.`
- Kotak Credit Card uses `INR` — needs its own shortcut

---

## Customization

**Adding a new bank:**
1. Add its identifying keywords to the bank detection block in the n8n Code node
2. Add its credit/debit keywords to the type regex if different
3. Add a new Shortcut if it uses a different amount format prefix

**Adding/changing categories:**
1. Update the inline keyboard in Workflow A Node 3
2. Update `categoryMap` in Workflow B Node 5
3. Update the category list in Dashboard column D

**Non-Indian banks:**
Change the amount regex to match your currency symbol. Everything else is generic.
