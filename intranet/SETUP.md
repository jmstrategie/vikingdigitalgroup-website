# VDG Intranet — Setup Guide

## File structure to add to your repo

```
your-repo/
├── intranet/
│   ├── index.html          ← Login page
│   ├── tracker.html        ← Main tracker app
│   ├── investors.json      ← Master investor data
│   ├── _redirects          ← Netlify routing
│   └── SETUP.md            ← This file
```

---

## Step 1 — Enable Netlify Identity

1. Go to your **Netlify dashboard** → your site → **Integrations** tab
2. Find **Netlify Identity** → click **Enable Identity**
3. Under **Registration preferences** → set to **Invite only**
4. Under **External providers** → leave off (email only)

---

## Step 2 — Add users

1. In Netlify Identity → **Invite users**
2. Enter the email addresses of anyone who needs access (e.g. jonathan@vikingdigitalgroup.com)
3. They'll receive an email to set their password

---

## Step 3 — Deploy

Commit and push the `intranet/` folder to your GitHub repo.
Netlify will auto-deploy. Your intranet will be live at:

```
https://vikingdigitalgroup.com/intranet/
```

---

## Step 4 — Update investor data

### Option A: Via this Claude chat (recommended for daily adds)
Drop new profile URLs in the chat. Claude scrapes, writes personalized messages,
and generates an updated `investors.json`. You commit it and Netlify redeploys.

### Option B: Via Google Sheets sync

1. Create a Google Sheet with these columns (row 1 = headers):
   ```
   id | name | url | focus | location | status | dateSent | day | notes | variant
   ```
2. **File → Share → Publish to web** → choose CSV format → copy the URL
3. Open `intranet/tracker.html` and paste the URL into:
   ```js
   const SHEET_CSV_URL = 'YOUR_URL_HERE';
   ```
4. Commit the change. Users can then click **↻ Sync Sheet** in the app.

> Note: Status/notes/variant changes made IN the app are saved to the browser's
> localStorage and persist across sessions. The Sheet sync merges Sheet values on top.

---

## How data works

- `investors.json` = master source of truth (profiles, messages, variants)
- `localStorage` = user overrides (status, notes, date sent, active variant)
- On load: app fetches `investors.json`, then applies localStorage overrides
- On Sheet sync: Sheet values are merged into localStorage overrides

This means you can update messages/profiles by committing a new `investors.json`
without losing any status updates users have made in the app.

---

## Daily workflow

1. Paste new investor profile URLs in Claude chat
2. Claude scrapes + writes personalized messages → generates new `investors.json`
3. Commit `investors.json` → Netlify redeploys in ~30 seconds
4. Open `vikingdigitalgroup.com/intranet/` → new investors appear automatically
