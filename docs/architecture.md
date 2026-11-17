# Architecture Notes

## Workflow overview

The pipeline runs as a scheduled n8n workflow. Each trigger cycle picks up one unprocessed lead row from Google Sheets and runs it through the full pipeline.

## Node breakdown

### 1. Schedule Trigger
- Fires every 1 minute
- No external dependency — just a timer

### 2. Get row(s) in sheet
- Reads from the `BDR` spreadsheet, `Sheet1`
- Filter: `Status` = `[empty]`
- Returns the first matching row (one item per execution)
- Uses Google Service Account credentials (not OAuth2 — the Sheets Trigger node requires OAuth2 and does not support Service Accounts)

### 3. Hunter (domainSearch)
- Takes `Company Name` from the sheet row, lowercases it, appends `.com`
- Expression: `{{ $json["Company Name"].toLowerCase() }}.com`
- Returns up to 10 contacts (free tier limit)
- Picks the first result: name, title, email, LinkedIn, phone

### 4. Message a model (Gemini)
- Model: `models/gemini-2.5-flash-lite`
- Prompt includes company name, contact name, title, and city
- Returns a short personalized cold outreach email body

### 5. Append or update row in sheet
- Matches on `Company Name` column
- Writes back: Contact Name, Title, Email, Pitch, Status, LinkedIn, Phone, FollowUp-Date, TimeStamp
- Follow-up date: `$now.plus({days:2}).toFormat('yyyy-MM-dd')`
- Timestamp: `$now.toFormat('yyyy-MM-dd HH:mm')`
- Status is hardcoded to `Email Sent`

### 6. Send a text message (Telegram)
- Sends a summary of the processed lead to a configured Telegram chat
- Covers: company, contact name, email, pitch preview

## Credential types used in n8n

| Node | Credential type |
|---|---|
| Google Sheets (read + write) | Google Service Account |
| Hunter | Hunter |
| Gemini | Google Gemini(PaLM) Api |
| Telegram | Telegram |

## Why Schedule Trigger instead of Google Sheets Trigger

The Google Sheets Trigger node in n8n uses polling via the Sheets API and requires OAuth2. It does not accept Service Account credentials. Since this project uses a Service Account (simpler to set up without an OAuth consent screen), the Schedule Trigger + filter approach was used instead.

## Known edge cases

- If Hunter returns no results for a domain, the workflow errors out at the Hunter node. Add error handling or a fallback branch if needed.
- Domain resolution (`name.com`) fails for multi-word company names or companies with non-standard domains.
- The workflow processes one row per minute. If multiple rows are added simultaneously, they are processed in subsequent trigger cycles.
