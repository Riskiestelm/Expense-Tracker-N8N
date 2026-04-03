# Expense Tracker Agent

Bank SMS arrives on your iPhone → syncs to your Mac → you tap a category on Telegram → it's logged to Google Sheets automatically. No manual data entry. Works with any Indian bank that sends SMS alerts.

Includes a Splitwise Mini App so you can split any expense to a group directly from the same Telegram message.

---

## How it works

```
Bank SMS
  │
  ▼ iCloud sync
Mac (chat.db)
  │
  ▼ Python script polls every 10s
n8n on Railway
  │
  ├──▶ Telegram message with category buttons
  │         │
  │         ▼ you tap a button + type description
  │
  ├──▶ Google Sheets (new row logged automatically)
  │
  └──▶ 💸 Split this button
            │
            ▼ Telegram Mini App opens
       Splitwise expense created
```

---

## Tech stack

| Tool | Role |
|------|------|
| **Python** (Mac) | Polls `~/Library/Messages/chat.db` every 10 seconds for new bank SMS |
| **n8n** (Railway) | Automation engine — receives SMS data, sends Telegram buttons, writes to Sheets |
| **Telegram Bot** | User interface — category buttons, description prompt |
| **Google Sheets** | Data store + live dashboard with per-bank balances and category pie chart |
| **Flask + Railway** | Hosts the Splitwise Mini App (group picker, split editor) |
| **Splitwise API** | Creates shared expenses on submission |
| **launchd** | Keeps the Python script running on Mac boot |

---

## Setup

Full step-by-step setup guide (zero prior knowledge assumed): **[INSTRUCTIONS.md](INSTRUCTIONS.md)**

Covers: Google Sheets formulas, Google Cloud OAuth, Telegram bot, n8n workflow configs with all code, Splitwise Mini App deployment, Python script, launchd plist, Mac permissions, end-to-end test, and troubleshooting.

**Setup time: ~3 hours**

---

## Status

- Tested with Jupiter (Federal Bank), Kotak 811, and Bank of Baroda
- Runs on any Mac with iMessage sync enabled
- Adaptable to other banks — see the Customization section in INSTRUCTIONS.md
