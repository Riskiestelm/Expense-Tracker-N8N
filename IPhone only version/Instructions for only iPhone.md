# Expense Tracker — iPhone-Only Setup Guide

> Bank SMS arrives on your iPhone → iOS Shortcut fires automatically → n8n parses and sends a Telegram message with category buttons → you tap one → Google Sheets logs the row. No Mac required, no Python, no launchd.

**Setup time: ~2–3 hours**

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Google Sheets Setup](#2-google-sheets-setup)
3. [n8n Setup on Railway](#3-n8n-setup-on-railway)
4. [Telegram Bot Setup](#4-telegram-bot-setup)
5. [iOS Shortcuts Setup](#5-ios-shortcuts-setup)
6. [Google Sheets OAuth](#6-google-sheets-oauth)
7. [End-to-End Test](#7-end-to-end-test)

---

## 1. Prerequisites

You need all of the following before starting.

### Accounts to create

| Account | Where | Notes |
|---------|-------|-------|
| Google account | google.com | For Google Sheets + Google Cloud |
| Telegram account | telegram.org | Free, available on iPhone |
| Railway account | railway.app | Free tier works; Hobby plan recommended for always-on |
| Splitwise account | splitwise.com | Optional — only needed for the "Split this" button |

### What you need on your iPhone

- **Bank SMS notifications enabled** for all your banks — check Settings → Notifications for each bank app
- **iOS 16 or later** (Shortcuts automation with "Run after confirmation immediately" requires iOS 16+)
- **Shortcuts app** installed (comes pre-installed; if removed, reinstall from the App Store)
- **Telegram** installed

### What n8n needs from you before you start

You will collect these during setup and fill them in as you go:

- Telegram bot token
- Your Telegram chat ID
- n8n Workflow A webhook URL
- n8n Workflow B Telegram webhook URL
- Google Sheets spreadsheet ID
- Google Cloud OAuth Client ID and Secret

---

## 2. Google Sheets Setup

### 2.1 Create the spreadsheet

1. Go to [sheets.google.com](https://sheets.google.com) on your computer
2. Click **Blank** to create a new spreadsheet
3. Rename it to: `Finance tally` (click the title at the top)
4. Right-click the tab at the bottom (`Sheet1`) → **Rename** → type: `2026 Consolidated`

### 2.2 Set up column headers

Click cell **A1** and enter the headers across row 1:

| Cell | Header |
|------|--------|
| A1 | `Expense and receivable name` |
| B1 | `Description` |
| C1 | `Amount` |
| D1 | `Date` |
| E1 | `Bank` |
| F1 | `Type` |

**What goes in each column:**

- **A** — Category (e.g. `Food`, `Groceries`, `Transport`)
- **B** — Description you type in Telegram (blank if you send `-` to skip)
- **C** — Amount. Debits: positive number. Credits/refunds: negative number
- **D** — Date in `MM/DD/YYYY` format — this is a real date, not text
- **E** — Bank (e.g. `Jupiter`, `Kotak`, `Kotak CC`, `BOB`)
- **F** — Transaction type: `debit` or `credit` (always lowercase)

### 2.3 Create the Dashboard sheet

1. Click the **+** button at the bottom to add a new sheet
2. Rename it: `Dashboard`

### 2.4 Month selector

In the `Dashboard` sheet, enter:

| Cell | What to enter |
|------|---------------|
| A1 | `Month:` |
| B1 | Type the current month, e.g. `April 2026` |
| A3 | `Start Date:` |
| B3 | `=DATE(VALUE(RIGHT(B1,4)),MATCH(LEFT(B1,FIND(" ",B1)-1),{"January","February","March","April","May","June","July","August","September","October","November","December"},0),1)` |
| A4 | `End Date:` |
| B4 | `=EOMONTH(B3,0)` |

> B1 is the only cell you manually change each month. Everything else updates automatically.

### 2.5 Per-bank balance section

Set up a balance block for each bank. The example below uses Jupiter, Kotak, Kotak CC, and BOB. Add or remove banks to match yours.

**Jupiter (rows 7–10):**

| Cell | Content |
|------|---------|
| A7 | `Jupiter Spent:` |
| B7 | `=SUMIFS('2026 Consolidated'!C:C,'2026 Consolidated'!E:E,"Jupiter",'2026 Consolidated'!D:D,">="&$B$3,'2026 Consolidated'!D:D,"<="&$B$4,'2026 Consolidated'!F:F,"debit")` |
| A8 | `Jupiter Received:` |
| B8 | `=ABS(SUMIFS('2026 Consolidated'!C:C,'2026 Consolidated'!E:E,"Jupiter",'2026 Consolidated'!D:D,">="&$B$3,'2026 Consolidated'!D:D,"<="&$B$4,'2026 Consolidated'!F:F,"credit"))` |
| A9 | `Jupiter Op Bal:` |
| B9 | *(Enter your opening balance manually — check your bank app)* |
| A10 | `Jupiter Current Bal:` |
| B10 | `=B9-B7+B8` |

**Kotak (rows 12–15):** Same structure, replace `"Jupiter"` with `"Kotak"` in formulas.

**Kotak CC (rows 17–20):** Same structure, replace `"Jupiter"` with `"Kotak CC"`.

**BOB (rows 22–25):** Same structure, replace `"Jupiter"` with `"BOB"`.

### 2.6 Category breakdown table

Starting at column D on the Dashboard sheet:

| Cell | Content |
|------|---------|
| D1 | `Category` |
| E1 | `Amount` |
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
| D15 | `Total` |

> These names must exactly match what gets written to column A. Case matters. No extra spaces.

In **E2**, enter:
```
=SUMIFS('2026 Consolidated'!C:C,'2026 Consolidated'!A:A,D2,'2026 Consolidated'!D:D,">="&$B$3,'2026 Consolidated'!D:D,"<="&$B$4,'2026 Consolidated'!F:F,"debit")
```

Drag E2 down to **E12** (all rows except Refund).

**For E13 (Refund)** — omit the `"debit"` filter because refunds are credits (negative numbers):
```
=SUMIFS('2026 Consolidated'!C:C,'2026 Consolidated'!A:A,D13,'2026 Consolidated'!D:D,">="&$B$3,'2026 Consolidated'!D:D,"<="&$B$4)
```

**For E14 (Random):**
```
=SUMIFS('2026 Consolidated'!C:C,'2026 Consolidated'!A:A,D14,'2026 Consolidated'!D:D,">="&$B$3,'2026 Consolidated'!D:D,"<="&$B$4,'2026 Consolidated'!F:F,"debit")
```

**E15:** `=SUM(E2:E14)`

### 2.7 Opening balance table

This lets balances cascade month-to-month. Starting at column H:

| Cell | Header |
|------|--------|
| H1 | `Month` |
| I1 | `Jupiter Op Bal` |
| J1 | `Kotak Op Bal` |
| K1 | `Kotak CC Op Bal` |
| L1 | `BOB Op Bal` |

In H2, type your first month (e.g. `March 2026`). Enter the actual opening balances in I2, J2, K2, L2 — look them up in your bank apps.

For each subsequent month: type the month in H3, H4, etc. For the opening balances (I3 onward), just enter the previous month's closing balance manually — it takes 30 seconds per month to update.

### 2.8 Pie chart

1. Select **D1:E14**
2. Click **Insert → Chart**
3. Change chart type to **Pie chart** if not auto-detected
4. Customize → Pie chart → Slice label: **Percentage**
5. Position the chart anywhere on the dashboard

### 2.9 Copy your Spreadsheet ID

From the URL of your spreadsheet, copy the long ID:
```
https://docs.google.com/spreadsheets/d/THIS_IS_THE_ID/edit
```
Save it — you'll need it when connecting n8n to Google Sheets.

---

## 3. n8n Setup on Railway

### 3.1 Deploy n8n

1. Go to [railway.app](https://railway.app) and sign in
2. Click **New Project → Deploy a Template**
3. Search for **n8n** → click **Deploy**
4. Wait ~2 minutes for the build
5. Click your n8n service → **Settings → Networking → Generate Domain**
6. You'll get a URL like `https://n8n-production-XXXX.up.railway.app`
7. Open that URL and create your n8n account (email + password)

### 3.2 Set up Telegram credential in n8n

1. In n8n: **Settings → Credentials → Add Credential**
2. Search for **Telegram API** → select it
3. Enter your bot token (from Part 4 — come back here after creating your bot)
4. Click **Save**. Name it `Telegram Bot`.

### 3.3 Create Workflow A — Expense Tracker: Send

In n8n, click **Workflows → New Workflow**. Name it: `Expense Tracker - Send`.

#### Node 1: Webhook

1. Click **Add first step** → search for **Webhook** → add it
2. Settings:
   - HTTP Method: `POST`
   - Path: leave as auto-generated (creates a UUID path)
   - Respond: `When Last Node Finishes`
3. Click the node to see the full **Webhook URL**:
   ```
   https://[your-n8n-railway-url]/webhook/[uuid]
   ```
4. **Copy and save this URL** — you will paste it into both iOS Shortcuts in Part 5.

#### Node 2: Code — Parse SMS

1. Click **+** to add a node → search for **Code** → add it
2. Language: JavaScript
3. Paste this code exactly:

```javascript
const body = $json.body;

let msg, amount, type, bank, date;

// iOS path — raw message arrives, parse everything here
msg = body.message || '';
const senderRaw = body.sender || '';

// Amount — handles Rs, Rs., and INR formats
const amountMatch = msg.match(/(?:Rs\.?\s*|INR\s*)([\d,]+(?:\.\d{1,2})?)/i);
amount = amountMatch ? amountMatch[1].replace(/,/g, '') : null;

// Bank detection — IMPORTANT: check Kotak CC FIRST before generic Kotak
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

// Type — "received" or "credited" = credit, everything else = debit
type = /received|credited/i.test(msg) ? 'credit' : 'debit';

// Date — handles DD-MM-YYYY, DD/MM/YYYY, DD-MM-YY, and DD-Mon-YYYY (e.g. 12-Apr-2026)
const monthMap = {
  jan:'01', feb:'02', mar:'03', apr:'04', may:'05', jun:'06',
  jul:'07', aug:'08', sep:'09', oct:'10', nov:'11', dec:'12'
};
let dateMatch = msg.match(/(\d{2})[\/\-]([A-Za-z]{3})[\/\-](\d{4})/);
if (dateMatch) {
  date = `${dateMatch[1]}-${monthMap[dateMatch[2].toLowerCase()]}-${dateMatch[3]}`;
} else {
  dateMatch = msg.match(/(\d{2}[\/\-]\d{2}[\/\-]\d{2,4})/);
  date = dateMatch ? dateMatch[1] : '';
}

return [{ json: { amount, type, bank, date, message: msg } }];
```

#### Node 3: Telegram — Send Message

1. Click **+** → search for **Telegram** → select **Send Message**
2. Credential: select your `Telegram Bot`
3. Chat ID: your Telegram chat ID (from Part 4)
4. Parse Mode: `HTML`
5. In the **Text** field, paste:

```
💳 New Transaction Detected

Bank: {{ $json.bank }}
Amount: ₹{{ $json.amount }}
Type: {{ $json.type }}
Date: {{ $json.date || new Date().toLocaleDateString('en-GB').split('/').join('-') }}
SMS: {{ $json.message }}

Select a category:
```

6. Scroll down to **Additional Fields** → enable **Reply Markup**
7. Reply Markup type: **Inline Keyboard**
8. Add buttons. Set the Row number for each button:

**Row 1:**
| Text | Callback Data |
|------|--------------|
| `🛒 Groceries` | `GR` |
| `🍔 Food` | `FD` |
| `🏢 Flat expenses` | `FE` |

**Row 2:**
| Text | Callback Data |
|------|--------------|
| `🛍 Shopping` | `SH` |
| `🥤 Beverages` | `BV` |
| `🎬 Entertainment` | `EN` |

**Row 3:**
| Text | Callback Data |
|------|--------------|
| `🚗 Transport` | `TP` |
| `💳 CC + previous dues` | `CC` |
| `💰 Investment` | `IN` |

**Row 4:**
| Text | Callback Data |
|------|--------------|
| `💈 Haircut` | `HC` |
| `🔄 Refund` | `RF` |
| `⛽ Petrol` | `PE` |
| `📝 Other` | `RD` |

**Row 5 (URL button — different type):**
- Type: `URL Button`
- Text: `💸 Split this`
- URL: `https://YOUR-SPLITWISE-MINIAPP-URL.up.railway.app/?amount={{ $json.amount }}&description={{ $json.message }}&bank={{ $json.bank }}&date={{ $json.date }}`

> Replace `YOUR-SPLITWISE-MINIAPP-URL` with your Railway URL if you deploy the optional Splitwise mini app. If you skip Splitwise, omit Row 5.

9. Click **Save** → click **Activate** (toggle turns green)

---

### 3.4 Create Workflow B — Expense Tracker: Receive

In n8n → **Workflows → New Workflow**. Name it: `Expense Tracker - Receive`.

#### Node 1: Telegram Trigger

1. Add a **Telegram Trigger** node
2. Credential: your `Telegram Bot`
3. Trigger On: check both **Callback Query** and **Message**
4. Click **Save**
5. n8n shows a **Webhook URL** for Telegram — it looks like:
   ```
   https://[your-n8n-railway-url]/webhook/[uuid]/webhook
   ```
6. Copy this URL — you'll register it with Telegram in Part 4.

#### Node 2: Switch

1. Add a **Switch** node after the Telegram Trigger
2. Add two rules:
   - **Rule 1** (Output 0 — button tap):
     - Value 1: `{{ $json.callback_query ? 'callback' : 'message' }}`
     - Operation: `equals`
     - Value 2: `callback`
   - **Rule 2** (Output 1 — text message):
     - Same expression
     - Operation: `equals`
     - Value 2: `message`

#### Node 3: Code — Save Transaction (connects to Switch Output 0)

This runs when you tap a category button. It stores the transaction in workflow memory until you send a description.

1. Add a **Code** node → connect to **Switch Output 0**
2. Language: JavaScript
3. Paste this code:

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

> Note: `bank` uses `.+` (not `\w+`) to capture multi-word names like `Kotak CC`.

#### Node 4: Telegram — Send Message (connects to Node 3)

1. Add a **Telegram** → **Send Message** node after Node 3
2. Credential: your `Telegram Bot`
3. Chat ID: `{{ $json.callback_query.message.chat.id }}`
4. Text: `Got it! Now send a description for this expense, or send - to skip.`

#### Node 5: Code — Retrieve Transaction (connects to Switch Output 1)

This runs when you reply with a description (or `-`). It builds the final row.

1. Add a **Code** node → connect to **Switch Output 1**
2. Paste this code:

```javascript
const staticData = $getWorkflowStaticData('global');
const pending = staticData.pendingTransaction;
const description = $input.all()[0].json.message.text;

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
  // DD-MM-YYYY → MM/DD/YYYY
  let m = raw.match(/^(\d{2})-(\d{2})-(\d{4})$/);
  if (m) return `${m[2]}/${m[1]}/${m[3]}`;
  // DD-MM-YY → MM/DD/20YY
  m = raw.match(/^(\d{2})-(\d{2})-(\d{2})$/);
  if (m) return `${m[2]}/${m[1]}/20${m[3]}`;
  // DDMMMYY (e.g. 12APR26) → MM/DD/20YY
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
// Credits stored as negative numbers
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

#### Node 6: Google Sheets — Append Row (connects to Node 5)

1. Add a **Google Sheets** node after Node 5
2. Operation: **Append Row**
3. Credential: your `Google Sheets` credential (set up in Part 6)
4. Document: select `Finance tally`
5. Sheet: `2026 Consolidated`
6. Column mapping:

| Sheet Column | Value |
|-------------|-------|
| A — Expense and receivable name | `{{ $json.category }}` |
| B — Description | `{{ $json.description }}` |
| C — Amount | `{{ $json.amount }}` |
| D — Date | `{{ $json.date }}` |
| E — Bank | `{{ $json.bank }}` |
| F — Type | `{{ $json.type }}` |

7. Click **Save** → **Activate** (toggle turns green)

---

## 4. Telegram Bot Setup

### 4.1 Create the bot

1. Open Telegram and search for **@BotFather**
2. Tap **Start**, then send: `/newbot`
3. Give it a name (e.g. `Expense Tally Bot`)
4. Give it a username ending in `bot` (e.g. `myexpensetally_bot`)
5. BotFather gives you a **bot token** like: `7123456789:AAH1234abcdefghijklm`
6. Save this token — paste it into your Telegram credential in n8n (Step 3.2)

### 4.2 Get your Chat ID

1. Open your new bot and send it any message (e.g. `hello`)
2. In your browser, visit (replace with your actual token):
   ```
   https://api.telegram.org/bot<YOUR_BOT_TOKEN>/getUpdates
   ```
3. In the JSON response, find `"chat":{"id":` — that number is your **Chat ID**
4. Save it. It looks like `1269390790`.

### 4.3 Register the Webhook B URL with Telegram

After activating Workflow B, register its URL with Telegram. Visit this URL in your browser (replace placeholders):
```
https://api.telegram.org/bot<YOUR_BOT_TOKEN>/setWebhook?url=<N8N_WORKFLOW_B_TELEGRAM_URL>
```

You should see: `{"ok":true,"result":true}`

Verify it worked:
```
https://api.telegram.org/bot<YOUR_BOT_TOKEN>/getWebhookInfo
```

The `url` field should show your n8n URL.

---

## 5. iOS Shortcuts Setup

You need **two** shortcuts — one for banks that use `Rs` in their SMS, and one for Kotak Credit Card which uses `INR`.

### 5.1 Shortcut 1 — "Rs" banks (Jupiter, Kotak 811, Bank of Baroda)

1. Open the **Shortcuts** app on your iPhone
2. Go to the **Automation** tab at the bottom
3. Tap **+** → **New Automation**
4. Scroll down and tap **Message**
5. Set up the trigger:
   - **When:** Message is received
   - **Message contains:** `Rs`
   - Leave sender blank (matches all senders)
6. Tap **Next**
7. Tap **Add Action**

**Action 1: Receive Input**
- Search for and add: **Receive [Messages] as Input**
- This makes the incoming message available as `Shortcut Input`

**Action 2: Get Contents of URL**
- Search for and add: **Get Contents of URL**
- URL: paste your Workflow A webhook URL (from Step 3.3, Node 1)
  ```
  https://[your-n8n-railway-url]/webhook/[uuid]
  ```
- Method: `POST`
- Headers: add `Content-Type` = `application/json`
- Body type: `JSON`
- Add two JSON fields:
  - Key: `message` → Value: tap the variable picker → select **Shortcut Input** → **Content**
  - Key: `sender` → Value: tap the variable picker → select **Shortcut Input** → **Sender**

8. Tap **Next** → toggle **OFF** "Ask Before Running" → tap **Done**

### 5.2 Shortcut 2 — "INR" banks (Kotak Credit Card)

Repeat the exact same steps as Shortcut 1, but in step 5, change:
- **Message contains:** `INR`

Everything else is identical — same Workflow A webhook URL, same JSON body fields.

### 5.3 Remove the confirmation tap (important)

By default, iOS asks you to confirm before running an automation. To make this fully automatic:

1. In Shortcuts → Automation tab, tap your new automation
2. Tap **Edit**
3. Look for **"Run after confirmation immediately"** — toggle it **ON**
4. Tap **Done**

Repeat for both shortcuts.

> This setting is only available on iOS 16+. Once enabled, the shortcut runs silently in the background as soon as a matching SMS arrives — no tap needed.

---

## 6. Google Sheets OAuth

n8n needs permission to write to your Google Sheet. This requires a Google Cloud OAuth app.

### 6.1 Create a Google Cloud project

1. Go to [console.cloud.google.com](https://console.cloud.google.com)
2. Click the project dropdown → **New Project** → name it `expense-tracker` → **Create**
3. Make sure the new project is selected

### 6.2 Enable the APIs

1. **APIs & Services → Library**
2. Search **Google Sheets API** → **Enable**
3. Search **Google Drive API** → **Enable**

### 6.3 Configure OAuth consent screen

1. **APIs & Services → OAuth consent screen**
2. User Type: **External** → **Create**
3. Fill in:
   - App name: `Expense Tracker`
   - User support email: your Gmail
   - Developer contact email: your Gmail
4. Click **Save and Continue** through the Scopes screen (no changes)
5. Test users: **Add Users** → add your Gmail → **Save and Continue**

### 6.4 Publish the app (critical — prevents token expiry)

1. Still on the OAuth consent screen, click the **Audience** tab
2. Click **Publish App** → confirm

> If you skip this step, the OAuth token expires every 7 days and n8n stops writing to Sheets.

### 6.5 Create OAuth credentials

1. **APIs & Services → Credentials → Create Credentials → OAuth Client ID**
2. Application type: **Web application**
3. Name: `n8n`
4. Under **Authorized redirect URIs**: you need the URI from n8n first — continue to 6.6 then come back

### 6.6 Connect Google Sheets in n8n

1. In n8n: **Settings → Credentials → Add Credential**
2. Search **Google Sheets OAuth2 API** → select it
3. Enter the **Client ID** and **Client Secret** from Step 6.5
4. n8n shows a **Redirect URI** — copy it
5. Go back to Google Cloud → your OAuth Client → add that URI under **Authorized redirect URIs** → **Save**
6. Back in n8n: click **Sign in with Google** → complete the OAuth flow
7. Name the credential `Google Sheets` → **Save**

---

## 7. End-to-End Test

Once everything is set up and both n8n workflows are activated (green toggle):

1. Have a bank send you an SMS — or find an existing SMS that contains `Rs` (e.g. from Jupiter or Kotak 811)
2. Forward that SMS to yourself, or wait for a real transaction
3. The iOS Shortcut should fire automatically within a few seconds of the SMS arriving
4. Open Telegram — you should see a message from your bot with the transaction details and category buttons
5. Tap a category button
6. Your bot replies: `Got it! Now send a description, or send - to skip.`
7. Reply with a description or `-`
8. Open your Google Sheet — a new row should appear in `2026 Consolidated`

**If nothing happens:**

- Check that both Shortcuts automations have "Run after confirmation immediately" enabled
- Open the Shortcuts app → tap your automation → run it manually to check for errors
- Check n8n's execution history: open Workflow A → click **Executions** tab — errors appear here
- Make sure Workflow A and Workflow B are both **Activated** (green toggle)
- Confirm the Webhook URL in the Shortcut matches exactly what n8n shows

**If Telegram sends a message but Google Sheets doesn't update:**

- Open Workflow B → Executions → check for errors on Node 6
- Make sure the Google Sheets credential is still connected (re-authenticate if needed)
- Confirm the spreadsheet name (`Finance tally`) and sheet name (`2026 Consolidated`) match exactly

---

## Optional: Splitwise Mini App

If you want the `💸 Split this` button to work, you need to deploy the Flask mini app on Railway. See the `splitwise-miniapp/` folder in this repo for the source code. Deploy it to Railway, set the `SPLITWISE_API_KEY` environment variable, and update the URL button in Workflow A Node 3 with your Railway URL.

---

## Category Reference

| Code | Full Name | Telegram Button |
|------|-----------|-----------------|
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
