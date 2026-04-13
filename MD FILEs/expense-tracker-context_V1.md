# Expense Tracker Automation — Full Context

## What We Are Building
Automating personal expense tracking. Currently Hemant manually logs every bank transaction into a Google Sheet every 3 days by opening each bank app, copying transactions, categorizing them, and entering them. This automates that entire process.

---

## The Full Flow (Completed Architecture)
```
Bank SMS arrives on iPhone
→ iMessage syncs to Mac (same Apple ID)
→ Python script on Mac polls ~/Library/Messages/chat.db
→ Parses SMS: extracts bank name, amount, credit/debit type
→ Sends parsed data to n8n webhook via POST request
→ n8n sends Telegram message asking for category + description
→ Hemant replies on Telegram in format: CATEGORY_CODE | description
→ n8n appends a row to Google Sheet
```

---

## What Is Already Done

### 1. Telegram Bot
- Bot created via @BotFather
- Bot token: **Hemant has this saved privately**
- Chat ID: **Hemant has this saved**
- Bot is working and tested

### 2. n8n
- Installed locally on Mac via `npx n8n`
- Runs at `http://localhost:5678`
- Account created and set up
- Workflow created with 3 nodes (see below)
- Workflow is **Published/Active**

### 3. n8n Workflow Nodes

#### Node 1: Webhook (Trigger)
- Method: POST
- This receives data from the Python script on Mac
- Production URL: Hemant has this — needs to be retrieved from n8n Webhook node

#### Node 2: Telegram — "Send message and wait for response"
- Connected to Hemant's bot via token
- Chat ID: Hemant's personal chat ID
- Response Type: Free Text
- Message template:
```
💳 New Transaction Detected

Bank: {{ $json.sender }}
Amount: ₹{{ $json.amount }}
Type: {{ $json.type }}
SMS: {{ $json.message }}

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

#### Node 3: Google Sheets — "Append Row"
- Connected to Hemant's Google account via OAuth2
- Google Cloud project ID: 1075000089385
- Both Google Sheets API and Google Drive API are enabled
- Sheet columns and their mappings:

| Sheet Column | n8n Value |
|---|---|
| Expense and receivable name | `{{ $json.data.split('|')[0].trim() }}` |
| Description of the expense | `{{ $json.data.split('|')[1].trim() }}` |
| Amount | `{{ $('Webhook').item.json.type === 'credit' ? '-' + $('Webhook').item.json.amount : $('Webhook').item.json.amount }}` |
| Date | `{{ new Date().toLocaleDateString('en-IN') }}` |

---

## Google Sheet Structure
Columns:
1. **Expense and receivable name** — category code (GR, FD, etc.)
2. **Description of the expense** — what Hemant types after the `|`
3. **Amount** — transaction amount
4. **Date** — auto-filled with today's date

### Critical: Credit vs Debit logic
- Debit (money spent) → stored as positive number e.g. `500`
- Credit (money received) → stored as negative number e.g. `-150`
- This is because the sheet formula deducts negatives from total expenses automatically
- The Python script must detect from SMS text whether it's credit or debit and send `type: "credit"` or `type: "debit"` in the webhook payload

---

## Category Codes
| Code | Category |
|---|---|
| GR | Groceries |
| RN | Rent |
| CK | Cook |
| MD | Maid |
| PF | Purifier |
| WF | Wifi |
| RC | Recharge |
| TR | Trip |
| RF | Refund |
| TP | Transportation |
| PT | Petrol |
| FD | Food |
| BV | Beverages |
| CC | Credit Card |
| IN | Investment |
| RD | Random |
| SH | Shopping |
| KT | Kotak |
| GE | Gaadi expense |
| EN | Entertainment |
| PD | Previous Dues |
| HC | Haircut |
| EL | Electricity |

---

## What Is NOT Done Yet (Next Steps)

### Step 1: Python Script on Mac
Needs to be written and set up. This script:
- Reads `~/Library/Messages/chat.db` (SQLite database where iMessage stores all messages)
- Polls every 10 seconds for new messages
- Filters only bank SMS messages
- Parses each SMS to extract:
  - `sender` — which bank
  - `amount` — transaction amount (numbers only, no ₹ symbol)
  - `type` — `"credit"` or `"debit"` based on keywords in SMS
  - `message` — full raw SMS text
- Sends this as a POST request to the n8n production webhook URL
- Must track already-processed messages to avoid duplicates

### Step 2: Mac Permissions
Terminal needs **Full Disk Access** to read `chat.db`:
- System Settings → Privacy & Security → Full Disk Access → Add Terminal

### Step 3: iMessage Setup on Mac
- iMessage must be enabled on Mac with same Apple ID as iPhone
- Settings → Messages → Enable Messages in iCloud

### Step 4: Auto-start Script on Mac Login
- Once script works, set it up as a launchd service or login item so it runs automatically when Mac starts

### Step 5: Test End-to-End
- Make a real transaction
- Confirm SMS arrives on iPhone → syncs to Mac → Python detects it → Telegram message arrives → reply with category → check Google Sheet

---

## Tech Stack
- **Mac** — runs Python script and n8n locally
- **n8n** — running at localhost:5678, started with `npx n8n`
- **Python** — for reading chat.db and hitting webhook
- **Telegram Bot** — for interactive category+description input
- **Google Sheets** — final destination for all expense data
- **iMessage sync** — bridges iPhone SMS to Mac

---

## Important Notes
- n8n must be running on Mac for the whole system to work (Terminal window with `npx n8n` must stay open, or set up as a background service later)
- The Python script must be running simultaneously
- If Mac is off or asleep, SMS will not be processed until Mac is back on
- Hemant is on Airtel India, iPhone, Indian banks send SMS for every transaction
- Keep bot token private — never share it publicly
