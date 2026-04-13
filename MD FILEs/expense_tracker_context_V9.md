# Expense Tracker — Full Build Context V9

## What We Built
Automated personal expense tracking for Jupiter (Federal Bank), Kotak 811, Kotak Credit Card, and Bank of Baroda transactions, with a Google Sheets dashboard and Splitwise expense splitting via a Telegram Mini App.

Two SMS entry points are supported:
- **Mac (preferred)** — Python script reads iMessage `chat.db` autonomously, zero user interaction
- **iPhone only** — iOS Shortcuts POSTs raw SMS to n8n; fully automated on iOS 18 via "Run after confirmation immediately" toggle

Everything after n8n (parse → Telegram → Google Sheets → Splitwise) is identical for both paths.

---

## Flows

### Mac Flow (fully autonomous)
```
Bank SMS arrives on iPhone
→ iMessage syncs to Mac (same Apple ID)
→ Python script polls ~/Library/Messages/chat.db every 10 seconds
→ Parses SMS: extracts amount, credit/debit type, date, bank
→ POSTs structured data to n8n webhook (hosted on Railway)
→ n8n sends Telegram message with inline category buttons
→ User taps a button → types description → row appended to Google Sheet
```

### iPhone-only Flow (fully automated via iOS Shortcuts)
```
Bank SMS arrives on iPhone
→ iOS Shortcut triggers on message containing "Rs" or "INR"
→ Runs immediately — no confirmation tap needed (enable "Run after confirmation immediately")
→ Shortcut POSTs raw message + sender to n8n webhook
→ n8n Code node parses: amount, type, bank, date
→ n8n sends Telegram message with inline category buttons
→ User taps a button → types description → row appended to Google Sheet
```

---

## Current Status
- ✅ Jupiter (Federal Bank) — operational via iOS Shortcuts
- ✅ Kotak 811 — operational via iOS Shortcuts
- ✅ Kotak Credit Card (INR format) — operational via separate Shortcut
- ✅ Bank of Baroda — operational via iOS Shortcuts
- ✅ n8n hosted on Railway (always-on)
- ✅ Telegram inline button flow working (13 categories)
- ✅ Google Sheet logging (category, description, amount, date, bank, type)
- ✅ Credits stored as negative amounts
- ✅ Dates written as MM/DD/YYYY
- ✅ Dashboard with month dropdown
- ✅ Opening balance table with auto-cascading formulas (Jupiter, Kotak, Kotak CC, BOB)
- ✅ Category breakdown with SUMIFS
- ✅ Pie chart
- ✅ Splitwise Mini App live on Railway

---

## Infrastructure

### iOS Shortcuts Setup

Two shortcuts needed:

**Shortcut 1 — trigger: message contains `Rs`**
Covers: Jupiter, Kotak 811, Bank of Baroda

**Shortcut 2 — trigger: message contains `INR`**
Covers: Kotak Credit Card and any bank using INR format

Both shortcuts have identical actions:
```
Receive messages as input
↓
Get Contents of URL
  URL: https://n8n-production-xxxx.up.railway.app/webhook/YOUR_WEBHOOK_A_ID
  Method: POST
  Body: JSON
    message → Shortcut Input (Content)
    sender  → Shortcut Input (Sender)
```

**To remove the confirmation tap:** In the automation settings, enable "Run after confirmation immediately". This makes it fully automated — no tap needed per SMS.

#### iOS Quirks
- Jupiter SMS uses `Rs` without a dot — trigger on `Rs` not `Rs.`
- Kotak Credit Card uses `INR` — separate shortcut required
- Sender field returns phone number, not bank ID — bank detection uses message content instead
- Gmail action in Shortcuts is broken — avoid
- `Get Contents of URL` is the correct action for HTTP POST

---

### n8n (Railway)
- Hosted at: `https://n8n-production-xxxx.up.railway.app`

#### Workflow A — Expense Tracker - Send
- Webhook URL: `https://n8n-production-xxxx.up.railway.app/webhook/YOUR_WEBHOOK_A_ID`
- Nodes: Webhook → Code (Parse SMS) → Telegram Send Message

**Parse SMS Code Node:**
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

  // Bank — Kotak CC must be checked before generic Kotak
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

  // Date — handles DD-MM-YYYY, DD/MM/YYYY, DD-MM-YY, and DD-Mon-YYYY (e.g. 12-Apr-2026)
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

**Telegram Send Message node:**
```
💳 New Transaction Detected

Bank: {{ $json.bank }}
Amount: ₹{{ $json.amount }}
Type: {{ $json.type }}
Date: {{ $json.date || new Date().toLocaleDateString('en-GB').split('/').join('-') }}
SMS: {{ $json.message }}

Select a category:
```
- Inline Keyboard:
  - Row 1: 🛒 Groceries (GR), 🍔 Food (FD), 🏢 Flat expenses (FE)
  - Row 2: 🛍 Shopping (SH), 🥤 Beverages (BV), 🎬 Entertainment (EN)
  - Row 3: 🚗 Transport (TP), 💳 CC + previous dues (CC), 💰 Investment (IN)
  - Row 4: 💈 Haircut (HC), 🔄 Refund (RF), ⛽ Petrol (PE), 📝 Other (RD)
  - Row 5: 💸 Split this (URL button → Mini App)

#### Workflow B — Expense Tracker - Receive
- Telegram webhook: `https://n8n-production-xxxx.up.railway.app/webhook/YOUR_WORKFLOW_B_ID/webhook`
- Nodes: Telegram Trigger → Switch → [Node 3] → [Node 4] / [Node 5] → [Node 6]

**Node 3 — Save transaction (Switch output 0):**
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

**Note:** `bank` uses `.+` (not `\w+`) to capture multi-word names like `Kotak CC`.

**Node 4 — Ask for description (connected to Node 3):**
- Chat ID: `{{ $json.callback_query.message.chat.id }}`
- Text: `Got it! Now send a description, or send - to skip.`

**Node 5 — Retrieve transaction (Switch output 1):**
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

**Node 6 — Google Sheets Append Row:**
- Document: Finance tally → Sheet: `2026 Consolidated`
- A=category, B=description, C=amount, D=date, E=bank, F=type

---

### Mac Setup (if using Mac path)
- Python script: `~/expense-tracker-agent/scripts/jupiter_tracker.py`
- Python path: `/opt/anaconda3/bin/python3.12`
- State file: `~/.jupiter_tracker_state.json`
- launchd plist: `~/Library/LaunchAgents/com.yourname.jupytertracker.plist`
- Logs: `~/Library/Logs/jupytertracker.log`
- Full Disk Access must be granted to `/opt/anaconda3/bin/python3.12`
- Mac must never sleep

---

### Telegram
- Bot: @YourExpenseBot
- Chat ID: `YOUR_TELEGRAM_CHAT_ID`

### Google Sheets
- Document: Finance tally
- Data sheet: `2026 Consolidated`
- Dashboard sheet: `Dashboard`
- Google Cloud Project ID: `YOUR_GCP_PROJECT_ID`
- OAuth Client ID: `YOUR_OAUTH_CLIENT_ID.apps.googleusercontent.com`

### Splitwise Mini App
- GitHub repo: `https://github.com/YOUR_USERNAME/splitwise-miniapp` (private)
- Railway URL: `https://your-splitwise-app.up.railway.app`
- Stack: Python Flask + gunicorn + plain HTML/JS
- Railway env variable: `SPLITWISE_API_KEY`
- Splitwise user ID: `YOUR_SPLITWISE_USER_ID`

---

## Google Sheet Structure

### Data Sheet: `2026 Consolidated`
| Column | Content | Notes |
|---|---|---|
| A | Category | Full name e.g. Food, Groceries |
| B | Description | Free text or blank |
| C | Amount | Positive = debit, Negative = credit |
| D | Date | MM/DD/YYYY format (real date) |
| E | Bank | Jupiter / Kotak / Kotak CC / BOB |
| F | Type | debit / credit (lowercase) |

### Dashboard Sheet
- Month dropdown in B1 (Data validation from H2:H13)
- Start/End date formulas in B3/B4
- Per-bank rows: Spent, Received, Opening Balance, Current Balance — for Jupiter, Kotak, Kotak CC, BOB
- Opening balance table in columns H–M (first month entered manually, rest cascade)
- Category breakdown in columns D–E with SUMIFS
- Pie chart on D1:E14

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
| Bank | Sender IDs | Amount Format | Status |
|---|---|---|---|
| Jupiter (Federal Bank) | FEDBNK-S, FEDBNK-T | Rs (no dot) | ✅ Live |
| Kotak 811 | KOTAKB-S + prefixed variants | Rs. | ✅ Live |
| Kotak Credit Card | varies | INR | ✅ Live |
| Bank of Baroda | BOBSMS-S, BOBTXN-S + prefixed variants | Rs. | ✅ Live |

### SMS Formats
**Kotak 811:**
- `Received Rs.X in your Kotak Bank AC X5858 from ... on DD-MM-YY` → credit
- `Rs.X spent via Kotak Debit Card XX9051 at ... on DD/MM/YYYY` → debit
- `Sent Rs.X from Kotak Bank AC X5858 to ... on DD-MM-YY` → debit

**Kotak Credit Card:**
- `INR X spent on Kotak Credit Card xXXXX on DD-Mon-YYYY at ...` → debit

**Jupiter:**
- FEDBNK-S: always debit
- FEDBNK-T: contains "debited" or "credited"

**BOB:**
- `Rs.X credited to A/C xx3092 on DD-MM-YY` → credit
- `Rs.X Dr. from A/C XXXXXX3092` → debit

---

## Known Issues / Quirks
- Jupiter SMS uses `Rs` without dot — Shortcut trigger must be `Rs` not `Rs.`
- Kotak Credit Card uses `INR` and `DD-Mon-YYYY` date format — needs separate Shortcut
- Kotak CC bank detection must come before generic Kotak check in the Code node
- Workflow B Node 3 bank regex must use `.+` not `\w+` to capture multi-word bank names like `Kotak CC`
- Shortcut sender field returns phone number — bank detection uses message content
- Credits stored as negative numbers in sheet
- Dates written as MM/DD/YYYY so Google Sheets treats them as real dates
- n8n webhook data lands in `body` wrapper: `$json.body.message` etc.
- Category names in Dashboard must exactly match column A of consolidated sheet
- Month values in opening balance table must have a space: `March 2026` not `March2026`
- Refund SUMIFS formula must NOT have the "debit" filter
- Mini App URL date param with `/` must be URL-encoded as `%2F`
- macOS Full Disk Access must be granted to exact Python binary path
- n8n Google Sheets OAuth app must be published (Audience tab in Google Cloud) to prevent token expiry

---

## How to Manage

### iOS Shortcuts
- Fully automated — no action needed
- To update webhook URL: edit "Get Contents of URL" in each shortcut

### Python script (Mac path)
```bash
launchctl list | grep jupytertracker
launchctl unload ~/Library/LaunchAgents/com.yourname.jupytertracker.plist
launchctl load ~/Library/LaunchAgents/com.yourname.jupytertracker.plist
tail -f ~/Library/Logs/jupytertracker.log
```

### n8n
- Always on at Railway
- Access at: `https://n8n-production-xxxx.up.railway.app`

### Splitwise Mini App
- Always on at Railway
- To update: push to `YOUR_USERNAME/splitwise-miniapp` → Railway auto-redeploys
