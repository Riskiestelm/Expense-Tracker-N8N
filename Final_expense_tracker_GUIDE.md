# Automated Expense Tracker — Complete Setup Guide

> **Turn bank SMS notifications into an automatically categorized Google Sheet — zero manual data entry.**

This guide walks you through building a fully automated personal expense tracking system from scratch. Bank transaction SMS arrives on your phone → syncs to your Mac → gets parsed → you tap a category on Telegram → it's logged to Google Sheets with a live dashboard.

---

## Table of Contents

- [How It Works](#how-it-works)
- [Prerequisites](#prerequisites)
- [Part 1: Google Sheets Setup](#part-1-google-sheets-setup)
- [Part 2: Telegram Bot Setup](#part-2-telegram-bot-setup)
- [Part 3: n8n Setup (Railway)](#part-3-n8n-setup-railway)
- [Part 4: Python Script (Mac)](#part-4-python-script-mac)
- [Part 5: Auto-Start on Mac Login](#part-5-auto-start-on-mac-login)
- [Part 6: Dashboard Sheet](#part-6-dashboard-sheet)
- [Part 7: Testing End-to-End](#part-7-testing-end-to-end)
- [Customization Guide](#customization-guide)
- [Troubleshooting](#troubleshooting)
- [Architecture Overview](#architecture-overview)

---

## How It Works

```
Bank SMS arrives on iPhone
  → iMessage syncs to Mac (same Apple ID, iCloud Messages enabled)
  → Python script polls ~/Library/Messages/chat.db every 10 seconds
  → Parses SMS: extracts amount, credit/debit type, date, bank name
  → POSTs to n8n webhook (hosted on Railway — always on)
  → n8n sends Telegram message with inline category buttons
  → You tap a category button
  → n8n asks for a description via Telegram
  → You type a description (or send `-` to skip)
  → n8n appends a row to your Google Sheet
```

The entire flow takes about 5 seconds from tapping a button to seeing the row in your sheet.

---

## Prerequisites

Before you begin, make sure you have:

- **A Mac** (MacBook, iMac, Mac Mini — any Mac that stays awake)
- **An iPhone** with bank SMS notifications enabled
- **Same Apple ID** on both Mac and iPhone, with **iCloud Messages** turned on (Settings → Apple ID → iCloud → Messages)
- **Python 3.10+** installed on your Mac (Anaconda or Homebrew — either works)
- **A Google account** (for Google Sheets and Google Cloud)
- **A Telegram account** (free, available on all platforms)
- **A Railway account** (free tier works, Hobby plan recommended for always-on) — [railway.app](https://railway.app)

### Verify iMessage Sync

On your Mac, open the Messages app. You should see your bank SMS messages syncing from your iPhone. If they're not showing up:

1. On iPhone: Settings → Apple ID → iCloud → Messages → toggle ON
2. On Mac: Messages → Settings → iMessage → Enable Messages in iCloud
3. Wait a few minutes for sync to complete

---

## Part 1: Google Sheets Setup

### 1.1 Create the Spreadsheet

1. Go to [sheets.google.com](https://sheets.google.com)
2. Create a new spreadsheet
3. Name it whatever you want (e.g., "Finance Tally" or "Expense Tracker")
4. Rename the first sheet tab (bottom of screen) to: `2026 Consolidated`
   - Use the current year. If you're setting this up in 2027, call it `2027 Consolidated`.

### 1.2 Set Up Column Headers

In row 1 of the `2026 Consolidated` sheet, enter these headers:

| Column | Header |
|--------|--------|
| A1 | `Expense and receivable name` |
| B1 | `Description` |
| C1 | `Amount` |
| D1 | `Date` |
| E1 | `Bank` |
| F1 | `Type` |

**How data will be stored:**

- **Column A** — Category name (e.g., Food, Groceries, Rent)
- **Column B** — Description you type in Telegram (or blank if you skip)
- **Column C** — Transaction amount. Debits are positive numbers. Credits (refunds, received money) are **negative** numbers.
- **Column D** — Date in `MM/DD/YYYY` format. This must be a real date (not text) so formulas work.
- **Column E** — Bank name (e.g., Jupiter, Kotak, BOB — or whatever your banks are)
- **Column F** — Either `debit` or `credit` (lowercase)

### 1.3 Enable Google Sheets API

You need API access so n8n can write to your sheet.

1. Go to [Google Cloud Console](https://console.cloud.google.com)
2. Create a new project (name it anything, e.g., "Expense Tracker")
3. In the left sidebar, go to **APIs & Services → Library**
4. Search for and enable:
   - **Google Sheets API**
   - **Google Drive API**
5. Go to **APIs & Services → Credentials**
6. Click **Create Credentials → OAuth Client ID**
7. If prompted, configure the **OAuth Consent Screen** first:
   - User type: **External**
   - App name: anything (e.g., "Expense Tracker")
   - Add your email as a test user
   - Save
8. Back in Credentials, create an **OAuth Client ID**:
   - Application type: **Web application**
   - Name: anything
   - Authorized redirect URIs: You'll get this from n8n (see Part 3)
9. Note down the **Client ID** and **Client Secret** — you'll need them in n8n.

---

## Part 2: Telegram Bot Setup

### 2.1 Create the Bot

1. Open Telegram and search for **@BotFather**
2. Send `/newbot`
3. Choose a name (e.g., "Expense Tally Bot")
4. Choose a username (must end in `bot`, e.g., `myexpensetally_bot`)
5. BotFather will give you a **bot token** — save this. It looks like: `7123456789:AAH1234abcdef...`

### 2.2 Get Your Chat ID

1. Open your new bot in Telegram and send it any message (e.g., "hello")
2. In your browser, visit:
   ```
   https://api.telegram.org/bot<YOUR_BOT_TOKEN>/getUpdates
   ```
   Replace `<YOUR_BOT_TOKEN>` with the token from step 5 above.
3. In the JSON response, find `"chat":{"id":XXXXXXXXXX}` — that number is your **Chat ID**. Save it.

---

## Part 3: n8n Setup (Railway)

n8n is the automation engine that connects everything. It receives parsed SMS data, sends you Telegram buttons, and writes to Google Sheets.

### 3.1 Deploy n8n on Railway

1. Go to [railway.app](https://railway.app) and sign in
2. Click **New Project → Deploy a Template**
3. Search for **n8n** and deploy it
4. Railway will give you a public URL like `https://n8n-production-XXXX.up.railway.app`
5. Open that URL in your browser and set up your n8n account (email + password)

### 3.2 Connect Google Sheets in n8n

1. In n8n, go to **Settings → Credentials**
2. Click **Add Credential → Google Sheets OAuth2 API**
3. Enter your **Client ID** and **Client Secret** from Part 1
4. n8n will show you a **redirect URI** — go back to Google Cloud Console and add this to your OAuth Client's authorized redirect URIs
5. Click **Connect** in n8n — it will open Google's OAuth flow. Sign in and grant access.

### 3.3 Connect Telegram in n8n

1. In n8n, go to **Settings → Credentials**
2. Click **Add Credential → Telegram API**
3. Enter your **bot token** from Part 2

### 3.4 Create Workflow A — "Expense Tracker - Send"

This workflow receives data from the Python script and sends you a Telegram message with category buttons.

**Node 1: Webhook**

1. Add a **Webhook** node
2. Method: POST
3. It will auto-generate a URL path — note down the full webhook URL (you'll need it for the Python script)
4. Set "Respond" to: **When Last Node Finishes**

**Node 2: Telegram — Send Message**

1. Add a **Telegram** node, connect it after the Webhook
2. Operation: **Send Message**
3. Chat ID: your chat ID from Part 2
4. Parse Mode: **HTML**
5. Message text:
```
💳 New Transaction Detected

Bank: {{ $json.body.sender }}
Amount: ₹{{ $json.body.amount }}
Type: {{ $json.body.type }}
Date: {{ $json.body.date }}
SMS: {{ $json.body.message }}

Select a category:
```
6. Under **Additional Fields** → **Reply Markup** → **Inline Keyboard**

Add your category buttons. Each button has a display label and a callback data code. Here is a recommended layout:

```
Row 1: 🛒 Groceries (GR)  |  🍔 Food (FD)         |  🏢 Flat expenses (FE)
Row 2: 🛍 Shopping (SH)   |  🥤 Beverages (BV)     |  🎬 Entertainment (EN)
Row 3: 🚗 Transport (TP)  |  💳 CC + previous dues (CC)  |  💰 Investment (IN)
Row 4: 💈 Haircut (HC)    |  🔄 Refund (RF)        |  ⛽ Petrol (PE)       |  📝 Other (RD)
```

> **Customize these categories** to match your spending habits. The callback data codes (GR, FD, FE, etc.) are what get stored and mapped to full names in Workflow B.

7. **Activate** the workflow.

### 3.5 Create Workflow B — "Expense Tracker - Receive"

This workflow handles your button taps, asks for a description, and writes to Google Sheets.

**Node 1: Telegram Trigger**

1. Add a **Telegram Trigger** node
2. Trigger On: **Callback Query** and **Message**
3. Use the Telegram credential you created earlier
4. n8n will show a webhook URL — register it with Telegram:
   - Visit in your browser:
     ```
     https://api.telegram.org/bot<YOUR_BOT_TOKEN>/setWebhook?url=<N8N_TELEGRAM_WEBHOOK_URL>
     ```
   - You should see `{"ok":true,"result":true}`

**Node 2: Switch**

1. Add a **Switch** node, connect it after the Telegram Trigger
2. Add two rules:
   - **Output 0** (Callback): `{{ $json.callback_query ? 'callback' : 'message' }}` → equals → `callback`
   - **Output 1** (Message): same expression → equals → `message`

**Node 3: Code — Save Transaction (connect to Switch Output 0)**

This runs when you tap a category button. It saves the transaction details temporarily.

```javascript
const data = $input.all()[0].json;
const staticData = $getWorkflowStaticData('global');
staticData.pendingTransaction = {
  category: data.callback_query.data,
  amount: data.callback_query.message.text.match(/Amount: ₹([\d,.]+)/)?.[1]?.replace(',', ''),
  type: data.callback_query.message.text.match(/Type: (\w+)/)?.[1],
  date: data.callback_query.message.text.match(/Date: ([\d\-]+)/)?.[1] || '',
  rawDate: data.callback_query.message.text.match(/on (\d{2}-\d{2}-\d{4}|\d{2}[A-Z]{3}\d{2})/i)?.[1] || '',
  bank: data.callback_query.message.text.match(/Bank: (\w+)/)?.[1],
  chat_id: data.callback_query.message.chat.id
};
return $input.all();
```

**Node 4: Telegram — Send Message (connect to Node 3)**

1. Chat ID: `{{ $json.callback_query.message.chat.id }}`
2. Text: `Got it! Now send a description for this expense, or send - to skip.`

**Node 5: Code — Retrieve Transaction (connect to Switch Output 1)**

This runs when you send a description (or `-` to skip). It builds the final row for Google Sheets.

```javascript
const staticData = $getWorkflowStaticData('global');
const pending = staticData.pendingTransaction;
const description = $input.all()[0].json.message.text;

// Map your callback codes to full category names
// ⚠️ CUSTOMIZE THIS to match your Workflow A buttons
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

// Convert date to MM/DD/YYYY so Google Sheets treats it as a real date
function toSheetDate(raw) {
  if (!raw) return '';

  // DD-MM-YYYY → MM/DD/YYYY
  let m = raw.match(/^(\d{2})-(\d{2})-(\d{4})$/);
  if (m) return `${m[2]}/${m[1]}/${m[3]}`;

  // DD-MM-YY → MM/DD/20YY
  m = raw.match(/^(\d{2})-(\d{2})-(\d{2})$/);
  if (m) return `${m[2]}/${m[1]}/20${m[3]}`;

  // DDMMMYY (e.g., 17MAR26) → MM/DD/20YY
  m = raw.match(/(\d{2})(JAN|FEB|MAR|APR|MAY|JUN|JUL|AUG|SEP|OCT|NOV|DEC)(\d{2})/i);
  if (m) {
    const months = {
      JAN:'01', FEB:'02', MAR:'03', APR:'04', MAY:'05', JUN:'06',
      JUL:'07', AUG:'08', SEP:'09', OCT:'10', NOV:'11', DEC:'12'
    };
    return `${months[m[2].toUpperCase()]}/${m[1]}/20${m[3]}`;
  }

  return raw;
}

const rawDate = pending.date || pending.rawDate;
const date = toSheetDate(rawDate);

// Credits stored as negative so formulas can subtract them
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

**Node 6: Google Sheets — Append Row (connect to Node 5)**

1. Operation: **Append Row**
2. Document: select your spreadsheet
3. Sheet: select your consolidated sheet (e.g., `2026 Consolidated`)
4. Column mapping:

| Sheet Column | Value |
|-------------|-------|
| A - Expense and receivable name | `{{ $json.category }}` |
| B - Description | `{{ $json.description }}` |
| C - Amount | `{{ $json.amount }}` |
| D - Date | `{{ $json.date }}` |
| E - Bank | `{{ $json.bank }}` |
| F - Type | `{{ $json.type }}` |

5. **Activate** the workflow.

---

## Part 4: Python Script (Mac)

This script runs on your Mac, polls iMessage's database for new bank SMS messages, parses them, and sends the data to n8n.

### 4.1 Install Dependencies

```bash
pip install requests
```

(sqlite3 is built into Python.)

### 4.2 Grant Full Disk Access

The iMessage database (`chat.db`) is protected by macOS. Your Python binary needs Full Disk Access.

1. Find your Python binary path:
   ```bash
   which python3
   ```
   Example output: `/opt/anaconda3/bin/python3.12` or `/usr/local/bin/python3`

2. Go to **System Settings → Privacy & Security → Full Disk Access**
3. Click the `+` button
4. Navigate to your Python binary and add it
5. Toggle it ON

### 4.3 Create the Script

Create a file (e.g., `~/expense-tracker/tracker.py`) with the content below.

> **⚠️ YOU MUST CUSTOMIZE THIS SCRIPT.** The SMS parsing logic below is built for specific Indian banks (Jupiter/Federal Bank, Kotak 811, Bank of Baroda). Your banks will send differently formatted SMS messages. Read the [Customization Guide](#customization-guide) to adapt the parsing for your banks.

```python
import sqlite3
import json
import time
import re
import requests
from pathlib import Path

# ⚠️ REPLACE with your Workflow A webhook URL from n8n
WEBHOOK_URL = "https://your-n8n-instance.up.railway.app/webhook/YOUR-WEBHOOK-PATH"

DB_PATH = Path.home() / "Library/Messages/chat.db"
STATE_FILE = Path.home() / ".expense_tracker_state.json"
POLL_INTERVAL = 10  # seconds

# ⚠️ REPLACE with your bank SMS sender IDs
# To find these: open Messages app on Mac, look at the sender name/number for bank SMS
# Common format: "XX-BANKNAME" e.g., "AD-HDFCBK", "VK-KOTAKB-S", "JM-FEDBNK-S"
# We match using endswith() to handle carrier prefixes
TRACKED_SENDERS = {"FEDBNK-S", "FEDBNK-T", "KOTAKB-S", "BOBSMS-S", "BOBTXN-S"}


def load_state():
    """Load set of already-processed message IDs."""
    if STATE_FILE.exists():
        with open(STATE_FILE) as f:
            return set(json.load(f))
    return set()


def save_state(processed_ids):
    """Persist processed message IDs so we don't re-send after restart."""
    with open(STATE_FILE, "w") as f:
        json.dump(list(processed_ids), f)


def decode_attributed_body(blob):
    """
    Extract SMS text from the attributedBody binary blob in chat.db.

    iMessage stores SMS text in a binary 'attributedBody' column, NOT the
    plain 'text' column. This function searches for known byte markers and
    extracts the readable text.
    """
    if blob is None:
        return None
    try:
        # Primary method: look for \x00Rs marker (works for most bank SMS)
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

        # Fallback: look for NSString marker
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
    """Extract the rupee amount from SMS text. Returns string like '1234.50'."""
    match = re.search(r'Rs\.?\s*([\d,]+(?:\.\d{1,2})?)', text, re.IGNORECASE)
    if match:
        return match.group(1).replace(",", "")
    return None


def normalise_date(raw):
    """
    Normalise date formats to DD-MM-YYYY.
    Input:  DD/MM/YYYY or DD-MM-YY
    Output: DD-MM-YYYY
    """
    m = re.match(r'(\d{2})/(\d{2})/(\d{4})', raw)
    if m:
        return f"{m.group(1)}-{m.group(2)}-{m.group(3)}"
    m = re.match(r'(\d{2})-(\d{2})-(\d{2})$', raw)
    if m:
        return f"{m.group(1)}-{m.group(2)}-20{m.group(3)}"
    return raw


def parse_sms(sender, text):
    """
    Parse a bank SMS and return structured data.

    ⚠️ THIS IS THE MAIN FUNCTION YOU NEED TO CUSTOMIZE FOR YOUR BANKS.

    Each bank sends SMS in a different format. You need to:
    1. Identify the sender ID for your bank
    2. Figure out how to detect debit vs credit
    3. Extract the date (if present in the SMS)

    Returns a dict with: sender, amount, type, date, message
    Or None if the SMS can't be parsed.
    """
    normalised = next((s for s in TRACKED_SENDERS if sender.endswith(s)), None)
    if normalised is None:
        return None

    amount = extract_amount(text)
    if not amount:
        return None

    text_lower = text.lower()

    # ─── Jupiter (Federal Bank) ───────────────────────────────────
    # FEDBNK-S: always debit (spending alerts)
    # FEDBNK-T: can be debit or credit (transaction alerts)
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

    # ─── Kotak 811 ────────────────────────────────────────────────
    # "Received Rs.X" → credit
    # "spent via Kotak Debit Card" or "Sent Rs.X" → debit
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

    # ─── Bank of Baroda ───────────────────────────────────────────
    # "credited" → credit
    # "dr." → debit
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
    """Query chat.db for new bank SMS messages."""
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
    """POST parsed transaction data to the n8n webhook."""
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
                # Mark as processed even if we can't parse it
                # (avoids retrying non-transaction SMS every 10 seconds)
                processed_ids.add(msg_id)
                save_state(processed_ids)

        time.sleep(POLL_INTERVAL)


if __name__ == "__main__":
    main()
```

### 4.4 Test the Script Manually

```bash
python3 ~/expense-tracker/tracker.py
```

You should see:
```
Expense Tracker started. Polling every 10 seconds...
Watching: FEDBNK-S, FEDBNK-T, KOTAKB-S, BOBSMS-S, BOBTXN-S
Press Ctrl+C to stop.
```

If there are existing bank SMS messages in your iMessage history, you'll see them being processed. Make a small test transaction (e.g., UPI ₹1 to a friend) to verify the full flow.

---

## Part 5: Auto-Start on Mac Login

You don't want to manually run the script every time you restart your Mac. A launchd plist makes it start automatically on login and restart if it crashes.

### 5.1 Create the Plist File

Create: `~/Library/LaunchAgents/com.expensetracker.plist`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.expensetracker</string>

    <key>ProgramArguments</key>
    <array>
        <!-- ⚠️ REPLACE with YOUR Python path (run: which python3) -->
        <string>/opt/anaconda3/bin/python3.12</string>
        <!-- ⚠️ REPLACE with YOUR script path -->
        <string>/Users/YOURUSERNAME/expense-tracker/tracker.py</string>
    </array>

    <key>RunAtLoad</key>
    <true/>

    <key>KeepAlive</key>
    <true/>

    <key>StandardOutPath</key>
    <string>/Users/YOURUSERNAME/Library/Logs/expensetracker.log</string>

    <key>StandardErrorPath</key>
    <string>/Users/YOURUSERNAME/Library/Logs/expensetracker.error.log</string>
</dict>
</plist>
```

> Replace `YOURUSERNAME` with your macOS username. Find it by running `whoami` in Terminal.

### 5.2 Load and Start

```bash
launchctl load ~/Library/LaunchAgents/com.expensetracker.plist
```

### 5.3 Verify It's Running

```bash
launchctl list | grep expensetracker
```

You should see a line with a PID (process ID), meaning it's running.

### 5.4 Useful Commands

```bash
# Stop the script
launchctl unload ~/Library/LaunchAgents/com.expensetracker.plist

# Restart
launchctl unload ~/Library/LaunchAgents/com.expensetracker.plist
launchctl load ~/Library/LaunchAgents/com.expensetracker.plist

# View live logs
tail -f ~/Library/Logs/expensetracker.log

# View error logs
tail -f ~/Library/Logs/expensetracker.error.log
```

### 5.5 Prevent Mac from Sleeping

The script needs your Mac to be awake (not sleeping) to poll for new messages.

1. Go to **System Settings → Battery** (or Energy Saver on older Macs)
2. Enable **Prevent your Mac from automatically sleeping when the display is off**

> **Note:** Locking your Mac (password screen) is fine — the script keeps running. It's only *sleep mode* that stops it.

---

## Part 6: Dashboard Sheet

Build a dashboard in the same spreadsheet to see your spending at a glance.

### 6.1 Create the Dashboard Sheet

1. In your spreadsheet, click `+` at the bottom to add a new sheet
2. Name it `Dashboard`

### 6.2 Month Selector

| Cell | Content | What It Does |
|------|---------|-------------|
| A1 | `Month:` | Label |
| B1 | `March 2026` | Type the month + year you want to view. Change this to switch months. |
| A3 | `Start Date:` | Label |
| B3 | `=DATE(VALUE(RIGHT(B1,4)),MATCH(LEFT(B1,FIND(" ",B1)-1),{"January","February","March","April","May","June","July","August","September","October","November","December"},0),1)` | Auto-calculates the first day of the month in B1 |
| A4 | `End Date:` | Label |
| B4 | `=EOMONTH(B3,0)` | Auto-calculates the last day of the month in B1 |

### 6.3 Per-Bank Balances

Set up a section for each of your banks. The example below uses three banks — adjust for however many you have.

**Jupiter (starting at row 7):**

| Cell | Content |
|------|---------|
| A7 | `Jupiter Spent:` |
| B7 | `=SUMIFS('2026 Consolidated'!C:C,'2026 Consolidated'!E:E,"Jupiter",'2026 Consolidated'!D:D,">="&$B$3,'2026 Consolidated'!D:D,"<="&$B$4,'2026 Consolidated'!F:F,"debit")` |
| A8 | `Jupiter Received:` |
| B8 | `=ABS(SUMIFS('2026 Consolidated'!C:C,'2026 Consolidated'!E:E,"Jupiter",'2026 Consolidated'!D:D,">="&$B$3,'2026 Consolidated'!D:D,"<="&$B$4,'2026 Consolidated'!F:F,"credit"))` |
| A9 | `Jupiter Op Bal:` |
| B9 | *(Enter your opening balance manually — check your bank app for the balance at the start of the month)* |
| A10 | `Jupiter Current Bal:` |
| B10 | `=B9-B7+B8` |

**Kotak (starting at row 12):**

| Cell | Content |
|------|---------|
| A12 | `Kotak Spent:` |
| B12 | `=SUMIFS('2026 Consolidated'!C:C,'2026 Consolidated'!E:E,"Kotak",'2026 Consolidated'!D:D,">="&$B$3,'2026 Consolidated'!D:D,"<="&$B$4,'2026 Consolidated'!F:F,"debit")` |
| A13 | `Kotak Received:` |
| B13 | `=ABS(SUMIFS('2026 Consolidated'!C:C,'2026 Consolidated'!E:E,"Kotak",'2026 Consolidated'!D:D,">="&$B$3,'2026 Consolidated'!D:D,"<="&$B$4,'2026 Consolidated'!F:F,"credit"))` |
| A14 | `Kotak Op Bal:` |
| B14 | *(Manual — opening balance)* |
| A15 | `Kotak Current Bal:` |
| B15 | `=B14-B12+B13` |

**BOB (starting at row 17):**

| Cell | Content |
|------|---------|
| A17 | `BOB Spent:` |
| B17 | `=SUMIFS('2026 Consolidated'!C:C,'2026 Consolidated'!E:E,"BOB",'2026 Consolidated'!D:D,">="&$B$3,'2026 Consolidated'!D:D,"<="&$B$4,'2026 Consolidated'!F:F,"debit")` |
| A18 | `BOB Received:` |
| B18 | `=ABS(SUMIFS('2026 Consolidated'!C:C,'2026 Consolidated'!E:E,"BOB",'2026 Consolidated'!D:D,">="&$B$3,'2026 Consolidated'!D:D,"<="&$B$4,'2026 Consolidated'!F:F,"credit"))` |
| A19 | `BOB Op Bal:` |
| B19 | *(Manual — opening balance)* |
| A20 | `BOB Current Bal:` |
| B20 | `=B19-B17+B18` |

> **Opening Balance**: At the start of each month, check each bank app and enter the balance manually. The "Current Bal" formula does the rest.

### 6.4 Category Breakdown Table

This table totals spending per category for the selected month. It drives the pie chart.

Starting at **D1**:

| Cell | Content |
|------|---------|
| D1 | `Category` |
| E1 | `Amount` |

Then in D2 onward, list every category you use:

| Cell | Category Name |
|------|--------------|
| D2 | `Groceries` |
| D3 | `Flat expenses` |
| D4 | `Food` |
| D5 | `Beverages` |
| D6 | `Shopping` |
| D7 | `Petrol & other expenses` |
| D8 | `Entertainment` |
| D9 | `Transport` |
| D10 | `CC + previous dues` |
| D11 | `Investment` |
| D12 | `Haircut` |
| D13 | `Refund` |
| D14 | `Random` |

> **⚠️ These names must exactly match** what's written in Column A of your consolidated sheet. Any mismatch (extra space, different casing) and the formula won't pick it up.

Now in **E2**, enter this formula:

```
=SUMIFS('2026 Consolidated'!C:C,'2026 Consolidated'!A:A,D2,'2026 Consolidated'!D:D,">="&$B$3,'2026 Consolidated'!D:D,"<="&$B$4,'2026 Consolidated'!F:F,"debit")
```

**Drag E2 down** through all your category rows (E3, E4, ... E14).

**Exception — Refund row:** For the Refund category, use a different formula that includes both debit and credit (since refunds are credits stored as negative numbers):

```
=SUMIFS('2026 Consolidated'!C:C,'2026 Consolidated'!A:A,D13,'2026 Consolidated'!D:D,">="&$B$3,'2026 Consolidated'!D:D,"<="&$B$4)
```

This removes the `F:F,"debit"` filter so it picks up credit (negative) amounts too.

**Add a Total row** below the last category:

| Cell | Content |
|------|---------|
| D15 | `Total` |
| E15 | `=SUM(E2:E14)` |

### 6.5 Pie Chart

1. Select the range **D1:E14** (category table headers through last category, exclude Total)
2. Go to **Insert → Chart**
3. Chart type: **Pie chart**
4. Under **Customize → Pie chart**: set Slice label to **Percentage**
5. Position the chart wherever you want on the dashboard

The pie chart auto-updates when:
- New transactions land in the consolidated sheet
- You change the month in B1

### 6.6 Month-End Process

When a new month starts (takes ~30 seconds):

1. Note down the closing balances (B10, B15, B20)
2. Change B1 to the new month (e.g., `April 2026`)
3. Paste the old closing balances as the new opening balances (B9, B14, B19)

That's it. The SUMIFS formulas and pie chart automatically adjust to the new month.

---

## Part 7: Testing End-to-End

1. Make sure your Python script is running (`launchctl list | grep expensetracker`)
2. Make sure both n8n workflows are activated
3. Make a small transaction from one of your tracked bank accounts (e.g., UPI ₹1)
4. Wait for the SMS to arrive on your iPhone → sync to Mac → appear in Telegram
5. Tap a category button in Telegram
6. Type a description (or `-` to skip)
7. Check your Google Sheet — a new row should appear in the consolidated sheet
8. Check your Dashboard — the numbers and pie chart should update

If any step fails, check the [Troubleshooting](#troubleshooting) section.

---

## Customization Guide

### Adding a New Bank

1. **Find the sender ID**: Open Messages on your Mac, find an SMS from the bank. The sender will be something like `VK-HDFCBK` or `AD-SBIBNK`. The part after the prefix (e.g., `HDFCBK`) is what you match with `endswith()`.

2. **Add to TRACKED_SENDERS** in the Python script:
   ```python
   TRACKED_SENDERS = {"FEDBNK-S", "KOTAKB-S", "YOURBANKID"}
   ```

3. **Add parsing logic** in `parse_sms()`:
   ```python
   elif normalised == "YOURBANKID":
       # Study a few SMS from this bank to understand the format
       # Look for keywords like "debited", "credited", "sent", "received"
       if "debited" in text_lower or "withdrawn" in text_lower:
           txn_type = "debit"
       elif "credited" in text_lower or "received" in text_lower:
           txn_type = "credit"
       else:
           print(f"[SKIP] YourBank: cannot determine type: {text[:60]}")
           return None

       date_match = re.search(r'(\d{2}[/-]\d{2}[/-]\d{2,4})', text)
       date = normalise_date(date_match.group(1)) if date_match else ""

       return {
           "sender": "YourBank",
           "amount": amount,
           "type": txn_type,
           "date": date,
           "message": text
       }
   ```

4. **Add dashboard formulas**: Copy one of the existing bank sections on the Dashboard sheet, change the bank name filter from `"Jupiter"` to `"YourBank"`.

5. **Restart the Python script**:
   ```bash
   launchctl unload ~/Library/LaunchAgents/com.expensetracker.plist
   launchctl load ~/Library/LaunchAgents/com.expensetracker.plist
   ```

### Adding or Changing Categories

1. Update the **inline keyboard** in n8n Workflow A (Node 2) — add/remove buttons
2. Update the **categoryMap** in n8n Workflow B (Node 5) — add/remove mappings
3. Update the **Dashboard** category table (Column D) — add/remove rows
4. Make sure the category name in the `categoryMap` **exactly matches** what's in Column D of the Dashboard

### Changing Banks for Non-Indian Users

The `extract_amount()` function looks for `Rs.` followed by a number. If your bank uses a different currency prefix:

```python
def extract_amount(text):
    # For USD: look for $ followed by amount
    match = re.search(r'\$\s*([\d,]+(?:\.\d{1,2})?)', text)
    # For EUR: look for € followed by amount
    # match = re.search(r'€\s*([\d,]+(?:\.\d{1,2})?)', text)
    if match:
        return match.group(1).replace(",", "")
    return None
```

---

## Troubleshooting

### Script says "chat.db not found"
- Full Disk Access not granted to your Python binary
- Check: System Settings → Privacy & Security → Full Disk Access
- Make sure the correct Python path is listed (run `which python3` to confirm)

### SMS not syncing to Mac
- Verify iCloud Messages is ON on both iPhone and Mac
- Both devices must use the same Apple ID
- Try toggling Messages in iCloud off and on

### n8n webhook returns error
- Make sure both workflows are **activated** (green toggle)
- Check the webhook URL in the Python script matches exactly
- Check n8n execution logs for errors

### Telegram bot not responding
- Verify the webhook is registered: visit `https://api.telegram.org/bot<TOKEN>/getWebhookInfo`
- The `url` field should show your n8n Telegram trigger URL
- Make sure you're messaging the bot from the account matching your Chat ID

### Google Sheets not getting data
- Check n8n credentials for Google Sheets (may need to re-authenticate)
- Verify the sheet name in Node 6 matches exactly
- Check n8n execution logs — the Google Sheets node will show errors

### Dashboard formulas showing 0 when there's data
- Category name mismatch: the name in Column D of Dashboard must **exactly** match Column A of the consolidated sheet (case-sensitive, no extra spaces)
- Date format issue: dates in Column D of the consolidated sheet must be real dates (`MM/DD/YYYY`), not text strings. Click a date cell — if the formula bar shows `3/15/2026` and not `'3/15/2026` (with a leading quote), it's a real date.

### Mac goes to sleep and script stops
- System Settings → Battery → Enable "Prevent your Mac from automatically sleeping when the display is off"
- Locking your Mac is fine; sleeping is not

### Script processes old messages on first run
- This is normal on the first run — it processes the last 200 messages
- After the first run, the state file tracks what's been processed
- If you want to skip old messages: run the script once, let it process everything, then delete the unwanted rows from the sheet

---

## Architecture Overview

```
┌─────────────┐     iCloud      ┌─────────────┐
│   iPhone     │ ──── sync ────→│    Mac       │
│  (Bank SMS)  │                 │  (chat.db)  │
└─────────────┘                 └──────┬──────┘
                                       │
                                       │ Python script
                                       │ polls every 10s
                                       │
                                       ▼
                              ┌─────────────────┐
                              │   n8n (Railway)  │
                              │                  │
                              │  Workflow A:      │
                              │  Webhook → Telegram│
                              │                  │
                              │  Workflow B:      │
                              │  Telegram → Sheets│
                              └────────┬─────────┘
                                       │
                         ┌─────────────┼─────────────┐
                         ▼                           ▼
                  ┌─────────────┐           ┌──────────────┐
                  │  Telegram   │           │ Google Sheets │
                  │  (buttons)  │           │ (data + dash) │
                  └─────────────┘           └──────────────┘
```

**Why this architecture?**

- **iMessage sync** eliminates the need for any server to receive SMS directly
- **n8n on Railway** runs 24/7 independently of your Mac (Mac only needs to be awake for the Python polling part)
- **Telegram** gives you a mobile-friendly interface to categorize on the go
- **Google Sheets** is free, accessible anywhere, and formulas handle all the math

---

## Key Technical Notes

- SMS text in `chat.db` is stored in the `attributedBody` binary blob, **not** the plain `text` column. The Python script handles this with a custom decoder.
- Sender IDs have carrier prefixes (e.g., `VK-KOTAKB-S`, `AD-KOTAKB-S`). The script uses `endswith()` matching to handle all variants.
- Credits are stored as **negative numbers** in the sheet. This makes balance formulas simple: `Opening - Spent + Received`.
- Dates must be in `MM/DD/YYYY` format for Google Sheets to treat them as real dates (not text). The n8n `toSheetDate` function handles this conversion.
- n8n webhook data arrives wrapped in a `body` object, so field paths in n8n are `$json.body.sender`, not `$json.sender`.
- The Python script's state file (`~/.expense_tracker_state.json`) persists processed message IDs. Delete this file to reprocess old messages (useful for testing, but will create duplicate rows).

---

## License

Do whatever you want with this. No warranty.
