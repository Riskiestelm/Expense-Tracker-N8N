# Expense Tracker Agent

Automatically log every bank transaction to Google Sheets by tapping one button on Telegram.

---

## Why I built this

I was manually logging every bank transaction into a spreadsheet every few days. Open Jupiter, copy the last 10 transactions, switch to Kotak, do the same, open the sheet, paste, categorise. It took 20–30 minutes whenever I finally sat down to do it, and I kept forgetting.

This automates the whole thing. Every SMS that arrives triggers a Telegram notification with category buttons. One tap, optionally type a description, and the row is in Google Sheets. The dashboard updates automatically — per-bank balances, monthly category breakdown, pie chart.

---

## How it works

There are two ways to get SMS into the system. Everything after that is identical.

**Mac method:**
A Python script running on your Mac polls the iMessage `chat.db` file every 10 seconds. When a new bank SMS appears, it parses the amount, type, date, and bank name, then POSTs that structured data to an n8n webhook hosted on Railway. n8n sends you a Telegram message with inline category buttons. You tap one, type a description (or `-` to skip), and n8n appends a row to Google Sheets.

**iPhone-only method:**
Two iOS Shortcuts automations watch for incoming messages containing `Rs` or `INR`. When a bank SMS matches, the shortcut fires automatically and POSTs the raw message text and sender to the same n8n webhook. The n8n Code node does the parsing — amount extraction, bank detection, credit/debit classification, date formatting. The rest of the flow is the same as the Mac method.

---

## What you get

- A Telegram notification for every bank transaction, within seconds of the SMS arriving
- 13 category buttons in the notification — one tap to categorise
- Google Sheets dashboard with per-bank balances (spent, received, current balance), monthly category breakdown with SUMIFS, and a pie chart
- Change the month in one cell and everything recalculates
- Optional: a `💸 Split this` button in every Telegram message that opens a Splitwise mini app to split the expense with a group

Tested with Jupiter (Federal Bank), Kotak 811, Kotak Credit Card, and Bank of Baroda. Works with any bank that sends SMS transaction alerts — you just need to check that the SMS contains `Rs` or `INR`.

---

## How to set it up

Before you start, read the prerequisites section in the relevant instruction file — you'll need a Railway account, a Telegram bot, Google Sheets, and a Google Cloud project for OAuth.

Two instruction files are included:

- **[Mac method](Mac%20Sync%20Version/INSTRUCTIONS.md)** — uses a Python script + launchd on Mac, requires iMessage sync between iPhone and Mac
- **[iPhone-only method](iPhone%20only%20version/Instructions%20for%20only%20iPhone.md)** — uses iOS Shortcuts, no Mac or Python required

---

## Tech stack

| Tool | Role |
|------|------|
| **n8n** (Railway) | Automation engine — parses SMS, sends Telegram buttons, writes to Sheets |
| **Railway** | Hosting for n8n and the optional Splitwise mini app |
| **Python** (Mac method) | Polls `chat.db` every 10 seconds for new bank SMS |
| **iOS Shortcuts** (iPhone method) | Fires on bank SMS, POSTs raw message to n8n |
| **Telegram Bot API** | User interface — category buttons, description prompt |
| **Google Sheets** | Data store and dashboard |
| **Flask + Splitwise API** | Optional mini app for splitting expenses |
