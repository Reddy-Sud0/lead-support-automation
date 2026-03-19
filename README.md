# Lead-to-Support Automation — n8n Workflow

A production-grade n8n workflow that converts inbound leads into a structured support pipeline with validation, enrichment, routing, storage, notifications, and daily digests.

---

## 📁 Repository Structure

```
lead-support-automation/
├── n8n workflow/
│   └── lead-to-support-automation.json   # Import this into n8n
├── sample-payloads/
│   ├── payload-01-high-urgency-company.json
│   ├── payload-02-normal-company.json
│   ├── payload-03-high-no-company.json
│   ├── payload-04-normal-gmail.json
│   ├── payload-05-high-domain-infer.json
│   ├── payload-06-normal-enterprise.json
│   ├── payload-07-spam-keywords.json       ← triggers dead-letter (spam)
│   ├── payload-08-invalid-email.json       ← triggers dead-letter (invalid email)
│   ├── payload-09-idempotency-duplicate.json ← send 3x to prove idempotency
│   └── payload-10-normal-startup.json
├── docs/
│   └── SETUP.md
└── README.md
```

---

## 🔧 Prerequisites

- **n8n** v1.30+ (self-hosted or cloud)
- **Google account** with Sheets + Gmail API enabled
- **Slack** workspace with a bot token
- **Trello** account with API key + token

---

## ⚙️ Setup Instructions

### Step 1 — Import the Workflow

1. Open n8n → **Workflows** → **Import from File**
2. Select `workflow/lead-to-support-automation.json`
3. Click **Import**

---

### Step 2 — Configure Credentials

Go to **Settings → Credentials** and create the following:

#### Google Sheets / Gmail (OAuth2)
1. Create a Google Cloud project
2. Enable **Google Sheets API** and **Gmail API**
3. Create OAuth2 credentials (Desktop App)
4. In n8n: **New Credential → Google OAuth2**
5. Enter Client ID and Client Secret → Authorize

#### Slack
1. Create a Slack App at https://api.slack.com/apps
2. Add Bot Token Scopes: `chat:write`, `channels:read`
3. Install app to your workspace
4. Copy **Bot User OAuth Token** → n8n: **New Credential → Slack OAuth2 API**

#### Trello
1. Get API Key from https://trello.com/app-key
2. Generate Token on same page
3. In n8n: **New Credential → Trello API**

---

### Step 3 — Configure Environment Variables

Replace these placeholders in the workflow nodes (or use n8n environment variables):

| Placeholder | Description | Where to find it |
|---|---|---|
| `YOUR_GOOGLE_SHEET_ID` | Google Sheet ID from URL | `docs.google.com/spreadsheets/d/SHEET_ID/` |
| `YOUR_SLACK_CHANNEL_ID` | Slack channel ID (not name) | Right-click channel → Copy Link → last segment |
| `YOUR_TRELLO_BOARD_ID` | Trello board ID | `trello.com/b/BOARD_ID/` |
| `YOUR_TRELLO_LIST_ID` | ID of the list for new tickets | Use Trello API: `GET /1/boards/{id}/lists` |
| `support@yourdomain.com` | Sender email address | Your verified Gmail |
| `team@yourdomain.com` | Digest recipient email | Your team email |

---

### Step 4 — Set Up Google Sheet

Create a Google Sheet with **two tabs**:

#### Tab 1: `Leads`
Columns (exact names):
```
IdempotencyKey | Name | Email | Company | CompanySource | Message | Urgency | Product | ReceivedAt | Status
```

#### Tab 2: `DeadLetter`
Columns:
```
Timestamp | FailureReason | RawPayload | ErrorType
```

---

### Step 5 — Activate the Workflow

1. Open the workflow in n8n
2. Click **Activate** (toggle in top-right)
3. Copy the webhook URL from the **"Webhook: Receive Lead"** node
4. Use this URL to POST leads

---

## 🚀 Running the Workflow

### Send a Test Lead

```bash
# Replace YOUR_WEBHOOK_URL with the URL from the Webhook node
curl -X POST YOUR_WEBHOOK_URL \
  -H "Content-Type: application/json" \
  -d @sample-payloads/payload-01-high-urgency-company.json
```

### High Urgency Path
```bash
curl -X POST YOUR_WEBHOOK_URL \
  -H "Content-Type: application/json" \
  -d @sample-payloads/payload-01-high-urgency-company.json
```
Expected: Slack alert + Trello ticket created + record in Google Sheets

### Normal Path
```bash
curl -X POST YOUR_WEBHOOK_URL \
  -H "Content-Type: application/json" \
  -d @sample-payloads/payload-02-normal-company.json
```
Expected: Confirmation email sent to lead + status logged

### Dead Letter (Spam)
```bash
curl -X POST YOUR_WEBHOOK_URL \
  -H "Content-Type: application/json" \
  -d @sample-payloads/payload-07-spam-keywords.json
```
Expected: 422 response + row written to `DeadLetter` sheet

### Dead Letter (Invalid Email)
```bash
curl -X POST YOUR_WEBHOOK_URL \
  -H "Content-Type: application/json" \
  -d @sample-payloads/payload-08-invalid-email.json
```
Expected: 422 response + row written to `DeadLetter` sheet

---

## 🔁 Idempotency Demonstration

Send payload-09 **three times** in a row:

```bash
for i in 1 2 3; do
  curl -X POST YOUR_WEBHOOK_URL \
    -H "Content-Type: application/json" \
    -d @sample-payloads/payload-09-idempotency-duplicate.json
  echo "Sent attempt $i"
  sleep 1
done
```

Open the Google Sheet `Leads` tab — you will see **only one row** for this lead.

**How it works:** The idempotency key is `base64(email + "|" + first50charsOfMessage)`. The Google Sheets node uses `appendOrUpdate` with `matchingColumns: ["IdempotencyKey"]`, which updates the existing row instead of adding a duplicate.

---

## 📊 Daily Digest

The workflow has a **Schedule Trigger** set to **6:00 PM daily**.

It will:
1. Read all rows from the `Leads` sheet
2. Filter to today's leads
3. Count by urgency and product
4. List the top 5 most recent leads
5. Send an HTML digest email to `team@yourdomain.com`
6. Post a summary to Slack

To test immediately, manually trigger the `Daily 6PM Trigger` node in n8n.

---

## 🔄 Workflow Architecture

```
[Webhook] → [Validate + Spam Check]
                    ↓
              [Valid?]
             /        \
      [YES]            [NO]
         ↓              ↓
    [Enrich]     [Dead Letter Sheet]
         ↓         + [422 Response]
    [Google Sheets Store]
         ↓
    [High Urgency?]
       /         \
   [YES]          [NO]
     ↓              ↓
[Slack Alert]   [Confirmation Email]
[Trello Ticket] [Log Status]
     ↓              ↓
  [200 OK]       [200 OK]

[Schedule 6PM] → [Read Leads] → [Aggregate] → [Build HTML]
                                                    ↓
                                           [Email + Slack Digest]
```

---

## 🛡️ Reliability Features

| Feature | Implementation |
|---|---|
| **Idempotency** | `appendOrUpdate` with unique idempotency key per lead |
| **Spam detection** | Keyword list + all-caps check + name length check |
| **Email validation** | RFC-compliant regex |
| **Dead letter queue** | Separate `DeadLetter` sheet tab with error reason |
| **Retries** | n8n built-in retry (3 attempts, 5s delay) on node failures |
| **Error logging** | All failed executions saved in n8n + DeadLetter sheet |
| **Response codes** | 200 for success, 422 for validation failures |

---

## 📋 Sample Payload Summary

| # | File | Urgency | Expected Path |
|---|---|---|---|
| 01 | payload-01-high-urgency-company.json | HIGH | Slack + Trello + Sheet |
| 02 | payload-02-normal-company.json | normal | Email + Sheet |
| 03 | payload-03-high-no-company.json | HIGH | Slack + Trello (company inferred from domain) |
| 04 | payload-04-normal-gmail.json | normal | Email (company = "Individual") |
| 05 | payload-05-high-domain-infer.json | HIGH | Slack + Trello (domain inferred) |
| 06 | payload-06-normal-enterprise.json | normal | Email + Sheet |
| 07 | payload-07-spam-keywords.json | - | ❌ Dead Letter (spam keywords) |
| 08 | payload-08-invalid-email.json | - | ❌ Dead Letter (invalid email) |
| 09 | payload-09-idempotency-duplicate.json | HIGH | Only 1 record (idempotency test) |
| 10 | payload-10-normal-startup.json | normal | Email + Sheet (domain inferred) |
