# Job-Match-Weekly-Search

An n8n automation that reads your CV, extracts your skills using AI, searches live remote job listings every week, scores each job against your profile, and emails you the top matches — fully automated, no manual steps once set up.

---

## What it does

The project is made up of **two separate n8n workflows** that share a single Google Sheet as their common data store.

### 1. CV Profile Update
Triggered by a public form. Submit your name, email, and CV (PDF) — the workflow extracts the text, asks an AI model to pull out relevant job-search keywords, and saves your profile (name, email, keywords, last updated) to a Google Sheet.

```
Form Trigger → Extract from File → Basic LLM Chain (Groq) → Edit Fields → Google Sheets (write to Profile tab)
```

### 2. Weekly Job Search
Runs automatically every **Sunday at 8:00 AM**. Reads your latest profile, pulls live job listings from RemoteOK, filters them by your keywords, checks each one against a log of previously-seen jobs (so nothing is ever sent twice), scores the new ones with AI against your profile, and emails you the top 3 matches scoring 70+ — or a friendly "nothing this week" message if none qualify.

```
Schedule Trigger (Sun 8am)
→ Read Profile (Google Sheets)
→ HTTP Request (RemoteOK API)
→ Code: filter jobs by keywords
→ Loop over each job
    → Dedup check (Google Sheets: JobLog)
    → IF already processed → skip, loop continues
    → IF new → Wait → AI scoring (Groq) → Edit Fields → Append to JobLog → loop continues
→ Filter (score ≥ 70 and not null)
→ IF has data?
    → TRUE  → Sort → Limit (top 3) → Aggregate → Edit Fields (build email) → Gmail
    → FALSE → Edit Fields (fallback "no matches" message) → Gmail
```

---

## Project structure

```
job-matcher/
├── README.md
├── cv-profile-update.json      
└── weekly-job-search.json      
```

The two `.json` files are exact exports of the n8n workflows and can be re-imported into any n8n instance (cloud or self-hosted).

---

## How it works — the shared Google Sheet

Both workflows read/write to one spreadsheet, **"Job Match Tracker"**, with two tabs:

**Profile** — one row per person, holding their latest submitted CV data
| name | email | keywords | last_updated |
|------|-------|----------|---------------|

**JobLog** — a running record of every job that's ever been scored, so it's never re-evaluated or re-sent
| job_id | title | company | url | match_score | reason | status | date_checked |
|--------|-------|---------|-----|-------------|--------|--------|----------------|

---

## Setting this up on your own device / account

### Requirements
- An [n8n](https://n8n.io) account (cloud, free tier works) or a self-hosted instance
- A Google account (for Sheets, Gmail, and Google Drive OAuth)
- A free [Groq](https://console.groq.com) API key (used for the AI scoring/keyword-extraction steps — no cost for this workload)

### Step-by-step

1. **Import both workflows**
   - In n8n: `Overview → Add workflow → ... menu → Import from File`
   - Import `workflows/cv-profile-update.json` and `workflows/weekly-job-search.json` separately

2. **Create the Google Sheet**
   - Create a new Google Sheet named anything you like (e.g. "Job Match Tracker")
   - Add two tabs: `Profile` and `JobLog`, with the exact column headers listed above in row 1

3. **Connect credentials in n8n**
   - **Google Sheets**: Add Credential → Google Sheets → Sign in with Google (grant Sheets access)
   - **Gmail**: Add Credential → Gmail → Sign in with Google (grant Gmail send access)
   - **Groq**: Add Credential → OpenAI (Groq uses an OpenAI-compatible API) → set Base URL to `https://api.groq.com/openai/v1` and paste your Groq API key

4. **Point both workflows at your Sheet**
   - In every Google Sheets node in both workflows, set the **Document** to your new sheet and the correct **Sheet** tab (`Profile` or `JobLog`)

5. **Update the AI nodes**
   - In each **Basic LLM Chain** / **Groq Chat Model** node, select your Groq credential and a model (e.g. `llama-3.1-8b-instant`)

6. **Publish the CV Profile Update workflow**
   - Activate it, then open its **Form Trigger** node to get the public form URL
   - Submit your own name, email, and CV once to populate the Profile tab

7. **Activate the Weekly Job Search workflow**
   - It will run automatically every Sunday at 8am from then on
   - You can also click **"Execute workflow"** any time to test it manually

---


