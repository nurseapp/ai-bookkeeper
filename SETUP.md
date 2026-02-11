# ChatBooks - n8n Cloud Setup Guide

WhatsApp receipt scanner powered by Claude Vision + Google Sheets, running on n8n Cloud.

## How It Works

1. Send a receipt photo via WhatsApp
2. Claude Vision extracts merchant, date, total, items, category
3. Data is recorded in Google Sheets with a live dashboard
4. You get a formatted confirmation back on WhatsApp

Plus 7 text commands: `help`, `summary`, `budget`, `last 5`, `total`, `delete last`, `categories`

---

## Prerequisites

- [n8n Cloud account](https://n8n.io/cloud/) (or self-hosted n8n)
- [Twilio account](https://www.twilio.com/) with WhatsApp Sandbox enabled
- [Anthropic API key](https://console.anthropic.com/) for Claude Vision
- [Google Cloud project](https://console.cloud.google.com/) with Sheets API enabled + service account

---

## Step 1: Create Google Spreadsheet

1. Create a new Google Spreadsheet and name it **"ChatBooks"**
2. Create 5 sheets (tabs) with these exact names:
   - **Transactions** - Add headers in row 1: `ID | Timestamp | Phone Number | Merchant | Receipt Date | Category | Currency | Subtotal | Tax | Tip | Total | Payment Method | Line Items | Confidence | Notes | Month`
   - **Monthly Summary** - Leave empty (formulas auto-populate)
   - **Categories** - Import `setup/categories-data.csv` (File > Import > Upload)
   - **Dashboard** - Add these formulas:
     - A1: `ChatBooks Dashboard`
     - A4: `Total Spent` / B4: `=SUMIFS(Transactions!K:K,Transactions!P:P,TEXT(TODAY(),"YYYY-MM"))`
     - A5: `Transaction Count` / B5: `=COUNTIFS(Transactions!P:P,TEXT(TODAY(),"YYYY-MM"))`
     - A6: `Daily Average` / B6: `=IFERROR(B4/DAY(TODAY()),0)`
   - **ConversationState** - Add headers in row 1: `phoneNumber | pendingAction | pendingTransactionJson | lastInteraction`

3. Note your **Spreadsheet ID** (from the URL: `https://docs.google.com/spreadsheets/d/SPREADSHEET_ID/edit`)

4. Share the spreadsheet with your Google service account email (Editor access)

---

## Step 2: Set Up Google Service Account

1. Go to [Google Cloud Console](https://console.cloud.google.com/)
2. Create a project (or use existing)
3. Enable the **Google Sheets API**
4. Go to **IAM & Admin > Service Accounts** > Create Service Account
5. Download the JSON key file
6. Share your spreadsheet with the service account email (e.g., `bookkeeper@project.iam.gserviceaccount.com`)

---

## Step 3: Configure n8n Credentials

In n8n Cloud, go to **Settings > Credentials** and create these 4 credentials:

### 3a. Twilio API
- Type: **Twilio API**
- Account SID: Your Twilio Account SID
- Auth Token: Your Twilio Auth Token

### 3b. Google Sheets (Service Account)
- Type: **Google Sheets API** (Service Account)
- Paste the full JSON key from Step 2

### 3c. Anthropic API (Header Auth)
- Type: **Header Auth**
- Name: `Anthropic API`
- Header Name: `x-api-key`
- Header Value: Your Anthropic API key (sk-ant-...)

### 3d. Twilio Media Auth (Basic Auth)
- Type: **HTTP Basic Auth**
- Name: `Twilio Media Auth`
- Username: Your Twilio Account SID
- Password: Your Twilio Auth Token

---

## Step 4: Set n8n Environment Variables

In n8n Cloud, go to **Settings > Variables** and add:

| Variable | Value |
|----------|-------|
| `GOOGLE_SPREADSHEET_ID` | Your spreadsheet ID from Step 1 |
| `TWILIO_WHATSAPP_NUMBER` | `whatsapp:+14155238886` (sandbox) or your production number |

---

## Step 5: Import the Workflow

1. In n8n, go to **Workflows > Import from File**
2. Select `n8n-workflow.json`
3. After import, open the workflow and update each node's credentials:
   - All **Google Sheets** nodes → select your Google Sheets credential
   - All **Twilio** nodes → select your Twilio API credential
   - Both **Claude** HTTP Request nodes → select your Anthropic API Header Auth credential
   - **Download Image** HTTP Request → select your Twilio Media Auth (Basic Auth) credential
4. **Activate** the workflow (toggle in top-right)

---

## Step 6: Configure Twilio Webhook

1. Copy the webhook URL from the **Twilio Webhook** node in n8n (click the node, it shows the URL)
   - It will look like: `https://your-instance.app.n8n.cloud/webhook/twilio-whatsapp`
2. Go to [Twilio Console > Messaging > Try it out > Send a WhatsApp message](https://console.twilio.com/us1/develop/sms/try-it-out/whatsapp-learn)
3. Set **"WHEN A MESSAGE COMES IN"** to your webhook URL (POST)
4. Join the sandbox by sending the join code from your WhatsApp to the Twilio sandbox number

---

## Step 7: Test It

1. Send `help` via WhatsApp → should receive the command list
2. Send a receipt photo → should receive extracted data confirmation
3. Send `summary` → should receive monthly spending breakdown
4. Send `last 5` → should show recent transactions
5. Open your Google Sheet → verify the Dashboard is updating

---

## Troubleshooting

**"I'm not getting any response"**
- Check that the workflow is activated (toggle on)
- Verify the Twilio webhook URL matches n8n's webhook URL
- Check n8n execution logs for errors

**"Image processing fails"**
- Ensure the Anthropic API key is valid and has credits
- Check that the Header Auth credential name matches what's in the HTTP Request nodes
- Very large images (>5MB) may need to be resized before sending

**"Google Sheets errors"**
- Verify the service account email has Editor access to the spreadsheet
- Ensure the Spreadsheet ID environment variable is correct
- Check that all 5 sheet names match exactly (case-sensitive)

**"Duplicate detection not working"**
- The ConversationState sheet must have the exact headers: `phoneNumber | pendingAction | pendingTransactionJson | lastInteraction`

---

## Customization

### Change budget limits
Edit the **Categories** sheet in Google Sheets. Change the Monthly Budget column values.

### Add a new category
1. Add a row to the Categories sheet
2. Update the `CATEGORIES` array in the **Process Extraction** Code node in n8n

### Change the Claude model
Edit both HTTP Request nodes (Claude Validate Image, Claude Extract Receipt) and change `claude-sonnet-4-20250514` to your preferred model.

### Change confidence threshold
In the **Format Confirmation** Code node, change `0.7` on the line `if (extraction.confidence.overall < 0.7)`.
