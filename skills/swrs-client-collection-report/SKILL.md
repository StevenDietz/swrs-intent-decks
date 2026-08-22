---
name: swrs-client-collection-report
description: >-
  Produce the full SWRS branded client collection activity report with write-down recommendations,
  hosted cover thumbnail, and delivered via email — for any client whose book spans multiple ADS
  location client keys sharing a single Nutshell account (the "multi-location" pattern, e.g. Oscar
  Car Rental, FRS franchise groups, multi-branch dealers). ALWAYS use when Steven asks to "run a
  report for [client]", "send the weekly report to [client]", "build the write-down report",
  "produce the collection activity report", "send [client] their Monday report", or when a client
  emails asking for a status update or says they missed a scheduled report. Also trigger when Steven
  says "do what we did for Oscar" or asks to set up a recurring report for a multi-location client.
  Pulls live data from all location client keys, merges the Account Summary xlsx, builds a branded
  HTML report (SWRS hero image bleed thumbnail + expandable per-account activity logs), pushes
  everything to GitHub Pages (swrs-intent-decks), and fires the send-email.yml workflow to deliver
  a clickable thumbnail email directly to the client contact.
---

# SWRS Client Collection Activity Report — Multi-Location

This skill codifies the full end-to-end workflow proven with Oscar Car Rental (Aug 22, 2026):
resolve all location client keys → pull data → build branded HTML report → push to GitHub Pages →
send clickable thumbnail email via send-email.yml.

## When to use this skill

- Client has multiple ADS ClientKeys (locations/franchises) under one Nutshell account
- Steven asks for a weekly/monthly status report, write-down report, or collection activity summary
- A client emails asking why they missed their Monday report
- Steven says "do what we did for Oscar" or "same format as Oscar"

---

## Step 1 — Resolve all client keys

```python
swrs_run_report(reportName="ClientLookup", searchName="<client name>")
```

Returns one row per Nutshell ID. Oscar pattern: single NutShellID (e.g. `170412-accounts`) maps to
one ADS ClientKey (e.g. 11181) as the *master* record, but the actual placed accounts live under
**location-level ClientKeys** (11181–11233 for Oscar). To find all of them:

1. Pull `swrs_account_summary(ClientKey=<NutShellID numeric part>, PageSize=500)` — this returns
   ALL accounts across all location keys linked to that Nutshell account in one call.
2. Collect the unique `CLIENT_KEY` values from the results — these are the location-level ADS keys.
3. Run `swrs_run_report(reportName="ClientCollectionLog", clientKey=<each location key>)` for every
   unique CLIENT_KEY found. Handle intermittent Cloudflare 403s with exponential backoff (3–5s
   between retries, up to 6 attempts). Cache each result to disk (`log_{key}.json`).

**Critical ID distinction:** `swrs_account_summary` takes the **Nutshell numeric ID** (e.g. 170412),
NOT the ADS ClientKey. Every other SWR-API tool takes the ADS ClientKey. Wrong ID = silent zero rows.

---

## Step 2 — Reconcile with the Account Summary xlsx

The client-provided or SSRS-exported "Account Summary by Manager" xlsx is the financial source of
truth. Parse with openpyxl (`data_only=True`):

- **Header row**: row index 5 (0-based), i.e. `min_row=6` in openpyxl
- **Columns**: [0] Client/Location, [2] Account#, [3] First Name, [6] Last Name, [7] Orig Debt,
  [8] Current Debt, [12] Balance, [13] Type/Status, [14] Most Current Comment
- **Subtotal rows**: acct field is blank — skip them
- **Sum of Payments**: stored as negative — take absolute value for "paid" amounts
- Strip all ADS fixed-width space padding (`str(val).strip()`)

Match xlsx rows to API rows by account# + balance proximity. Unmatched xlsx rows (e.g. corporate
placements not yet linked in API) = "pending" group; include them in the report with xlsx data only.

---

## Step 3 — Build the merged dataset

For each account combine:
- **Identity**: last/first name, acct#, location/client name, AGYID, DEBT_KEY
- **Financials**: orig placed, current debt, balance, payments (default $0 if none)
- **Dates**: DOR, DOA from API
- **Status**: prefer live log status over xlsx type; apply client-facing masking (see Step 4)
- **Activity counts**: from ClientCollectionLog Notes — classify each note:
  - `letters`: "letter series sent letter #"
  - `emails`: "workflow email sent", "email sent", "con early", "con late", "email bal @ cbr",
    "email in collections", "ta sent email"
  - `validation`: "validation email sent", "validation message sent"
  - `sms`: "ta sent sms"
  - `calls`: "skit outgoing call", "tcn outbound call", "left message"
  - `skiptrace`: "ln phone number added", "ln address updated", "accurint"
  - `cbr`: "pulled cbr", "pulled manual cbr", "email bal @ cbr"
- **Full chronological log**: all notes sorted by Note_DateTime (oldest → newest)
- **Numbers on file**: from DialAttempts (number, action type, dial count)

---

## Step 4 — Client-facing status masking (REQUIRED)

**Never disclose "Restricted" status to clients.** Replace everywhere it appears:

| Internal Status | Client-Facing Label | Badge Color |
|---|---|---|
| Active | Active | Green |
| New Business | New Business | Green |
| Follow Up | Follow Up | Amber |
| Cease and Desist | Cease and Desist | Red |
| Client Remove / Remove | Client Remove | Red |
| **Restricted** | **Under Review** | **Amber** |
| Letter Hold | Under Review | Amber |

Apply this mask to: status badges, portfolio status table, note text (scrub the word "Restricted"
from any note lines), and the write-down section. Verify with `assert "Restricted" not in html_out`.

**Write-down section**: only include accounts whose effective client-facing status is in:
`{Cease and Desist, Client Remove, Remove, Cancel, Inactive, Bankruptcy, Deceased, Fraud Accnt,
Dispute, Skips}`. Do NOT include Restricted/Under Review in the client-facing write-down table.

---

## Step 5 — Build the branded HTML report

### Design spec (SWRS brand)
- **Font**: Fira Sans (Google Fonts CDN), weights 300/400/700
- **Palette**: Navy `#070B97` · Blue `#0446F1` · Sky `#0495F1` · Light `#82CCFA` · Deep `#070731` · Slate `#3D3D52`
- **Logo**: `/mnt/skills/user/swrs-brand/assets/logos/swrs-logo-color.png` (embed as base64)

### Report structure
1. **Hero banner**: gradient `#070731 → #070B97 → #0446F1`, logo top-left, report title, client name + date
2. **KPI strip**: accounts placed, $ placed, $ balance, $ collected, total activity count — floating cards
3. **Executive Summary**: prose paragraph covering total actions by type, liquidation status, contact quality assessment, any legal-escalation flags (high-balance accounts, asset search results)
4. **Portfolio by Status table**: status | count | balance (sorted by balance desc)
5. **Write-down section**: table with debtor | acct# | location | balance | status | basis (only client-facing write-down statuses)
6. **Per-account cards grouped by location**: each card shows placed/balance/paid, status badge, activity counts, most recent note, and `<details>` expandable full chronological log with numbers on file
7. **Methodology footnote**: data pull date, account counts, regulatory basis (Reg F, FCRA)
8. **Footer**: SWRS address, phone, email, confidentiality line

Verify no "Restricted" anywhere before writing output file.

---

## Step 6 — Build the 1200×630 cover thumbnail

Use the SWRS website hero image as full bleed background:
- **Hero URL**: `https://e174cb2a.delivery.rocketcdn.me/wp-content/uploads/2023/08/hero-image-1.jpg`
- Download and embed as base64 in the card HTML
- **Overlay**: `linear-gradient(100deg, rgba(7,7,49,0.92) 0%, rgba(7,11,151,0.82) 38%, rgba(4,70,241,0.55) 62%, rgba(4,70,241,0.15) 100%)` — left-heavy so text reads on the photo
- **Logo**: white version (`swrs-logo-white.png`) top-left, embedded base64
- **Content left**: eyebrow "Southwest Recovery Services", h1 report title, client pill badge, CTA button "View Full Report →"
- **Stat cards right**: 3 cards (accounts placed, balance, activity count) with dark semi-transparent background
- **Footer bar**: SWRS contact info left, report URL right

Render pipeline:
```python
from weasyprint import HTML
HTML('card.html').write_pdf('card.pdf')
# pdftoppm -png -r 128 -f 1 -l 1 card.pdf cardpg
# PIL resize to 1200x630, save JPEG quality=88
```
Target: under 200KB. `@page { size: 1200px 630px; margin: 0; }` in the card CSS.

---

## Step 7 — Push to GitHub Pages (swrs-intent-decks)

PAT: `/mnt/project/GITHUBOPEN.txt` — has full repo write access.
Base URL: `https://stevendietz.github.io/swrs-intent-decks/`

Push three files via GitHub Contents API (`PUT /repos/StevenDietz/swrs-intent-decks/contents/`):

| Local file | Repo path | Purpose |
|---|---|---|
| `report.html` | `clients/{slug}/index.html` | Full report (live URL) |
| `cover.jpg` | `clients/{slug}/cover.jpg` | Thumbnail (referenced in email) |
| `email-card.html` | `clients/{slug}/email-card.html` | Copy-page for Outlook paste |

`{slug}` = kebab-case client name (e.g. `oscar-car-rental`).

For each file: GET existing sha first (404 = new file, omit sha from payload). PUT with
`{"message": "...", "content": "<base64>", "sha": "<existing sha if updating>"}`.

**Live URLs after push:**
- Report: `https://stevendietz.github.io/swrs-intent-decks/clients/{slug}/`
- Cover: `https://stevendietz.github.io/swrs-intent-decks/clients/{slug}/cover.jpg`
- Copy page: `https://stevendietz.github.io/swrs-intent-decks/clients/{slug}/email-card.html`

GitHub Pages typically propagates within 1–2 minutes.

---

## Step 8 — Send the email via send-email.yml

Workflow: `StevenDietz/NUTSHELL-AUTOMATIONS` → `.github/workflows/send-email.yml` (ID 305007992)

Dispatch via POST to `/actions/workflows/send-email.yml/dispatches`:
```json
{
  "ref": "main",
  "inputs": {
    "to": "<client contact email>",
    "cc": "<any cc, e.g. secondary contact>",
    "subject": "Re: <thread subject or 'Your Weekly Collection Activity Report'>",
    "body": "<html body — see template below>",
    "sender": "sdietz@swrecovery.com",
    "content_type": "html",
    "attachment": ""
  }
}
```

**Email body template:**
```html
<div style="font-family:'Segoe UI',Arial,sans-serif;color:#1a1a2e;max-width:640px;margin:0 auto">
<p>Hi {contact_first_name},</p>
<p>{opening — e.g. "Please find your current collection activity report below, covering all {n} accounts across your franchise locations."}</p>

<a href="{REPORT_URL}" target="_blank" style="display:block;margin:24px 0;">
  <img src="{COVER_URL}" alt="{CLIENT} Collection Activity Report — Southwest Recovery Services — Click to view"
       width="600" style="display:block;border-radius:8px;border:0;" />
</a>

<p><a href="{REPORT_URL}" style="color:#0446F1;font-weight:bold;">&#128073; View the full report</a></p>

<p>{closing — e.g. "The report covers every account and all collection activity logged to date. Going forward you can expect this every Monday. Please don't hesitate to reach out with any questions."}</p>

<p style="margin-top:32px">Best regards,<br><br>
<strong>Steven Dietz</strong><br>CEO, Southwest Recovery Services<br>
(214) 387-8068 x 310<br>sdietz@swrecovery.com<br>www.swrecovery.com</p>
</div>
```

Poll for completion: GET `/actions/workflows/send-email.yml/runs?event=workflow_dispatch&per_page=1`
every ~22 seconds until `status == "completed"`. Confirm `conclusion == "success"` before reporting done.

**Always send a proof copy to `sdietz@swrecovery.com` first** if Steven requests it, with subject
prefixed "FWD EXAMPLE:". Use the same workflow dispatch, just swap the `to` field.

---

## Step 9 — Log to Notion

Append a session summary to Notion page `3a57e624-60b2-812a-8d2f-db065fca5a28` using
`Notion:notion-update-page` with `command: insert_content`, `position: end`. Include:
- Client name, ADS keys resolved, Nutshell ID
- Portfolio totals: accounts, placed $, balance, collected $, liquidation %
- Activity totals by type
- Write-down recommendations (internal dollar total, all statuses including Restricted)
- Report URL, cover URL
- Send confirmation (to, cc, workflow conclusion)

---

## Key learnings & gotchas

- **NutShellID ≠ ADS ClientKey**: `swrs_account_summary` takes the numeric Nutshell ID; all other
  SWR-API tools take the ADS ClientKey. Sending the wrong one returns silent zero rows.
- **Multi-location pattern**: one Nutshell ID → one master ADS key → many location ADS keys.
  `swrs_account_summary` returns all locations; collect unique `CLIENT_KEY` values from results to
  get the full location key list for `ClientCollectionLog` pulls.
- **403 rate limiting on MCP worker**: handle with backoff (3–5s, up to 6 retries). Cache each
  `log_{key}.json` to disk so a timeout doesn't lose completed pulls.
- **Impala portal-linkage defect**: if `swrs_account_summary` returns zero rows despite a valid
  Nutshell ID, and `swrs_documents` for the master ADS key returns results under a single DEBT_KEY
  with many invoice documents collapsed — this is the known Impala import linkage defect. Flag it
  for IT/Alex; fall back to the xlsx as financial source of truth.
- **ADS space padding**: all string fields are fixed-width padded. Always `.strip()` before use.
- **"Restricted" must never reach the client**: scrub from all output including note text. Verify
  programmatically before pushing to GitHub.
- **Cover JPG must be hosted before the email sends**: GitHub Pages takes ~1–2 min to propagate.
  If sending immediately, Outlook may show a broken image on first open; it resolves on refresh.
  Consider a 90-second delay between push and dispatch, or note this to Steven.
- **send-email.yml sender default**: defaults to `sdietz@swrecovery.com`. Pass `sender` explicitly
  to be safe. The `attachment` input takes a repo-relative path — file must be committed to the
  repo first if used.
- **GitHub PAT location**: `/mnt/project/GITHUBOPEN.txt` — full repo write access. Never echo the value.
