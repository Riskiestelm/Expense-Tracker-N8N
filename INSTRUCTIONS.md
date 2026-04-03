# Automated Expense Tracker — Complete Setup Guide

> Turn bank SMS notifications into an automatically categorized Google Sheet — zero manual data entry.

This guide walks you through building the full system from scratch. Bank transaction SMS arrives on your phone → syncs to your Mac → gets parsed → you tap a category on Telegram → it's logged to Google Sheets with a live dashboard. You can also split any expense to Splitwise directly from the Telegram message.

**Time to set up: ~3 hours**

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Google Sheets Setup](#2-google-sheets-setup)
3. [Google Cloud Setup](#3-google-cloud-setup)
4. [Telegram Bot Setup](#4-telegram-bot-setup)
5. [n8n on Railway — Workflow A (Send)](#5-n8n-on-railway--workflow-a-send)
6. [n8n on Railway — Workflow B (Receive)](#6-n8n-on-railway--workflow-b-receive)
7. [Splitwise API + Mini App on Railway](#7-splitwise-api--mini-app-on-railway)
8. [Python Script Setup](#8-python-script-setup)
9. [launchd Auto-Start on Mac](#9-launchd-auto-start-on-mac)
10. [Mac Permissions and Settings](#10-mac-permissions-and-settings)
11. [End-to-End Test](#11-end-to-end-test)
12. [Troubleshooting](#12-troubleshooting)

---

## 1. Prerequisites

You need all of the following before starting. Do not skip any.

### Accounts to create

| Account | Where | Notes |
|---------|-------|-------|
| Google account | google.com | For Google Sheets + Google Cloud |
| Telegram account | telegram.org | Free, available on all platforms |
| Railway account | railway.app | Free tier works; Hobby plan recommended for always-on |
| Splitwise account | splitwise.com | Free, needed for expense splitting |
| GitHub account | github.com | For hosting the Mini App code |

### Hardware and OS

- **A Mac** (MacBook, iMac, Mac Mini — any Mac that stays awake during the day)
- **An iPhone** with bank SMS notifications enabled
- **Same Apple ID** on both Mac and iPhone
- **iCloud Messages enabled** on both devices

### Software to install on Mac

- **Python 3.10+** — install via [Anaconda](https://www.anaconda.com/download) or [Homebrew](https://brew.sh) (`brew install python`)
- **git** — comes with Xcode Command Line Tools (`xcode-select --install`)
- **Node.js** — not required for this project

### Verify iMessage sync is working

On your Mac, open the **Messages** app. You should see your bank SMS messages appearing there from your iPhone.

If bank SMS is not showing up:
1. On iPhone: Settings → Apple ID → iCloud → Messages → toggle **ON**
2. On Mac: Messages app → Settings → iMessage → check **Enable Messages in iCloud**
3. Wait 2–5 minutes for sync to complete
4. Send yourself a test SMS from any number and confirm it shows up on Mac

---

## 2. Google Sheets Setup

### 2.1 Create the spreadsheet

1. Go to [sheets.google.com](https://sheets.google.com)
2. Click **Blank** to create a new spreadsheet
3. Click the title at the top and rename it to something like `Finance Tally`
4. Right-click the sheet tab at the bottom (called `Sheet1`) → **Rename**
5. Name it: `2026 Consolidated`
   - If you're setting this up in a different year, use that year instead

### 2.2 Set up column headers

Click cell **A1** and enter the following headers across row 1:

| Cell | Header text |
|------|-------------|
| A1 | `Expense and receivable name` |
| B1 | `Description` |
| C1 | `Amount` |
| D1 | `Date` |
| E1 | `Bank` |
| F1 | `Type` |

**How data will look:**

- **A** — Category (e.g., `Food`, `Groceries`, `Transport`)
- **B** — Description you type in Telegram (blank if you skip with `-`)
- **C** — Amount. Debits: positive. Credits/refunds: **negative** numbers
- **D** — Date in `MM/DD/YYYY` format (a real date, not text)
- **E** — Bank name (e.g., `Jupiter`, `Kotak`, `BOB`)
- **F** — Transaction type: `debit` or `credit` (always lowercase)

### 2.3 Create the Dashboard sheet

1. Click the **`+`** button at the bottom of the spreadsheet to add a new sheet
2. Rename it: `Dashboard`

### 2.4 Set up the Month Selector

In the `Dashboard` sheet, enter:

| Cell | What to enter |
|------|---------------|
| A1 | `Month:` |
| B1 | Type the current month and year, e.g. `March 2026` |
| A3 | `Start Date:` |
| B3 | Paste this formula: `=DATE(VALUE(RIGHT(B1,4)),MATCH(LEFT(B1,FIND(" ",B1)-1),{"January","February","March","April","May","June","July","August","September","October","November","December"},0),1)` |
| A4 | `End Date:` |
| B4 | `=EOMONTH(B3,0)` |

> B1 is the only cell you'll change manually — change it to `April 2026` when the month rolls over. Everything else updates automatically.

### 2.5 Per-bank balance section

Set up a balance block for each bank. The example below uses Jupiter, Kotak, and BOB. Add or remove banks to match yours.

**Jupiter (rows 7–10):**

| Cell | Formula or content |
|------|--------------------|
| A7 | `Jupiter Spent:` |
| B7 | `=SUMIFS('2026 Consolidated'!C:C,'2026 Consolidated'!E:E,"Jupiter",'2026 Consolidated'!D:D,">="&$B$3,'2026 Consolidated'!D:D,"<="&$B$4,'2026 Consolidated'!F:F,"debit")` |
| A8 | `Jupiter Received:` |
| B8 | `=ABS(SUMIFS('2026 Consolidated'!C:C,'2026 Consolidated'!E:E,"Jupiter",'2026 Consolidated'!D:D,">="&$B$3,'2026 Consolidated'!D:D,"<="&$B$4,'2026 Consolidated'!F:F,"credit"))` |
| A9 | `Jupiter Op Bal:` |
| B9 | *(Enter your bank's opening balance manually — check your bank app)* |
| A10 | `Jupiter Current Bal:` |
| B10 | `=B9-B7+B8` |

**Kotak (rows 12–15):** Same structure — copy the Jupiter block and replace `"Jupiter"` with `"Kotak"` in each formula.

**BOB (rows 17–20):** Same structure — replace `"Jupiter"` with `"BOB"`.

### 2.6 Category breakdown table

This drives the pie chart. Starting at column D:

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

> These names **must exactly match** the category names in column A of the consolidated sheet. Case matters. No extra spaces.

In **E2**, enter this formula:
```
=SUMIFS('2026 Consolidated'!C:C,'2026 Consolidated'!A:A,D2,'2026 Consolidated'!D:D,">="&$B$3,'2026 Consolidated'!D:D,"<="&$B$4,'2026 Consolidated'!F:F,"debit")
```

Drag E2 down to E12 (all non-Refund rows).

**For the Refund row (E13)**, use a different formula that omits the `"debit"` filter — because refunds are credits (negative numbers):
```
=SUMIFS('2026 Consolidated'!C:C,'2026 Consolidated'!A:A,D13,'2026 Consolidated'!D:D,">="&$B$3,'2026 Consolidated'!D:D,"<="&$B$4)
```

In **E14** (Random):
```
=SUMIFS('2026 Consolidated'!C:C,'2026 Consolidated'!A:A,D14,'2026 Consolidated'!D:D,">="&$B$3,'2026 Consolidated'!D:D,"<="&$B$4,'2026 Consolidated'!F:F,"debit")
```

In **E15**: `=SUM(E2:E14)`

### 2.7 Opening balance table

This table stores opening balances for each month so the dashboard can show cascading balances.

Starting at column H, row 1:

| Cell | Header |
|------|--------|
| H1 | `Month` |
| I1 | `Jupiter Op Bal` |
| J1 | `Kotak Op Bal` |
| K1 | `BOB Op Bal` |

In H2, type the first month you want to track (e.g., `March 2026`). Enter the actual opening balances in I2, J2, K2 manually — check your bank apps for the balance at the start of that month.

For H3 onward (subsequent months), type the month name. For the balance columns (I3, J3, K3), the formula cascades the previous closing balance:

For **I3** (Jupiter opening balance for row 3's month):
```
=INDEX(Dashboard!B$9:B$9,1)+INDEX(Dashboard!B$8:B$8,1)-INDEX(Dashboard!B$7:B$7,1)
```

Actually the simpler approach: just manually enter the new opening balance at the start of each month (takes 30 seconds — check your bank app, copy the three numbers). The formulas on the Dashboard calculate current balance from there.

### 2.8 Pie chart

1. Select cells **D1:E14** (category headers through last category row, not the Total row)
2. Click **Insert → Chart**
3. Google Sheets will auto-detect a pie chart. If not, change chart type to **Pie chart**
4. Click **Customize** → **Pie chart** → set Slice label to **Percentage**
5. Position the chart anywhere on the dashboard

The pie chart automatically updates when the month in B1 changes or new transactions come in.

### 2.9 Copy the Spreadsheet ID

From the URL of your spreadsheet, copy the long ID string:
```
https://docs.google.com/spreadsheets/d/THIS_IS_THE_ID/edit
```
Save it — you'll need it when setting up n8n.

---

## 3. Google Cloud Setup

n8n needs OAuth credentials to write to your Google Sheet.

### 3.1 Create a project

1. Go to [console.cloud.google.com](https://console.cloud.google.com)
2. Click the project dropdown at the top → **New Project**
3. Name it anything (e.g., `expense-tracker`) → **Create**
4. Make sure the new project is selected in the dropdown

### 3.2 Enable the APIs

1. In the left sidebar: **APIs & Services → Library**
2. Search for **Google Sheets API** → click it → **Enable**
3. Go back to Library, search for **Google Drive API** → **Enable**

### 3.3 Configure OAuth consent screen

1. **APIs & Services → OAuth consent screen**
2. User Type: **External** → **Create**
3. Fill in:
   - App name: `Expense Tracker`
   - User support email: your Gmail
   - Developer contact email: your Gmail
4. Click **Save and Continue** through the Scopes screen (no changes needed)
5. On Test users: click **Add Users** → add your own Gmail → **Save**
6. Click **Save and Continue** → **Back to Dashboard**

### 3.4 Create OAuth credentials

1. **APIs & Services → Credentials → Create Credentials → OAuth Client ID**
2. Application type: **Web application**
3. Name: `n8n`
4. Under **Authorized redirect URIs**: you'll add n8n's URI here, but first deploy n8n (Part 5). Come back after Step 5.3 and add the URI that n8n shows you.
5. Click **Create**
6. A dialog shows your **Client ID** and **Client Secret** — save both

---

## 4. Telegram Bot Setup

### 4.1 Create the bot

1. Open Telegram and search for **@BotFather**
2. Tap **Start**, then send: `/newbot`
3. BotFather asks for a name — type anything (e.g., `Expense Tally Bot`)
4. BotFather asks for a username — must end in `bot` (e.g., `myexpensetally_bot`)
5. BotFather gives you a **bot token** that looks like: `7123456789:AAH1234abcdefghijklm`
6. Save this token — you'll use it in n8n

### 4.2 Get your Chat ID

1. Open your new bot in Telegram and send it any message (e.g., `hello`)
2. Open your browser and visit (replace with your actual token):
   ```
   https://api.telegram.org/bot<YOUR_BOT_TOKEN>/getUpdates
   ```
3. In the JSON response, find the number after `"chat":{"id":` — that's your **Chat ID**
4. Save it. It looks like `1269390790`

---

## 5. n8n on Railway — Workflow A (Send)

Workflow A receives transaction data from the Python script and sends you a Telegram message with category buttons.

### 5.1 Deploy n8n on Railway

1. Go to [railway.app](https://railway.app) and sign in
2. Click **New Project → Deploy a Template**
3. Search for **n8n** and click **Deploy**
4. Railway will build and deploy n8n. Wait ~2 minutes.
5. Click on your n8n service → **Settings → Networking → Generate Domain**
6. You'll get a public URL like `https://n8n-production-XXXX.up.railway.app`
7. Open that URL in your browser
8. Create your n8n account (email + password)

### 5.2 Connect Google Sheets credential

1. In n8n: **Settings → Credentials → Add Credential**
2. Search for **Google Sheets OAuth2 API** → select it
3. Enter the **Client ID** and **Client Secret** from Part 3
4. n8n shows a **Redirect URI** — copy it
5. Go back to Google Cloud Console → **APIs & Services → Credentials → your OAuth Client**
6. Under Authorized redirect URIs, click **Add URI** → paste the URI from n8n → **Save**
7. Back in n8n, click **Sign in with Google** → complete the OAuth flow
8. The credential should show as connected. Name it `Google Sheets`.

### 5.3 Connect Telegram credential

1. In n8n: **Settings → Credentials → Add Credential**
2. Search for **Telegram API** → select it
3. Enter your **bot token** from Part 4
4. Click **Save**. Name it `Telegram Bot`.

### 5.4 Create Workflow A

1. In n8n, click **Workflows → New Workflow**
2. Name it: `Expense Tracker - Send`

#### Node 1: Webhook

1. Click **Add first step** → search for **Webhook** → add it
2. Settings:
   - HTTP Method: `POST`
   - Path: leave as auto-generated (it will create a unique UUID path)
   - Respond: `When Last Node Finishes`
3. Click the node to see the full **Webhook URL** — it looks like:
   `https://your-n8n.up.railway.app/webhook/YOUR-UUID-HERE`
4. **Copy and save this URL** — you'll put it in the Python script later

#### Node 2: Telegram — Send Message

1. Click the **+** to add a node after Webhook
2. Search for **Telegram** → select **Send Message**
3. Credential: select your `Telegram Bot`
4. Settings:
   - Chat ID: paste your Chat ID from Part 4 (e.g., `1269390790`)
   - Parse Mode: `HTML`
5. In the **Text** field, paste exactly:
```
💳 New Transaction Detected

Bank: {{ $json.body.sender }}
Amount: ₹{{ $json.body.amount }}
Type: {{ $json.body.type }}
Date: {{ $json.body.date }}
SMS: {{ $json.body.message }}

Select a category:
```

6. Scroll down to **Additional Fields** → enable **Reply Markup**
7. Reply Markup type: **Inline Keyboard**
8. Add the following buttons. For each button:
   - Click **Add Item** under the keyboard
   - Set **Type** to `Callback Button` (except the last row which is a URL button)
   - Fill in **Text** and **Callback Data**

**Row 1** (3 buttons):
| Text | Callback Data |
|------|--------------|
| `🛒 Groceries` | `GR` |
| `🍔 Food` | `FD` |
| `🏢 Flat expenses` | `FE` |

**Row 2** (3 buttons):
| Text | Callback Data |
|------|--------------|
| `🛍 Shopping` | `SH` |
| `🥤 Beverages` | `BV` |
| `🎬 Entertainment` | `EN` |

**Row 3** (3 buttons):
| Text | Callback Data |
|------|--------------|
| `🚗 Transport` | `TP` |
| `💳 CC + previous dues` | `CC` |
| `💰 Investment` | `IN` |

**Row 4** (4 buttons):
| Text | Callback Data |
|------|--------------|
| `💈 Haircut` | `HC` |
| `🔄 Refund` | `RF` |
| `⛽ Petrol` | `PE` |
| `📝 Other` | `RD` |

**Row 5** (1 URL button — different type):
- Type: `URL Button`
- Text: `💸 Split this`
- URL:
```
https://YOUR-SPLITWISE-MINIAPP-URL.up.railway.app/?amount={{ $json.body.amount }}&description={{ $json.body.message }}&bank={{ $json.body.sender }}&date={{ $json.body.date }}
```
Replace `YOUR-SPLITWISE-MINIAPP-URL` with the Railway URL from Part 7. (You can come back and update this after Part 7.)

> Note: n8n's inline keyboard builder groups buttons into rows. Each time you add a button, there's a "Row" field — set it to 1, 2, 3, 4, or 5 to place the button in the correct row.

9. Click **Save** in the top right
10. Click the **Activate** toggle to turn the workflow on (it turns green)

---

## 6. n8n on Railway — Workflow B (Receive)

Workflow B handles your button taps, asks for a description, and writes the row to Google Sheets.

### 6.1 Create Workflow B

1. In n8n → **Workflows → New Workflow**
2. Name it: `Expense Tracker - Receive`

#### Node 1: Telegram Trigger

1. Add a **Telegram Trigger** node
2. Credential: select your `Telegram Bot`
3. Trigger On: check both **Callback Query** and **Message**
4. Click **Save**
5. n8n will show a **Webhook URL** for Telegram — copy it. It looks like:
   `https://your-n8n.up.railway.app/webhook/SOME-UUID/webhook`
6. Register this URL with Telegram by visiting in your browser (replace placeholders):
   ```
   https://api.telegram.org/bot<YOUR_BOT_TOKEN>/setWebhook?url=<N8N_TELEGRAM_WEBHOOK_URL>
   ```
7. You should see: `{"ok":true,"result":true}`
8. Verify it worked:
   ```
   https://api.telegram.org/bot<YOUR_BOT_TOKEN>/getWebhookInfo
   ```
   The `url` field should show your n8n URL.

#### Node 2: Switch

1. Add a **Switch** node, connect it after the Telegram Trigger
2. Add two rules:
   - **Rule 1** (Output 0 — callback):
     - Value 1: `{{ $json.callback_query ? 'callback' : 'message' }}`
     - Operation: `equals`
     - Value 2: `callback`
   - **Rule 2** (Output 1 — message):
     - Same expression
     - Operation: `equals`
     - Value 2: `message`

#### Node 3: Code — Save Transaction (connects to Switch Output 0)

This runs when you tap a category button. It stores the transaction details in workflow memory.

1. Add a **Code** node, connect it to **Switch Output 0**
2. Language: JavaScript
3. Paste this code exactly:

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

#### Node 4: Telegram — Send Message (connects to Node 3)

This asks you for a description after you tap a category.

1. Add a **Telegram** node, connect it after Node 3
2. Operation: **Send Message**
3. Credential: your `Telegram Bot`
4. Chat ID: `{{ $json.callback_query.message.chat.id }}`
5. Text: `Got it! Now send a description for this expense, or send - to skip.`

#### Node 5: Code — Retrieve Transaction (connects to Switch Output 1)

This runs when you send a description (or `-`). It builds the final row for Google Sheets.

1. Add a **Code** node, connect it to **Switch Output 1**
2. Paste this code exactly:

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
  let m = raw.match(/^(\d{2})-(\d{2})-(\d{4})$/);
  if (m) return `${m[2]}/${m[1]}/${m[3]}`;
  m = raw.match(/^(\d{2})-(\d{2})-(\d{2})$/);
  if (m) return `${m[2]}/${m[1]}/20${m[3]}`;
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

1. Add a **Google Sheets** node, connect it after Node 5
2. Operation: **Append Row**
3. Credential: your `Google Sheets` credential
4. Document: click the selector and find your spreadsheet (e.g., `Finance Tally`)
5. Sheet: select `2026 Consolidated`
6. Column mapping — map each column to its value:

| Sheet Column | Value |
|-------------|-------|
| A — Expense and receivable name | `{{ $json.category }}` |
| B — Description | `{{ $json.description }}` |
| C — Amount | `{{ $json.amount }}` |
| D — Date | `{{ $json.date }}` |
| E — Bank | `{{ $json.bank }}` |
| F — Type | `{{ $json.type }}` |

7. Click **Save** → click **Activate** (turn green)

---

## 7. Splitwise API + Mini App on Railway

This is optional but lets you split any expense to Splitwise directly from Telegram.

### 7.1 Get your Splitwise API key

1. Go to [secure.splitwise.com/oauth_clients](https://secure.splitwise.com/oauth_clients)
2. Click **Register your application**
3. Fill in any name (e.g., `expense-tracker-bot`) and a dummy callback URL (e.g., `https://localhost`)
4. After creating, you'll see an **API Key** — copy it
5. Also note your Splitwise user ID: go to [secure.splitwise.com/api/v3.0/get_current_user](https://secure.splitwise.com/api/v3.0/get_current_user) (you'll be prompted to authenticate) — find the `"id"` field at the top level

### 7.2 Create the Mini App project on GitHub

1. Create a new **public** GitHub repository (e.g., `splitwise-miniapp`)
2. Clone it locally:
   ```bash
   git clone https://github.com/YOUR_USERNAME/splitwise-miniapp.git
   cd splitwise-miniapp
   ```

### 7.3 Create the Flask app files

Create the following files inside the repo:

**`requirements.txt`:**
```
Flask==3.1.3
flask-cors==6.0.2
requests==2.33.1
gunicorn==25.3.0
```

**`Procfile`:**
```
web: gunicorn app:app --bind 0.0.0.0:$PORT
```

**`app.py`** — Replace `YOUR_USER_ID` with your Splitwise user ID from step 7.1:
```python
import os
import requests
from flask import Flask, jsonify, request, send_from_directory
from flask_cors import CORS

app = Flask(__name__, static_folder='static')
CORS(app)

SPLITWISE_API_KEY = os.environ.get('SPLITWISE_API_KEY')
SPLITWISE_BASE = 'https://secure.splitwise.com/api/v3.0'
YOUR_USER_ID = 46271821  # Replace with your Splitwise user ID

def sw_headers():
    return {'Authorization': f'Bearer {SPLITWISE_API_KEY}'}

@app.route('/')
def index():
    return send_from_directory('static', 'index.html')

@app.get('/groups')
def get_groups():
    resp = requests.get(f'{SPLITWISE_BASE}/get_groups', headers=sw_headers())
    data = resp.json()
    groups = [
        {
            'id': g['id'],
            'name': g['name'],
            'members': [
                {'id': m['id'], 'name': m['first_name']}
                for m in g['members']
            ]
        }
        for g in data['groups']
        if g['id'] != 0
    ]
    return jsonify(groups)

@app.post('/create-expense')
def create_expense():
    body = request.json
    group_id = body['group_id']
    description = body['description']
    total = float(body['total'])
    splits = body['splits']

    payload = {
        'cost': f'{total:.2f}',
        'description': description,
        'group_id': group_id,
        'currency_code': 'INR',
    }

    payload.update({
        'users__0__user_id':    YOUR_USER_ID,
        'users__0__paid_share': f'{total:.2f}',
        'users__0__owed_share': f'{splits["self_amount"]:.2f}',
    })

    for i, split in enumerate(splits['others']):
        idx = i + 1
        payload.update({
            f'users__{idx}__user_id':    split['user_id'],
            f'users__{idx}__paid_share': '0.00',
            f'users__{idx}__owed_share': f'{split["amount"]:.2f}',
        })

    resp = requests.post(
        f'{SPLITWISE_BASE}/create_expense',
        headers=sw_headers(),
        data=payload
    )
    return jsonify(resp.json()), resp.status_code

if __name__ == '__main__':
    port = int(os.environ.get('PORT', 5001))
    app.run(debug=True, port=port)
```

**`static/index.html`** — Create the `static/` directory and put this file inside it. This is the Telegram Mini App UI:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no" />
  <title>Split Expense</title>
  <script src="https://telegram.org/js/telegram-web-app.js"></script>
  <style>
    *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

    :root {
      --bg:        #17212b;
      --surface:   #232e3c;
      --surface2:  #2b3a4d;
      --accent:    #5288c1;
      --accent-dk: #3d6b9e;
      --text:      #e8e8e8;
      --text-muted:#7d8b99;
      --success:   #4fae4e;
      --error:     #e05c5c;
      --border:    #2d3f55;
      --radius:    12px;
      --font: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
    }

    html, body {
      height: 100%;
      background: var(--bg);
      color: var(--text);
      font-family: var(--font);
      font-size: 15px;
      line-height: 1.45;
    }

    body {
      display: flex;
      flex-direction: column;
      min-height: 100vh;
      padding-bottom: env(safe-area-inset-bottom);
    }

    #app {
      flex: 1;
      display: flex;
      flex-direction: column;
      max-width: 480px;
      width: 100%;
      margin: 0 auto;
      padding: 12px 12px 96px;
      gap: 10px;
    }

    .tx-card {
      background: var(--surface);
      border-radius: var(--radius);
      padding: 14px 16px;
      display: flex;
      flex-direction: column;
      gap: 4px;
    }

    .tx-amount { font-size: 26px; font-weight: 700; }
    .tx-amount span { color: var(--accent); }

    .tx-desc {
      font-size: 14px;
      color: var(--text-muted);
      white-space: nowrap;
      overflow: hidden;
      text-overflow: ellipsis;
    }

    .tx-desc-input {
      font-size: 14px;
      font-family: var(--font);
      font-weight: 500;
      color: var(--text);
      background: transparent;
      border: none;
      border-bottom: 1px solid var(--border);
      outline: none;
      width: 100%;
      padding: 1px 0 2px;
    }

    .tx-desc-input:focus { border-bottom-color: var(--accent); }

    .tx-meta { display: flex; gap: 10px; margin-top: 2px; }

    .tag {
      font-size: 11px;
      font-weight: 600;
      background: var(--surface2);
      color: var(--text-muted);
      border-radius: 6px;
      padding: 2px 8px;
      text-transform: uppercase;
    }

    .section-label {
      font-size: 11px;
      font-weight: 700;
      color: var(--text-muted);
      letter-spacing: 1px;
      text-transform: uppercase;
      padding: 4px 4px 0;
    }

    .group-list { display: flex; flex-direction: column; gap: 8px; }

    .group-card {
      background: var(--surface);
      border-radius: var(--radius);
      padding: 14px 16px;
      min-height: 56px;
      cursor: pointer;
      border: 1.5px solid transparent;
      display: flex;
      align-items: center;
      justify-content: space-between;
      touch-action: manipulation;
    }

    .group-card:active { background: var(--surface2); }
    .group-card:hover  { border-color: var(--accent); }
    .group-card .name { font-size: 15px; font-weight: 600; }
    .group-card .members-count { font-size: 12px; color: var(--text-muted); margin-top: 2px; }
    .group-card .chevron { color: var(--text-muted); font-size: 18px; }

    .step2-header { display: flex; align-items: center; gap: 10px; }

    .back-btn {
      background: var(--surface2);
      border: none;
      color: var(--accent);
      font-size: 14px;
      font-weight: 600;
      padding: 0 14px;
      min-height: 44px;
      border-radius: 8px;
      cursor: pointer;
      white-space: nowrap;
      display: inline-flex;
      align-items: center;
      touch-action: manipulation;
    }

    .group-title {
      font-size: 16px;
      font-weight: 700;
      flex: 1;
      white-space: nowrap;
      overflow: hidden;
      text-overflow: ellipsis;
    }

    .member-list { display: flex; flex-direction: column; gap: 8px; }

    .member-row {
      background: var(--surface);
      border-radius: var(--radius);
      padding: 10px 14px;
      min-height: 60px;
      display: flex;
      align-items: center;
      gap: 12px;
    }

    .member-row.you { border-left: 3px solid var(--accent); }

    .avatar {
      width: 36px;
      height: 36px;
      border-radius: 50%;
      background: var(--accent);
      display: flex;
      align-items: center;
      justify-content: center;
      font-size: 15px;
      font-weight: 700;
      flex-shrink: 0;
      color: #fff;
    }

    .member-name { flex: 1; font-size: 14px; font-weight: 500; }

    .you-badge {
      font-size: 10px;
      background: var(--accent);
      color: #fff;
      border-radius: 4px;
      padding: 1px 5px;
      margin-left: 4px;
      font-weight: 700;
    }

    .amount-input {
      width: 100px;
      background: var(--surface2);
      border: 1.5px solid var(--border);
      border-radius: 8px;
      color: var(--text);
      font-size: 16px;
      font-weight: 600;
      padding: 0 10px;
      min-height: 48px;
      text-align: right;
      outline: none;
      font-family: var(--font);
      -webkit-appearance: none;
      touch-action: manipulation;
    }

    .amount-input:focus { border-color: var(--accent); }

    .balance-bar {
      background: var(--surface);
      border-radius: var(--radius);
      padding: 12px 16px;
      display: flex;
      align-items: center;
      justify-content: space-between;
      gap: 8px;
    }

    .balance-label { font-size: 13px; color: var(--text-muted); }
    .balance-numbers { display: flex; align-items: baseline; gap: 4px; font-size: 15px; font-weight: 700; }
    .balance-total { color: var(--text); }
    .balance-sep   { color: var(--text-muted); font-weight: 400; }
    .balance-tx    { color: var(--text-muted); }

    .balance-diff {
      font-size: 12px;
      font-weight: 700;
      border-radius: 6px;
      padding: 3px 8px;
    }

    .balance-diff.ok    { background: rgba(79,174,78,0.15); color: var(--success); }
    .balance-diff.over  { background: rgba(224,92,92,0.15);  color: var(--error); }
    .balance-diff.under { background: rgba(130,130,130,0.12); color: var(--text-muted); }

    .submit-wrap {
      position: fixed;
      bottom: 0;
      left: 0;
      right: 0;
      padding: 12px 16px calc(16px + env(safe-area-inset-bottom));
      background: linear-gradient(to top, var(--bg) 75%, transparent);
      display: flex;
    }

    .submit-btn {
      flex: 1;
      max-width: 480px;
      margin: 0 auto;
      background: var(--accent);
      color: #fff;
      font-size: 16px;
      font-weight: 700;
      border: none;
      border-radius: var(--radius);
      padding: 0 14px;
      min-height: 52px;
      cursor: pointer;
      transition: background 0.15s, opacity 0.15s;
      touch-action: manipulation;
    }

    .submit-btn:hover    { background: var(--accent-dk); }
    .submit-btn:disabled { opacity: 0.4; cursor: not-allowed; }

    .spinner-wrap {
      flex: 1;
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: center;
      gap: 16px;
      padding: 40px;
    }

    .spinner {
      width: 36px;
      height: 36px;
      border: 3px solid var(--surface2);
      border-top-color: var(--accent);
      border-radius: 50%;
      animation: spin 0.8s linear infinite;
    }

    @keyframes spin { to { transform: rotate(360deg); } }
    .spinner-text { color: var(--text-muted); font-size: 14px; }

    .success-overlay {
      position: fixed;
      inset: 0;
      background: var(--bg);
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: center;
      gap: 16px;
      z-index: 100;
    }

    .success-icon {
      width: 72px;
      height: 72px;
      background: rgba(79,174,78,0.15);
      border-radius: 50%;
      display: flex;
      align-items: center;
      justify-content: center;
      font-size: 36px;
    }

    .success-title { font-size: 22px; font-weight: 700; }

    .success-sub {
      font-size: 14px;
      color: var(--text-muted);
      text-align: center;
      max-width: 280px;
    }

    .error-msg {
      background: rgba(224,92,92,0.12);
      color: var(--error);
      border-radius: 10px;
      padding: 12px 16px;
      font-size: 13px;
      text-align: center;
    }

    .hidden { display: none !important; }
  </style>
</head>
<body>
<div id="app"></div>
<script>
(function () {
  const tg = window.Telegram?.WebApp ?? null;
  if (tg) { tg.ready(); tg.expand(); }

  const params = new URLSearchParams(window.location.search);
  const TX = {
    amount:      parseFloat(params.get('amount'))  || 0,
    description: params.get('description')         || 'Expense',
    bank:        params.get('bank')                || '',
    date:        params.get('date')                || '',
  };

  // Replace with YOUR Splitwise user ID
  const MY_USER_ID = 46271821;
  const BASE = '';

  let groups = [];
  let selectedGrp = null;

  const app = document.getElementById('app');

  function fmt(n) { return '₹' + parseFloat(n).toFixed(2); }
  function initials(name) {
    return (name || '?').trim().split(/\s+/).map(w => w[0]).join('').slice(0, 2).toUpperCase();
  }
  function avatarColor(name) {
    const palette = ['#5288c1','#4f8b72','#8b6f5e','#7b68b2','#b05555','#4a8fa8'];
    let h = 0;
    for (let c of (name || '')) h = (h * 31 + c.charCodeAt(0)) & 0xffff;
    return palette[h % palette.length];
  }
  function escHtml(s) {
    return String(s).replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;')
      .replace(/"/g,'&quot;').replace(/'/g,'&#39;');
  }

  function txCardHTML(editableDesc = false) {
    const metaParts = [];
    if (TX.bank) metaParts.push(`<span class="tag">${escHtml(TX.bank)}</span>`);
    if (TX.date) metaParts.push(`<span class="tag">${escHtml(TX.date)}</span>`);
    const descHTML = editableDesc
      ? `<input id="tx-desc-input" class="tx-desc-input" type="text" value="${escHtml(TX.description)}" maxlength="200" />`
      : `<div class="tx-desc">${escHtml(TX.description)}</div>`;
    return `
      <div class="tx-card">
        <div class="tx-amount"><span>₹</span>${TX.amount.toFixed(2)}</div>
        ${descHTML}
        ${metaParts.length ? `<div class="tx-meta">${metaParts.join('')}</div>` : ''}
      </div>`;
  }

  function renderStep1() {
    app.innerHTML = `
      ${txCardHTML()}
      <div class="section-label">Choose a group</div>
      <div id="group-list" class="group-list">
        <div class="spinner-wrap"><div class="spinner"></div><div class="spinner-text">Loading groups…</div></div>
      </div>`;
    fetchGroups();
  }

  async function fetchGroups() {
    try {
      const res = await fetch(`${BASE}/groups`);
      if (!res.ok) throw new Error(`HTTP ${res.status}`);
      groups = await res.json();
      renderGroupList();
    } catch (e) {
      document.getElementById('group-list').innerHTML =
        `<div class="error-msg">Failed to load groups.<br>${escHtml(e.message)}</div>`;
    }
  }

  function renderGroupList() {
    const list = document.getElementById('group-list');
    if (!groups.length) { list.innerHTML = '<div class="error-msg">No groups found.</div>'; return; }
    list.innerHTML = groups.map(g => `
      <div class="group-card" data-id="${g.id}">
        <div>
          <div class="name">${escHtml(g.name)}</div>
          <div class="members-count">${g.members.length} member${g.members.length !== 1 ? 's' : ''}</div>
        </div>
        <div class="chevron">›</div>
      </div>`).join('');
    list.querySelectorAll('.group-card').forEach(card => {
      card.addEventListener('click', () => {
        selectedGrp = groups.find(g => g.id === parseInt(card.dataset.id));
        renderStep2();
      });
    });
  }

  function renderStep2() {
    const grp = selectedGrp;
    const perHead = grp.members.length > 0 ? (TX.amount / grp.members.length).toFixed(2) : '0.00';
    const membersHTML = grp.members.map(m => {
      const isYou = m.id === MY_USER_ID;
      const col = avatarColor(m.name);
      return `
        <div class="member-row ${isYou ? 'you' : ''}" data-uid="${m.id}">
          <div class="avatar" style="background:${col}">${initials(m.name)}</div>
          <div class="member-name">${escHtml(m.name)}${isYou ? '<span class="you-badge">YOU</span>' : ''}</div>
          <input class="amount-input" type="number" inputmode="decimal" min="0" step="0.01" value="${perHead}" data-uid="${m.id}" />
        </div>`;
    }).join('');

    app.innerHTML = `
      ${txCardHTML(true)}
      <div class="step2-header">
        <button class="back-btn" id="back-btn">← Back</button>
        <div class="group-title">${escHtml(grp.name)}</div>
      </div>
      <div class="section-label">Who owes what?</div>
      <div class="member-list">${membersHTML}</div>
      <div class="balance-bar" id="balance-bar">
        <span class="balance-label">Split total</span>
        <div style="display:flex;align-items:center;gap:8px">
          <div class="balance-numbers">
            <span class="balance-total" id="bal-split">₹0.00</span>
            <span class="balance-sep">/</span>
            <span class="balance-tx">${fmt(TX.amount)}</span>
          </div>
          <span class="balance-diff under" id="bal-diff">—</span>
        </div>
      </div>
      <div id="submit-error" class="error-msg hidden"></div>`;

    const wrap = document.createElement('div');
    wrap.className = 'submit-wrap';
    wrap.innerHTML = '<button class="submit-btn" id="submit-btn">Submit Split</button>';
    document.body.appendChild(wrap);

    document.getElementById('back-btn').addEventListener('click', () => { wrap.remove(); renderStep1(); });

    const descInput = document.getElementById('tx-desc-input');
    descInput.addEventListener('focus', () => descInput.select());
    descInput.addEventListener('keydown', e => { if (e.key === 'Enter') descInput.blur(); });

    updateBalance();
    app.querySelectorAll('.amount-input').forEach(inp => inp.addEventListener('input', updateBalance));
    document.getElementById('submit-btn').addEventListener('click', handleSubmit);
  }

  function getSplitValues() {
    const inputs = app.querySelectorAll('.amount-input');
    const splits = [];
    inputs.forEach(inp => splits.push({ uid: parseInt(inp.dataset.uid), amount: parseFloat(inp.value) || 0 }));
    return splits;
  }

  function updateBalance() {
    const splits = getSplitValues();
    const total = splits.reduce((s, x) => s + x.amount, 0);
    const diff = total - TX.amount;
    const balanced = Math.abs(diff) < 0.01;
    document.getElementById('bal-split').textContent = fmt(total);
    const diffEl = document.getElementById('bal-diff');
    if (balanced) { diffEl.textContent = 'Balanced ✓'; diffEl.className = 'balance-diff ok'; }
    else if (diff > 0) { diffEl.textContent = `+${fmt(diff)} over`; diffEl.className = 'balance-diff over'; }
    else { diffEl.textContent = `${fmt(diff)} short`; diffEl.className = 'balance-diff under'; }
    const btn = document.getElementById('submit-btn');
    if (btn) btn.disabled = !balanced;
  }

  async function handleSubmit() {
    const splits = getSplitValues();
    const total = splits.reduce((s, x) => s + x.amount, 0);
    const errEl = document.getElementById('submit-error');
    const submitBtn = document.getElementById('submit-btn');
    errEl.classList.add('hidden');

    if (Math.abs(total - TX.amount) > 0.01) {
      errEl.textContent = `Split total (${fmt(total)}) must equal transaction amount (${fmt(TX.amount)}).`;
      errEl.classList.remove('hidden');
      return;
    }

    const selfSplit = splits.find(s => s.uid === MY_USER_ID);
    const otherSplits = splits.filter(s => s.uid !== MY_USER_ID);

    if (!selfSplit) {
      errEl.textContent = 'Could not find your user in this group.';
      errEl.classList.remove('hidden');
      return;
    }

    const description = document.getElementById('tx-desc-input')?.value.trim() || TX.description;
    const payload = {
      group_id: selectedGrp.id,
      description,
      total: TX.amount,
      splits: {
        self_amount: selfSplit.amount,
        others: otherSplits.map(s => ({ user_id: s.uid, amount: s.amount })),
      },
    };

    submitBtn.disabled = true;
    submitBtn.textContent = 'Submitting…';

    try {
      const res = await fetch(`${BASE}/create-expense`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload),
      });
      if (!res.ok) {
        const data = await res.json().catch(() => ({}));
        throw new Error(data?.error ?? `HTTP ${res.status}`);
      }
      showSuccess();
    } catch (e) {
      errEl.textContent = `Error: ${e.message}`;
      errEl.classList.remove('hidden');
      submitBtn.disabled = false;
      submitBtn.textContent = 'Submit Split';
    }
  }

  function showSuccess() {
    document.querySelector('.submit-wrap')?.remove();
    const overlay = document.createElement('div');
    overlay.className = 'success-overlay';
    overlay.innerHTML = `
      <div class="success-icon">✓</div>
      <div class="success-title">Expense Added!</div>
      <div class="success-sub">
        ${escHtml(document.getElementById('tx-desc-input')?.value.trim() || TX.description)}<br>
        <strong>${fmt(TX.amount)}</strong> split across ${selectedGrp.members.length} people in <strong>${escHtml(selectedGrp.name)}</strong>.
      </div>`;
    document.body.appendChild(overlay);
    setTimeout(() => {
      if (tg) tg.close();
      else overlay.innerHTML += '<div style="color:var(--text-muted);font-size:13px;margin-top:8px">(Close this tab)</div>';
    }, 1800);
  }

  renderStep1();
})();
</script>
</body>
</html>
```

### 7.4 Create `.env.example`

```
SPLITWISE_API_KEY=your_splitwise_api_key_here
```

### 7.5 Push to GitHub and deploy on Railway

```bash
git add .
git commit -m "Initial Mini App"
git push
```

Now deploy on Railway:

1. Go to [railway.app](https://railway.app) → **New Project → Deploy from GitHub repo**
2. Connect your GitHub account if prompted
3. Select your `splitwise-miniapp` repository
4. Railway auto-detects the Procfile and deploys
5. Go to **Settings → Networking → Generate Domain** → copy the URL (e.g., `https://splitwise-miniapp-production.up.railway.app`)
6. Go to **Variables → Add Variable**:
   - Name: `SPLITWISE_API_KEY`
   - Value: your API key from step 7.1
7. Click **Deploy** to redeploy with the variable

### 7.6 Set up Telegram Mini App access

For the `💸 Split this` button to open as a proper Telegram Mini App (full screen inside Telegram):

1. Open **@BotFather** in Telegram
2. Send `/newapp`
3. Select your expense tracker bot
4. Title: anything (e.g., `Split Expense`)
5. Description: anything
6. Photos: skip (send `/empty` if it asks)
7. URL: your Railway Mini App URL (e.g., `https://splitwise-miniapp-production.up.railway.app`)
8. Short name: anything (e.g., `split`)

Now go back to n8n Workflow A, Node 2 and update the Row 5 button URL to your actual Railway URL.

---

## 8. Python Script Setup

The Python script runs on your Mac, polls iMessage's database for new bank SMS, parses them, and sends data to n8n.

### 8.1 Install dependencies

```bash
pip install requests
```

`sqlite3` is built into Python — no install needed.

### 8.2 Create the script file

Create a directory and file for the script:

```bash
mkdir -p ~/expense-tracker/scripts
```

Create `~/expense-tracker/scripts/tracker.py` with the following content. **You must replace:**
- `WEBHOOK_URL` with your actual Workflow A webhook URL from Step 5.4
- `TRACKED_SENDERS` with the sender IDs for your banks (see note below)
- The `parse_sms()` function logic to match your bank's SMS format

```python
import sqlite3
import json
import time
import re
import requests
from pathlib import Path

# Replace with your n8n Workflow A webhook URL
WEBHOOK_URL = "https://your-n8n-instance.up.railway.app/webhook/YOUR-WEBHOOK-UUID"

DB_PATH = Path.home() / "Library/Messages/chat.db"
STATE_FILE = Path.home() / ".expense_tracker_state.json"
POLL_INTERVAL = 10  # seconds between polls

# Replace with the SMS sender IDs for YOUR banks.
# To find them: open Messages on Mac, look at who sent the bank SMS.
# Format is usually like "AD-HDFCBK", "VK-KOTAKB-S", "JM-FEDBNK-T"
# We match with endswith() to handle carrier prefixes (AD-, VK-, JM- etc.)
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
    """
    SMS text in chat.db is stored in a binary 'attributedBody' column,
    not the plain 'text' column. This decoder extracts the readable text.
    """
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
    """Extract rupee amount from SMS. Modify the regex if your bank uses a different format."""
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
    """
    Parse a bank SMS and return a dict with: sender, amount, type, date, message.
    Returns None if the SMS cannot be parsed or is not a transaction.

    THIS IS THE MAIN FUNCTION YOU NEED TO CUSTOMIZE FOR YOUR BANKS.
    Study a few SMS messages from each bank and figure out:
    - How to detect debit vs credit (keywords like "debited", "credited", "Dr.", "Cr.")
    - Where the date is and what format it's in
    """
    normalised = next((s for s in TRACKED_SENDERS if sender.endswith(s)), None)
    if normalised is None:
        return None

    amount = extract_amount(text)
    if not amount:
        return None

    text_lower = text.lower()

    # ── Jupiter (Federal Bank) ──────────────────────────────────────────
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

    # ── Kotak 811 ────────────────────────────────────────────────────────
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

    # ── Bank of Baroda ───────────────────────────────────────────────────
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
            print(f"[OK] Sent: Rs.{payload['amount']} {payload['type']} via {payload['sender']}")
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

**Finding your bank sender IDs:**
Open the Messages app on your Mac. Find a bank SMS and look at the sender — it will be something like `AD-HDFCBK` or `VK-KOTAKB-S`. Add the base part (after the carrier prefix) to `TRACKED_SENDERS`. The script uses `endswith()` matching, so `"HDFCBK"` matches both `AD-HDFCBK` and `JM-HDFCBK`.

### 8.3 Test manually

```bash
python3 ~/expense-tracker/scripts/tracker.py
```

Expected output:
```
Expense Tracker started. Polling every 10 seconds...
Watching: FEDBNK-S, FEDBNK-T, KOTAKB-S, BOBSMS-S, BOBTXN-S
Press Ctrl+C to stop.
```

If there are existing bank SMS in your iMessage history, you may see some being processed on the first run. Make a small test transaction (e.g., UPI ₹1) to verify the full end-to-end flow works.

---

## 9. launchd Auto-Start on Mac

This makes the script start automatically on login and restart if it crashes — without Terminal needing to be open.

### 9.1 Find your Python path

```bash
which python3
```

Note the output (e.g., `/opt/anaconda3/bin/python3.12` or `/usr/local/bin/python3`).

### 9.2 Find your macOS username

```bash
whoami
```

### 9.3 Create the plist file

Create the file `~/Library/LaunchAgents/com.expensetracker.plist`:

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
        <!-- Replace with your Python path from: which python3 -->
        <string>/opt/anaconda3/bin/python3.12</string>
        <!-- Replace YOURUSERNAME with output of: whoami -->
        <string>/Users/YOURUSERNAME/expense-tracker/scripts/tracker.py</string>
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

Replace `YOURUSERNAME` in all four places.

### 9.4 Load and start

```bash
launchctl load ~/Library/LaunchAgents/com.expensetracker.plist
```

### 9.5 Verify it's running

```bash
launchctl list | grep expensetracker
```

You should see a line with a PID (a number in the first column). If you see a `-`, it crashed — check the error log.

### 9.6 Manage the script

```bash
# Stop
launchctl unload ~/Library/LaunchAgents/com.expensetracker.plist

# Restart
launchctl unload ~/Library/LaunchAgents/com.expensetracker.plist
launchctl load ~/Library/LaunchAgents/com.expensetracker.plist

# View live output
tail -f ~/Library/Logs/expensetracker.log

# View errors
tail -f ~/Library/Logs/expensetracker.error.log
```

---

## 10. Mac Permissions and Settings

### 10.1 Grant Full Disk Access to Python

The iMessage database (`~/Library/Messages/chat.db`) is protected by macOS. Python needs explicit Full Disk Access permission to read it.

1. Go to **System Settings → Privacy & Security → Full Disk Access**
2. Click the **`+`** button
3. Use **Cmd+Shift+G** to go to a specific path, type your Python path (e.g., `/opt/anaconda3/bin/python3.12`)
4. Click **Open**
5. Make sure the toggle next to your Python binary is **ON**

> The exact Python binary you grant access to must match what's in the plist `ProgramArguments`. If they differ, the script will fail to read chat.db.

### 10.2 Prevent Mac from sleeping

The script stops working if your Mac goes to sleep (not just locked — sleep stops all processes).

1. Go to **System Settings → Battery**
2. Enable: **Prevent your Mac from automatically sleeping when the display is off**

> Locking your Mac (password screen) is fine. Sleep mode is not.

### 10.3 Verify iCloud Messages

Check that bank SMS messages are syncing to your Mac:

1. Open **Messages** app on Mac
2. Look for SMS from your banks — they should appear there
3. If not: on iPhone → Settings → Apple ID → iCloud → Messages → **ON**
4. On Mac → Messages → Settings → iMessage → **Enable Messages in iCloud**

---

## 11. End-to-End Test

Run through these steps in order to verify everything is working.

1. **Check the Python script is running:**
   ```bash
   launchctl list | grep expensetracker
   ```
   Should show a PID (number), not `-`.

2. **Check n8n workflows are active:**
   - Open your n8n dashboard
   - Both `Expense Tracker - Send` and `Expense Tracker - Receive` should have a green active toggle

3. **Make a small test transaction:**
   - Make a ₹1 UPI payment from one of your tracked bank accounts (or any real transaction)
   - Wait for the bank SMS to arrive on your iPhone

4. **Watch it flow through:**
   - SMS appears on iPhone → syncs to Mac Messages → Python script detects it within 10 seconds
   - You receive a Telegram message with the transaction details and category buttons

5. **Tap a category button in Telegram**

6. **Send a description** (or `-` to skip)

7. **Check Google Sheets:**
   - Open your spreadsheet → `2026 Consolidated` sheet
   - A new row should have appeared at the bottom

8. **Check the Dashboard:**
   - Switch to the `Dashboard` sheet
   - The category amounts and pie chart should reflect the new transaction

9. **Test the Split button (optional):**
   - Tap `💸 Split this` in the Telegram message
   - The Mini App should open inside Telegram showing your groups
   - Select a group, adjust the split, tap Submit
   - Check Splitwise — the expense should appear there

---

## 12. Troubleshooting

### "chat.db not found" error in logs

- Full Disk Access is not granted to the Python binary
- Check System Settings → Privacy & Security → Full Disk Access
- Run `which python3` — the path shown must exactly match what you added in Full Disk Access

### Bank SMS not showing up in Mac Messages

- iCloud Messages is not enabled on iPhone or Mac (see Section 10.3)
- Both devices must be signed into the same Apple ID
- Toggle iCloud Messages off and back on, wait 5 minutes

### Python script starts then immediately crashes (PID shows `-`)

```bash
tail -20 ~/Library/Logs/expensetracker.error.log
```
Common causes: wrong Python path in plist, wrong script path in plist, missing `requests` library.

### n8n webhook returns 404 or timeout

- Make sure Workflow A is **Activated** (green toggle)
- The WEBHOOK_URL in tracker.py must match exactly what n8n shows — copy-paste it, don't type it

### Telegram bot sends no message

- Check n8n execution history (click on Workflow A → Executions) for errors
- Verify the Chat ID in Node 2 matches your actual Telegram Chat ID from Part 4
- Re-check the Telegram credential is valid

### Tapping category button does nothing

- Telegram webhook may not be registered. Visit:
  ```
  https://api.telegram.org/bot<TOKEN>/getWebhookInfo
  ```
  The `url` field should contain your n8n Telegram Trigger URL

### Google Sheets not getting new rows

- Check n8n Workflow B execution history for errors on Node 6
- Google Sheets credential may need re-authentication (tokens expire)
- Verify the sheet name in Node 6 matches exactly: `2026 Consolidated`

### Dashboard formulas show 0 despite data in the sheet

- Category name mismatch — the name in Column D of Dashboard must **exactly** match Column A of the consolidated sheet (case-sensitive, no trailing spaces)
- Date format issue — dates in Column D must be real dates, not text strings. Click a date cell. If the formula bar shows `'03/15/2026` (with a leading quote), it's stored as text and SUMIFS won't find it. The n8n `toSheetDate` function produces `MM/DD/YYYY` which Google Sheets correctly parses as a date.

### Splitwise Mini App shows "Failed to load groups"

- Check the `SPLITWISE_API_KEY` environment variable is set correctly in Railway
- Verify the API key works: `curl -H "Authorization: Bearer YOUR_KEY" https://secure.splitwise.com/api/v3.0/get_current_user`

### Mac goes to sleep and script stops

- Enable "Prevent your Mac from automatically sleeping" in System Settings → Battery

### Script processes 100+ old messages on first run

- This is normal. The script processes the last 200 messages in chat.db on first run.
- Wait for it to finish, then delete any unwanted rows from the Google Sheet.
- The state file (`~/.expense_tracker_state.json`) records what's been processed — future runs start from where it left off.

### Port conflict when running Flask locally (port 5000 busy)

- macOS uses port 5000 for AirPlay Receiver
- Use port 5001 for local dev: `SPLITWISE_API_KEY=xxx python app.py`
- The `app.py` already defaults to port 5001 when run directly

---

## Customization Guide

### Adding a new bank

1. Find the sender ID (open Messages on Mac, look at the bank SMS sender)
2. Add it to `TRACKED_SENDERS` in the Python script
3. Add a new `elif normalised == "YOUR_SENDER_ID":` block in `parse_sms()` — look at the existing blocks for the pattern
4. Update `TRACKED_SENDERS` and restart: `launchctl unload ... && launchctl load ...`
5. Optionally add a new bank section to the Dashboard sheet

### Adding or changing categories

1. Add/remove buttons in n8n Workflow A, Node 2 (inline keyboard)
2. Update `categoryMap` in Workflow B, Node 5 (add the new code → name mapping)
3. Add the new category name to Dashboard column D
4. Make sure the name in the `categoryMap` **exactly** matches column D

### Using different currency

The `extract_amount()` function in tracker.py looks for `Rs.` followed by a number. Change the regex for your currency:
- USD: `r'\$\s*([\d,]+(?:\.\d{1,2})?)'`
- EUR: `r'€\s*([\d,]+(?:\.\d{1,2})?)'`
- GBP: `r'£\s*([\d,]+(?:\.\d{1,2})?)'`

Also change `'currency_code': 'INR'` in `app.py` to match your currency.

### Month-end process (30 seconds)

1. Note the closing balance for each bank from the Dashboard
2. Change cell B1 to the new month (e.g., `April 2026`)
3. Enter the closing balances from last month as the new opening balances (B9, B14, B19)
4. Done — all formulas and the pie chart auto-update
