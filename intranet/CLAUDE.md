# Viking Digital Group — Project Context for Claude

## What this repo is

Static HTML website for **Viking Digital Group** (vikingdigitalgroup.com), a digital asset acquisition company focused on acquiring and scaling YouTube channels and e-commerce brands. Hosted on **Netlify**, deployed automatically from this GitHub repo.

**Owner:** Jonathan Martin — Founder & Managing Partner
**Email:** jonathan@vikingdigitalgroup.com
**Site:** vikingdigitalgroup.com

---

## Repo structure

```
/
├── index.html                  ← Main public website
├── intranet/
│   ├── index.html              ← Intranet login page (Netlify Identity)
│   ├── tracker.html            ← Investor outreach tracker app
│   ├── investors.json          ← Master investor data (source of truth)
│   ├── _redirects              ← Netlify routing rules
│   └── SETUP.md                ← Intranet setup guide
└── CLAUDE.md                   ← This file
```

---

## Brand & design standards

- **Primary color:** Navy `#1F3864` / Dark navy `#0D1117`
- **Accent:** Gold `#C9A84C` / Light gold `#E8C97A`
- **Fonts:** Syne (headings), DM Sans (body), DM Mono (labels/code)
- **Tone:** Institutional, confident, minimal — never casual or startup-y
- **URL:** Always `vikingdigitalgroup.com` — never `www.vikingdigitalgroup.com`
- **Year:** Always use current year (2026), never hardcode 2025

---

## Intranet — investor tracker

The `/intranet/` folder is a protected internal tool for tracking investor outreach on [askforfunding.com](https://askforfunding.com).

### How it works

- **`investors.json`** is the single source of truth for all investor profiles and message copy
- **`tracker.html`** loads `investors.json` on startup, merges with localStorage overrides (status, notes, variant chosen)
- **Netlify Identity** handles email/password authentication — registration is invite-only
- **Google Sheets sync** is optional — configure `SHEET_CSV_URL` in `tracker.html`

### investors.json schema

```json
{
  "lastUpdated": "YYYY-MM-DD",
  "investors": [
    {
      "id": "unique-slug",
      "name": "Investor or Fund Name",
      "url": "https://askforfunding.com/investor/...",
      "focus": "E-Commerce / AI / etc.",
      "location": "City, Province/State",
      "day": 1,
      "dayLabel": "13 May 2026",
      "status": "pending",
      "dateSent": "",
      "notes": "Contact: Name (Title). Key info.",
      "variant": "A",
      "recommend": "A",
      "variants": {
        "A": {
          "subject": "Email subject line",
          "body": "Full email body text"
        },
        "B": {
          "subject": "Alternative subject line",
          "body": "Alternative email body text"
        }
      }
    }
  ]
}
```

### Status values
- `pending` — not yet contacted
- `sent` — outreach sent
- `replied` — investor replied
- `pass` — investor passed

### Message style guide (for writing investor variants)

All outreach emails follow these rules:
- **Signed:** Jonathan Martin, Founder & Managing Partner, Viking Digital Group
- **Email:** jonathan@vikingdigitalgroup.com
- **Website:** vikingdigitalgroup.com (no www)
- **Key numbers to reference:** $250K–$300K raise | $192K+/yr net profit | 85–90% margins | 35–40% IRR | ~2.3yr payback | 4–5x exit at 36 months
- **Variant A:** ROI-forward — lead with numbers and metrics
- **Variant B:** Curiosity hook — lead with a contrarian or personalized angle, soft CTA
- **Personalization:** Reference the investor's specific thesis, portfolio companies, or background
- **Length:** Short — 4–6 short paragraphs max, no fluff
- **Closing:** Always end with a soft CTA ("Happy to connect", "Would love 15 minutes", etc.)

---

## Common tasks Claude should know how to do

### Add new investors to investors.json

When asked to add investors (usually with an askforfunding.com profile URL):

1. Fetch the profile page and extract: name, focus/industries, location, contact name, bio/description
2. Assign the next available `day` number (increment from the highest existing day)
3. Set `dayLabel` to the appropriate date
4. Write two personalized email variants (A: ROI-forward, B: curiosity hook) based on their background
5. Set `recommend` to whichever variant better matches their profile
6. Set `status: "pending"` and leave `dateSent` empty
7. Update `lastUpdated` to today's date
8. Commit the updated `investors.json` with message: `feat: add [investor names] to tracker (Day X)`

### Update investor status

If asked to mark investors as sent/replied/passed, update the `status` and `dateSent` fields in `investors.json` accordingly.

### Update message copy

If asked to revise email variants, update the relevant `variants.A` or `variants.B` objects in `investors.json`.

### Netlify Identity users

Do NOT add/remove users via code — direct Jonathan to the Netlify dashboard → Identity tab to manage users.

---

## Deployment

- **Platform:** Netlify (auto-deploy on push to main)
- **Domain:** vikingdigitalgroup.com
- **Build:** No build step — pure static HTML
- **Deploy time:** ~20–30 seconds after push

### Never do

- Push directly to `main` without a PR (unless explicitly told to)
- Modify `netlify.toml` or environment variables without asking
- Change Netlify Identity configuration via code
- Add npm/node dependencies (site must stay dependency-free)
- Break the static-only constraint — no server-side code

---

## Working with Jonathan

- Jonathan will often paste askforfunding.com profile URLs directly in GitHub issues
- He may say "add these investors" or "mark X as sent" — interpret naturally
- Always generate a PR (not a direct push) so he can review before it goes live
- Keep commit messages clean: `feat:`, `fix:`, `update:` prefixes
- When generating email copy, match the tone of existing variants in investors.json — professional, direct, personalized
