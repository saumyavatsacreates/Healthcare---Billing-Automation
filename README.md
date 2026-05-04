

# 🏥 Healthcare Billing AI — Automated Claim Scoring Workflow

An n8n workflow that automatically ingests healthcare billing documents from Google Drive, extracts and validates all billable fields using a three-stage AI pipeline, scores each claim on a 100-point composite scale, and routes it to the appropriate action — with full audit logging and running insights written to Google Sheets.

---

## What It Does

When a new billing document lands in a monitored Google Drive folder, this workflow:

1. Converts it to plain text via Google Docs export
2. Validates that readable text was extracted
3. Runs three sequential AI calls to extract billable items, check coding compliance, and assess denial risk
4. Applies seven deterministic rule-based compliance checks
5. Computes a composite score and assigns a tier
6. Routes the claim to one of four outcomes: Auto Submit, Human Review, Hold & Correct, or Reject
7. Stores all structured data to a Google Sheet and sends a tailored email notification
8. Updates running insights (total claims, avg score, collection rate, top denial reasons)

---

## Pipeline Overview

```
Google Drive (Bills folder)
    ↓
Download File
    ↓
Copy → Google Doc  →  Export as Plain Text
    ↓
Extract & Validate Text
    ↓ (invalid → Log to AuditLog sheet)
AI Stage 1: Extract Billable Items       (ICD-10, CPT, line items, totals)
    ↓
AI Stage 2: Code Mapping & Compliance    (unbundling, upcoding, duplicates)
    ↓
Validation Layer                         (7 rule-based checks → PASS/REVIEW/FAIL)
    ↓
AI Stage 3: Denial Risk Assessment       (denial probability, doc gaps, appeal strength)
    ↓
Scoring Engine                           (0–100 composite score)
    ↓  (parallel)
Insights Generator → Write Insights to Sheet
    ↓
Decision Node (Switch)
    ├── Auto Submit   → Store + Notify: Clean Claim ✅
    ├── Human Review  → Store + Notify: Needs Review ⚠️
    ├── Hold & Correct→ Store + Notify: High Risk 🚨
    └── Reject        → Store only
```

### Scoring Breakdown

| Component | Weight | Source |
|---|---|---|
| Code Accuracy | 30 pts | AI Stage 2 accuracy score |
| Validation | 25 pts | Rule-based validation score |
| Denial Risk | 25 pts | 1 − denial probability |
| Documentation | 20 pts | Doc gaps count |
| **Total** | **100 pts** | |

### Tier Thresholds

| Score | Tier | Action |
|---|---|---|
| ≥ 80 | Clean Claim | Auto Submit |
| 60–79 | Review Required | Human Review |
| 40–59 | High Risk | Hold & Correct |
| < 40 | Reject | Reject |

---

## Prerequisites

- **n8n** (self-hosted or cloud) — tested on v1.121.3
- **Google account** with:
  - Google Drive (Bills folder created)
  - Google Sheets (with `Billing`, `Insights`, and `AuditLog` tabs)
  - Gmail
- **Gemini API key** — [Get one at Google AI Studio](https://aistudio.google.com)
  > ⚠️ If you hit free-tier limits, see [Switching to Groq](#switching-to-groq) below

---

## Setup

### 1. Import the Workflow

In n8n: **Workflows → Import from File** → select `Healthcare_Billing_AI_Scored_Workflow.json`

### 2. Create Your Google Sheet

Create a Google Sheet with three tabs:

**`Billing` tab** — add these headers in row 1:
```
Timestamp | File ID | File Name | Patient ID | Visit Date | Provider | Facility Type | Insurance Type | Total Billed | Est Reimbursement | Final Score | Tier | Action | Score Breakdown | Validation Status | Validation Flags | Denial Probability | Denial Reasons | Appeal Strength | Doc Gaps | Recommended Actions | Compliance Flags
```

**`Insights` tab** — add these headers:
```
Timestamp | Total Claims | Total Billed | Total Est | Collection Rate | Avg Score | Tier Counts | Top Denials
```

**`AuditLog` tab** — add these headers:
```
Timestamp | File ID | File Name | Status | Score | Notes
```

### 3. Configure Credentials

In n8n **Settings → Credentials**, create:

| Credential Name | Type | Used By |
|---|---|---|
| `Google Drive account` | Google Drive OAuth2 | Drive trigger, file nodes |
| `Google Sheets account` | Google Sheets OAuth2 | All sheet nodes |
| `Gmail account` | Gmail OAuth2 | All email nodes |

### 4. Set Your Google Sheet ID

In each Google Sheets node, update the **Document ID** to your Sheet's ID.  
You can find it in the URL: `https://docs.google.com/spreadsheets/d/YOUR_SHEET_ID/edit`

### 5. Set Your Gemini API Key

In each of the three AI nodes (`AI: Extract Billable Items`, `AI: Code Mapping & Compliance`, `AI: Denial Risk`), update the query parameter:
```
key = YOUR_GEMINI_API_KEY
```

### 6. Set the Bills Folder

In the `New Billing Case` trigger node, update the **Folder to Watch** to your Google Drive folder ID.

### 7. Set Notification Email

In each Gmail notify node, update the `sendTo` field to your email address.

### 8. Activate

Toggle the workflow **Active** in n8n.

---

## Switching to Groq

If Gemini free-tier quota is exhausted, replace the Gemini nodes with Groq:

**URL:** `https://api.groq.com/openai/v1/chat/completions`

**Headers:**
```
Content-Type: application/json
Authorization: Bearer YOUR_GROQ_API_KEY
```

**Body format:**
```json
{
  "model": "llama-3.3-70b-versatile",
  "messages": [{ "role": "user", "content": "YOUR_PROMPT" }],
  "temperature": 0.1,
  "max_tokens": 1500
}
```

**Response path changes** from:
```js
candidates[0].content.parts[0].text
```
to:
```js
choices[0].message.content
```

Get a free Groq key at [console.groq.com](https://console.groq.com). Free tier: 14,400 requests/day.

---

## Nodes Reference

| Node | Type | Purpose |
|---|---|---|
| New Billing Case | Google Drive Trigger | Polls Bills folder every minute for new files |
| Download File | Google Drive | Downloads binary file |
| Copy Bill → Google Doc | HTTP Request | Clones file as Google Doc for text extraction |
| Export Bill as Text | HTTP Request | Exports Doc as plain text |
| Extract & Validate Text | Code | Validates text length; captures file metadata |
| IF: Text Valid? | IF | Routes valid/invalid text |
| Log Invalid Bill | Google Sheets | Logs unreadable files to AuditLog |
| AI: Extract Billable Items | HTTP Request | Gemini/Groq — Stage 1 extraction |
| Parse: Billable Items | Code | Parses Stage 1 JSON response |
| AI: Code Mapping & Compliance | HTTP Request | Gemini/Groq — Stage 2 compliance |
| Parse: Code Compliance | Code | Parses Stage 2 JSON response |
| Validation Layer | Code | 7 deterministic compliance rules |
| AI: Denial Risk | HTTP Request | Gemini/Groq — Stage 3 denial risk |
| Parse: Denial Risk | Code | Parses Stage 3 JSON response |
| Scoring Engine | Code | Computes composite 0–100 score |
| Insights Generator | Code | Updates running aggregates in static data |
| Decision Node | Switch | Routes by action tag |
| Store Billing Data | Google Sheets | Writes all fields to Billing tab |
| Write Insights to Sheet | Google Sheets | Appends insights snapshot |
| Notify: Clean Claim | Gmail | Email for auto-submit claims |
| Notify: Needs Review | Gmail | Email for review-required claims |
| Notify: High Risk | Gmail | Email for high-risk / hold claims |

---

## Known Limitations

- **No OCR** — scanned image PDFs will fail at text extraction. Workaround: use Google Docs manually to OCR before upload, or add an OCR API node.
- **Generic compliance rules** — validation rules apply CMS general guidelines, not payer-specific policies.
- **No feedback loop** — denied claims are not fed back to retrain the scoring model.
- **Static scoring weights** — the 30/25/25/20 split is fixed and not calibrated to actual outcomes.
- **Google Sheets scale** — beyond ~5,000 rows performance degrades. Migrate to BigQuery for production use.

---

## Customising the Scoring Weights

In the `Scoring Engine` node, adjust the weights to match your payer mix:

```js
const codeScore    = Math.round((d.codeAccuracyScore / 100) * 30);  // change 30
const valScore     = Math.round((d.validationScore / 100) * 25);     // change 25
const denialScore  = Math.round((1 - d.denialProbability) * 25);     // change 25
const docScore     = Math.max(0, 20 - (docGaps * 4));                // change 20
```

Weights must sum to 100.

---

## License

MIT — free to use, modify, and deploy. Attribution appreciated.
