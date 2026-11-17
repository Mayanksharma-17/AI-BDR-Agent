# AI BDR Agent

<video src="https://github.com/user-attachments/assets/5b5e8d3f-5982-430a-a40a-7e79f84085a0" controls width="100%"></video>

An automated Business Development Representative pipeline built on n8n. Given a list of company names in a Google Sheet, the agent finds the top contact at each company via Hunter.io, generates a personalized outreach pitch using Gemini AI, writes the results back to the sheet, and sends a Telegram notification — all without manual intervention.

---

## What it does

1. **Trigger** — Runs on a schedule (every minute by default). Reads all rows from the Google Sheet where `Status` is blank.
2. **Contact discovery** — Queries Hunter.io domain search to find the most relevant contact (name, title, email, LinkedIn, phone) at the company.
3. **Pitch generation** — Sends contact and company context to Gemini AI, which writes a short, personalized cold outreach email.
4. **Sheet update** — Writes Contact Name, Title, Email, Pitch, LinkedIn, Phone, Follow-up Date (T+2 days), Timestamp, and Status = `Email Sent` back to the same row.
5. **Notification** — Sends a summary of the processed lead to a Telegram bot.

---

## Stack

| Component | Purpose |
|---|---|
| n8n (self-hosted) | Workflow orchestration |
| Google Sheets | Lead input and output storage |
| Hunter.io | Contact and email discovery |
| Google Gemini API | AI pitch generation |
| Telegram Bot API | Real-time notifications |

---

## Prerequisites

Before importing the workflow, you need accounts and credentials for all of the following:

- **n8n** — self-hosted, v1.0 or later. [Install guide](https://docs.n8n.io/hosting/)
- **Google Cloud project** with Sheets API and Drive API enabled, and a Service Account with a JSON key downloaded
- **Hunter.io** account — free tier works, limited to 10 results per domain search
- **Google Gemini API key** — available at [aistudio.google.com](https://aistudio.google.com)
- **Telegram bot** — create one via [@BotFather](https://t.me/BotFather), get the token, and retrieve your Chat ID

---

## Setup

### 1. Google Sheet

Create a sheet named `BDR` with `Sheet1` as the tab name. The first row must have these exact column headers in order:

```
Company Name | City | Contact Name | Title | Email | Pitch | Status | LinkedIn | Phone | FollowUp-Date | TimeStamp
```

Share the sheet with your Service Account email (e.g. `your-sa@your-project.iam.gserviceaccount.com`) with Editor access.

To add leads, fill in `Company Name` (and optionally `City`). Leave all other columns blank. The agent fills the rest.

---

### 2. n8n Credentials

In n8n, go to **Settings > Credentials** and create the following:

**Google Sheets (Service Account)**
- Credential type: `Google Service Account`
- Paste the contents of your Service Account JSON key

**Hunter**
- Credential type: `Hunter`
- Enter your Hunter.io API key

**Google Gemini**
- Credential type: `Google Gemini(PaLM) Api`
- Enter your Gemini API key

**Telegram**
- Credential type: `Telegram`
- Enter your bot token

---

### 3. Import the workflow

1. In n8n, click the **+** icon to create a new workflow
2. Click the **...** menu (top right) > **Import from file**
3. Select `workflow/bdr-agent.json` from this repository
4. After import, open each node and re-assign the credentials you created in step 2
5. In the **Get row(s) in sheet** node, confirm the Document is set to `BDR` and Sheet to `Sheet1`
6. In the **Append or update row in sheet** node, confirm Column to match on is set to `Company Name`
7. In the **Send a text message** node, set your Telegram Chat ID
8. Click **Publish**

---

### 4. Run it

Add a row to the sheet with a company name (e.g. `Stripe`). Within one minute the schedule trigger fires, the workflow runs, and the row is populated with contact data, a pitch, and a follow-up date. A Telegram message confirms completion.

---

## File structure

```
AI-BDR-Agent/
├── workflow/
│   └── bdr-agent.json      # n8n workflow export — import this into n8n
├── docs/
│   └── architecture.md     # Flow diagram and node-level notes
└── README.md
```

---

## Google Sheet schema

| Column | Filled by |
|---|---|
| Company Name | You (input) |
| City | You (optional input) |
| Contact Name | Hunter.io |
| Title | Hunter.io |
| Email | Hunter.io |
| Pitch | Gemini AI |
| Status | Workflow (`Email Sent`) |
| LinkedIn | Hunter.io |
| Phone | Hunter.io |
| FollowUp-Date | Workflow (current date + 2 days) |
| TimeStamp | Workflow (run time) |

---

## Known limitations

- **No email sending** — The workflow generates and stores the pitch but does not send emails. This was intentional during development to avoid unintended outreach. Adding a Gmail or SMTP node after the Gemini step is straightforward if needed.
- **Domain resolution is naive** — Company names are lowercased and `.com` is appended (e.g. `Swiggy` → `swiggy.com`). Multi-word names like `Tata Consultancy Services` will not resolve correctly. A preprocessing step or manual domain column would fix this.
- **Hunter.io free tier limits** — Free accounts are capped at 10 results per domain search. LinkedIn and phone fields are often empty on free-tier responses.
- **Service Account does not work with Google Sheets Trigger** — The workflow uses a Schedule Trigger instead. The Google Sheets Trigger node requires OAuth2 and does not support Service Account auth in n8n.
- **One lead per run** — The workflow processes the first unprocessed row per trigger cycle. For bulk processing, the schedule interval can be reduced or the workflow logic extended with a loop.

---

## Architecture

```
Schedule Trigger (every 1 min)
        |
        v
Get rows from Google Sheet
  [Filter: Status = blank]
        |
        v
Hunter.io Domain Search
  [Input: companyname.com]
  [Output: name, title, email, linkedin, phone]
        |
        v
Gemini AI — Generate pitch
  [Input: contact + company context]
  [Output: personalized cold email body]
        |
        v
Append / Update row in Google Sheet
  [Match on: Company Name]
  [Writes: all contact fields + pitch + status + dates]
        |
        v
Telegram — Send notification
  [Summary of processed lead]
```

---

## Author

Mayank Sharma — built as part of a task for GoPratle
