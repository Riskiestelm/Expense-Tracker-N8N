# Expense Tracker — Full Build Context V7

## What We Built
Automated personal expense tracking for Jupiter (Federal Bank), Kotak 811, and Bank of Baroda transactions, with a Google Sheets dashboard and Splitwise expense splitting via a Telegram Mini App.

The original pipeline depended on a Mac running a Python script to read iMessage's `chat.db`. That Mac is no longer available. In this session we migrated the SMS entry point to **iOS Shortcuts** on iPhone, which directly POSTs to the n8n webhook. Everything after n8n (Telegram → Google Sheets → Splitwise) is completely unchanged.

---

## New Flow (as of V7)
```
Bank SMS arrives on iPhone
→ iOS Shortcut triggers on any message containing "Rs."
→ User taps the confirmation prompt (iOS 18 requirement — cannot be bypassed)
→ Shortcut parses the SMS: extracts amount, detects credit/debit, detects bank
→ POSTs directly to n8n webhook (hosted on Railway)
→ n8n sends Telegram message with inline category buttons
→ User taps a button
→ n8n asks for description via Telegram
→ User types description (or sends - to skip)
→ n8n appends row to Google Sheet
```

The Mac-based Python script is no longer part of the flow.

---

## iOS Shortcut — Current State

### What works
- Trigger: any message containing `Rs.`
- Amount extraction via regex Match Text → Get Group At Index
- Amount stored as variable and passed to webhook
- POST to n8n webhook confirmed working — Telegram notification fires correctly

### What is still being built (next session)
- Credit/debit detection via If/Otherwise block — **in progress, not yet tested**
- Date extraction — **not yet built**
- Bank detection (currently hardcoded as `Kotak`) — **not yet built**

### Shortcut Action Order
```
Receive messages as input
↓
Match  Rs\.?\s*([\d,]+(?:\.\d{1,2})?)  in  Content
↓
Get Group At Index 1 in Matches
↓
Set Variable: Amount = (Text output of Get Group At Index)
↓
If Content contains "received"
    Set Variable: Type = credit
Otherwise
    Set Variable: Type = debit
End If
↓
Get Contents of URL (POST to n8n webhook)
  - URL: https://n8n-production-0e8d.up.railway.app/webhook/4027de78-94e3-4172-8722-e151fc229832
  - Method: POST
  - Body: JSON
    - sender: Kotak (hardcoded — needs to be dynamic)
    - amount: Amount variable
    - type: Type variable (once If block is complete)
    - message: Shortcut Input → Content
    - date: (blank — needs to be built)
```

### What needs to be done next session

**1. Fix the Type variable in the webhook action**
- The `type` field in the webhook JSON currently has `debit` hardcoded
- It needs to be changed to the `Type` variable set by the If/Otherwise block
- The If/Otherwise block itself needs to be tested end to end

**2. Add bank detection**
- Currently `sender` is hardcoded as `Kotak`
- Need an If/Otherwise chain to detect which bank sent the SMS:
  - If Content contains `Kotak Bank` → Bank = `Kotak`
  - If Content contains `Federal Bank` → Bank = `Jupiter`
  - If Content contains a BOB-specific phrase → Bank = `BOB`
- Then pass the `Bank` variable into the webhook `sender` field

**3. Add date extraction**
- Kotak SMS date format: `DD-MM-YY` or `DD/MM/YYYY`
- Need Match Text with pattern `(\d{2}[\/\-]\d{2}[\/\-]\d{2,4})` → Get Group At Index 1
- Store as `Date` variable and pass into webhook `date` field
- n8n's Node 5 already handles date normalisation to MM/DD/YYYY — so just pass whatever raw date the SMS contains

**4. Test with a real bank SMS**
- Once all three above are done, test with an actual Kotak transaction
- Verify Telegram shows correct amount, type, bank, date
- Verify Google Sheet row is written correctly

**5. Replicate for Jupiter and BOB**
- The single Shortcut handles all banks via the bank detection If chain
- No separate shortcuts needed — one automation covers everything

---

## Why iOS Shortcuts and not something else

### What was explored
- **Mac Python script** — original solution, no longer viable (Mac unavailable)
- **Android + MacroDroid** — ruled out (no Android device available)
- **iOS Shortcuts via email** — tried Send Email (Mail app not configured) and Gmail action (broken integration, "Could Not Run Send Message" error persists regardless of Gmail accounts being logged in)
- **iOS Shortcuts direct webhook POST** — works ✅ chosen path

### iOS 18 limitation
iOS 18 requires a manual confirmation tap for automations that make network requests. There is no workaround. Every bank SMS will require one tap to approve before the Shortcut runs. This is a known Apple restriction and cannot be bypassed.

### Key Shortcuts quirks learned
- The sender field in the Message trigger strips dashes — `KOTAKB-S` becomes unparseable. Workaround: leave sender blank and filter by message content instead.
- `Get Details of Messages` action does not exist on all iOS versions — use `Shortcut Input → Content` instead to get SMS body.
- Gmail Send Message action throws "Could Not Run Send Message" even with accounts logged in — broken integration, avoid.
- `Get Contents of URL` is the correct action for HTTP POST requests.
- Variables are set via `Set Variable` action and referenced by name in subsequent actions.
- The `Match Text` + `Get Group At Index 1` combo is how you do regex capture groups in Shortcuts.

---

## Infrastructure (unchanged from V6)

### n8n (Railway)
- Hosted at: `https://n8n-production-0e8d.up.railway.app`

#### Workflow A — Expense Tracker - Send
- Webhook URL: `https://n8n-production-0e8d.up.railway.app/webhook/4027de78-94e3-4172-8722-e151fc229832`
- Nodes: Webhook → Telegram Send Message (with inline keyboard)
- Inline keyboard: 13 category buttons + "💸 Split this" URL button

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

### Splitwise Mini App
- GitHub repo: `https://github.com/Riskiestelm/splitwise-miniapp` (private)
- Railway URL: `https://splitwise-miniapp-production.up.railway.app`
- Stack: Python Flask + gunicorn + plain HTML/JS frontend
- Railway env variable: `SPLITWISE_API_KEY`
- Auto-deploys from GitHub on every push to main

---

## n8n Workflow B — Node 5 Code (for reference)

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
| Jupiter (Federal Bank) | FEDBNK-S, FEDBNK-T | ✅ Previously live via Python, now via Shortcut (in progress) |
| Kotak 811 | KOTAKB-S, VK-KOTAKB-S, AD-KOTAKB-S | ✅ Shortcut working (amount extraction confirmed) |
| Bank of Baroda | BOBSMS-S, BOBTXN-S + prefixed variants | ✅ Previously live via Python, now via Shortcut (in progress) |

### Kotak SMS Formats
1. `Received Rs.X in your Kotak Bank AC X5858 from ... on DD-MM-YY` → credit
2. `Rs.X spent via Kotak Debit Card XX9051 at ... on DD/MM/YYYY` → debit
3. `Sent Rs.X from Kotak Bank AC X5858 to ... on DD-MM-YY` → debit

### Jupiter SMS Formats
- FEDBNK-S: always debit
- FEDBNK-T: contains "debited" or "credited"

### BOB SMS Formats
1. `Rs.X credited to A/C xx3092 on DD-MM-YY` → credit
2. `Rs.X Dr. from A/C XXXXXX3092` → debit

---

## Known Issues / Quirks
- iOS 18 requires manual confirmation tap for every automation — cannot be bypassed
- Shortcut sender field strips dashes — use message content filtering instead of sender filtering
- Gmail action in Shortcuts is broken — throws "Could Not Run Send Message" regardless of accounts
- Credits stored as negative numbers in the sheet
- Dates written as MM/DD/YYYY so Google Sheets treats them as real dates (not text)
- n8n webhook data lands in a `body` wrapper: field paths are `$json.body.sender` etc.
- Category names in Dashboard column D must exactly match column A in consolidated sheet
- Month values in opening balance table must have a space: `March 2026` not `March2026`
- Refund formula in Dashboard must NOT have the "debit" filter
- Mini App URL date param with `/` must be URL-encoded as `%2F`

---

## Splitwise Mini App — Architecture (unchanged)

### Flask Endpoints
- `GET /` — serves static/index.html
- `GET /groups` — fetches Splitwise groups
- `POST /create-expense` — creates Splitwise expense

### Active Splitwise Groups
| Group | ID |
|---|---|
| Reliable Pride | 66008700 |
| BANG-A-LORE | 70649501 |
| H2k2 | 92517925 |
| H@-BSDK | 81408874 |
| YouTube Premium | 74648390 |

### Key Member IDs
| Name | Splitwise ID |
|---|---|
| Hemant | 46271821 |
| Ayush | 45606672 |
| Shubh | 45186417 |
| Kunal | 58911127 |
| Kriti | 67346118 |
| Tanishka | 49666031 |
| Harsh Popat | 62176651 |
| Harsh Kashyap | 62761467 |
