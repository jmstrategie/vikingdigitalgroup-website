# VDG Investor Reply Automation — Setup Guide

Automatically detects investor replies from askforfunding.com, classifies
their intent with Claude AI, and updates the tracker dashboard.

---

## How it works

```
Zoho Mail (askforfunding reply notification)
    ↓
Zapier — trigger on new email
    ↓
Zapier — HTTP POST to Claude API
    ↓  returns: { investor_id, intent, aiSummary, suggestedAction }
Zapier — update Google Sheet row
    ↓
Tracker — click "Sync Sheet" to pull latest data
```

---

## Part 1 — Google Sheet setup

### 1.1 Create the sheet

Create a new Google Sheet named **"VDG Investor Tracker"**.

Row 1 must be these exact headers (copy/paste):

| A   | B    | C      | D        | E           | F      | G           | H         | I               | J     | K       |
|-----|------|--------|----------|-------------|--------|-------------|-----------|-----------------|-------|---------|
| id  | name | status | dateSent | dateReplied | intent | intentLabel | aiSummary | suggestedAction | notes | variant |

### 1.2 Seed with current investors

Paste this data starting at row 2 (one investor per row).
The `id` column must match the slugs in `investors.json` exactly.

Current investor IDs (copy into column A):
```
lukasz-kaiser
s-ventures
josh-woodward
william-mougayar
vitality-capital
interaction-ventures
mu-ventures
onwave-ventures
ballas-capital
northvest-capital
todd-dunlop
chris-mikulin
tenor-investments
```

Fill column B with their display names. Leave C–K blank for now
(Zapier will write to them; you can also fill C/D manually for already-sent investors).

### 1.3 Publish the sheet

1. **File → Share → Publish to web**
2. Select the sheet tab → **CSV** format
3. Click **Publish** → copy the URL
4. Open `intranet/tracker.html`, find this line near the top of the `<script>`:
   ```js
   const SHEET_CSV_URL = '';
   ```
   Paste your URL between the quotes. Commit and push.

---

## Part 2 — Zapier setup

Go to **zapier.com** → **Create Zap**.

---

### Step 1 — Trigger: New Email in Zoho Mail

- **App:** Zoho Mail
- **Event:** New Email
- Connect your Zoho account when prompted.
- **Filter setup** (in the trigger or as a Filter step):
  - `From Email` contains `askforfunding.com`
  - This ensures only reply notifications fire the zap.

> **Tip:** Send yourself a test message through askforfunding first so Zapier
> can find a sample email to work with.

---

### Step 2 — Code (JavaScript) — Build the Claude payload

Add a **Code by Zapier** step (JavaScript):

**Input data:**
- `subject` → from Step 1: Email Subject
- `body` → from Step 1: Email Body (plain text)

**Code:**
```javascript
// Investor list — keep this in sync with investors.json
const INVESTORS = [
  { id: "lukasz-kaiser",         name: "Lukasz Kaiser" },
  { id: "s-ventures",            name: "S Ventures" },
  { id: "josh-woodward",         name: "Josh Woodward" },
  { id: "william-mougayar",      name: "William Mougayar" },
  { id: "vitality-capital",      name: "Vitality Capital / Gershon Hurwen" },
  { id: "interaction-ventures",  name: "Interaction Ventures / Frederic Thouin" },
  { id: "mu-ventures",           name: "mu ventures / Gary Benerofe" },
  { id: "onwave-ventures",       name: "OnWave Ventures / Vlad Ilchuk" },
  { id: "ballas-capital",        name: "Ballas Capital / Mike Ballas" },
  { id: "northvest-capital",     name: "Northvest Capital / Jake Malczewski" },
  { id: "todd-dunlop",           name: "Todd Dunlop" },
  { id: "chris-mikulin",         name: "Chris Mikulin" },
  { id: "tenor-investments",     name: "Tenor Investments / Hugh Dobbie" },
];

const investorList = INVESTORS.map(i => `- ${i.name} (id: ${i.id})`).join('\n');

const prompt = `You are analyzing an email notification from askforfunding.com.
An investor has replied to a funding outreach message sent by Jonathan Martin at Viking Digital Group.

## Investor list
${investorList}

## Email received
Subject: ${inputData.subject}
Body:
${inputData.body}

## Your task
1. Identify WHICH investor sent this reply by matching their name (or company name) to the list above.
2. Classify their intent based on the reply content.
3. Write a one-sentence summary of what they said.
4. Suggest the best next action for Jonathan.

## Intent values (use exactly one):
- "call"       — they want to schedule a call or meeting
- "interested" — positive response, want more information
- "followup"   — non-committal, worth following up in a few days
- "info"       — asking specific questions about the deal
- "pass"       — not interested, declining

## Response format
Return ONLY valid JSON, no other text:
{
  "investor_id": "the-id-from-the-list",
  "investor_name": "Display name",
  "intent": "call|interested|followup|info|pass",
  "intent_label": "Short label (max 4 words, e.g. 'Wants a call')",
  "ai_summary": "One sentence describing what they said and their tone.",
  "suggested_action": "Specific next step for Jonathan.",
  "confidence": 0.0
}`;

return { prompt };
```

---

### Step 3 — HTTP POST to Claude API

Add a **Webhooks by Zapier** step:

- **Action:** POST
- **URL:** `https://api.anthropic.com/v1/messages`
- **Payload type:** JSON
- **Data:**
  ```json
  {
    "model": "claude-haiku-4-5",
    "max_tokens": 400,
    "messages": [
      {
        "role": "user",
        "content": "«prompt from Step 2»"
      }
    ]
  }
  ```
  *(Map `content` to the `prompt` output from the Code step)*

- **Headers:**
  ```
  x-api-key:         YOUR_ANTHROPIC_API_KEY
  anthropic-version: 2023-06-01
  content-type:      application/json
  ```

---

### Step 4 — Code (JavaScript) — Parse Claude's response

Add another **Code by Zapier** step:

**Input data:**
- `response` → from Step 3: Body (the full HTTP response text)

**Code:**
```javascript
try {
  const body    = JSON.parse(inputData.response);
  const text    = body.content[0].text.trim();
  // Strip markdown code fences if present
  const cleaned = text.replace(/^```json?\s*/,'').replace(/```\s*$/,'');
  const result  = JSON.parse(cleaned);
  return {
    investor_id:      result.investor_id      || '',
    investor_name:    result.investor_name    || '',
    intent:           result.intent           || '',
    intent_label:     result.intent_label     || '',
    ai_summary:       result.ai_summary       || '',
    suggested_action: result.suggested_action || '',
  };
} catch(e) {
  return {
    investor_id: '', investor_name: '', intent: 'followup',
    intent_label: 'Review needed', ai_summary: 'Could not parse AI response.',
    suggested_action: 'Review email manually.',
  };
}
```

---

### Step 5 — Google Sheets: Update Row

Add a **Google Sheets** step:

- **Action:** Update Spreadsheet Row
- **Spreadsheet:** VDG Investor Tracker
- **Lookup column:** A (id)
- **Lookup value:** `investor_id` from Step 4
- **Row values to update:**
  - Column C (status): `replied`
  - Column E (dateReplied): today's date — use Zapier's `{{zap_meta_human_now}}` or a Formatter step
  - Column F (intent): `intent` from Step 4
  - Column G (intentLabel): `intent_label` from Step 4
  - Column H (aiSummary): `ai_summary` from Step 4
  - Column I (suggestedAction): `suggested_action` from Step 4

---

### Step 6 (optional) — Email: Draft reply notification

Add a **Zoho Mail** or **Gmail** step:

- **Action:** Send Email
- **To:** jonathan@vikingdigitalgroup.com
- **Subject:** `↩ Reply from «investor_name» — «intent_label»`
- **Body:**
  ```
  Investor: «investor_name»
  Intent:   «intent_label»

  Summary:
  «ai_summary»

  Suggested next action:
  «suggested_action»

  → Open tracker: https://vikingdigitalgroup.com/intranet/
  ```

---

## Part 3 — Wire the tracker to the Sheet

Once the Sheet URL is set in `tracker.html` (Step 1.3 above):

1. Open `vikingdigitalgroup.com/intranet/`
2. Click **↻ Sync Sheet**
3. The intent badges, AI summaries, and reply dates will appear instantly

New intent badges will show:
- 📞 **Wants a call** — gold badge, top priority
- ✦ **Interested** — green badge
- ↩ **Follow up** — blue badge
- ? **Wants info** — purple badge
- — **Not interested** — muted badge

Warm Leads stat (top of tracker) counts `call` + `interested` replies automatically.

---

## Part 4 — Costs

| Service | Cost |
|---|---|
| Zapier Free | 100 tasks/month — enough for this volume |
| Anthropic API | ~$0.001 per email (Haiku model) — $5 lasts ~5,000 replies |
| Google Sheets | Free |

Total expected cost: **under $1/month** at current outreach volume.

---

## Troubleshooting

**Zap fires but wrong investor matched**
→ Check that the investor name/company appears in the askforfunding email body.
   Update the INVESTORS array in Step 2 to add alternate names if needed.

**Sheet not updating**
→ Make sure column A in your Sheet exactly matches the `id` slugs in `investors.json`.

**Tracker not showing AI data after sync**
→ Confirm `SHEET_CSV_URL` is set in `tracker.html` and the Sheet is published as CSV.

**Claude returns invalid JSON**
→ The Step 4 parser handles this gracefully — check the `investor_id` field is populated.
