# RegWave
**Regulatory Intelligence Platform for Financial Services Compliance**

RegWave aggregates, classifies, and summarizes regulatory updates from major US regulators — 
helping compliance teams stay current without manually monitoring sources.

---

## 🧭 Overview

This platform **fully automates** the collection, AI-processing, and structured review of regulatory updates and enforcement actions from all major U.S. financial regulators. It monitors for:

- **BSA** — Bank Secrecy Act
- **AML** — Anti-Money Laundering
- **KYC** — Know Your Customer
- **ABC** — Anti-Bribery & Corruption
- **Fraud** fines & enforcement actions
- **Regulatory developments** across financial services

The system runs **completely automatically**, every morning before business hours, and routes processed data through a structured compliance review workflow with defined user roles.

> **💰 Cost: $0 under current free-tier limits. The system is designed to operate entirely within free tiers, though higher volume or API usage may require upgrades.**

---

## 🏗️ Architecture

```
8 Regulatory Agencies → 12 Workflows
        ↓
n8n Workflows (12 source flows + 1 error handler)
   ├── Manual crawling & parsing (7 flows)
   └── Tavily search API (1 flow — FFIEC only)
        ↓
AI Classification & Summarization
(Groq llama-4-scout-17b + OpenRouter Google Gemma 4.0 fallback)
        ↓
Supabase (PostgreSQL Database)
        ↓
Caddy Reverse Proxy (HTTPS + auto SSL)
        ↓
Custom Domain (HTTPS secured, no port number)
        ↓
Web Frontend (Netlify)
        ↓
User Review Workflow
(Doer → Reviewer → Stored decisions)
        ↓
Telegram Error Alerts (real-time)
```

---

## 🏛️ Regulatory Sources

The platform monitors **8 major U.S. regulatory agencies** across **12 automated workflows** (plus one error handler):

| Source | Type | Collection Method |
|--------|------|-------------------|
| **FDIC** | News & Speeches | Manual crawl |
| **FinCEN** | News & Guidance | Manual crawl |
| **OCC** | News, Bulletins & Speeches | RSS + parse + PDF extraction |
| **U.S. Treasury** | News | Manual crawl |
| **FINRA** | News & Enforcement | Manual crawl |
| **Federal Reserve** | News & Speeches | RSS + parse |
| **SEC** | Press Releases | RSS + parse |
| **FFIEC** | Press Releases | Tavily search |

> **Note:** Tavily is used **exclusively** for FFIEC. All other sources use direct crawling, HTML parsing, and content cleaning pipelines. OCC speeches are published as PDFs — these are handled via a dedicated PDF extraction branch (see Workflow 4).

---

## ⚙️ Core Components

### 🔹 n8n — Automation Engine
**What it is:** An open-source workflow automation platform (like Zapier but self-hosted and free).

**What it does in this project:**
- Triggers all workflows on a daily schedule (5:00–5:35 AM ET, staggered every 5 minutes)
- Fetches and parses regulatory content from all 8 agencies across 12 workflows
- Calls AI APIs for classification and summarization
- Checks Supabase for duplicates before processing
- Writes processed articles to the database
- Sends Telegram alerts when any workflow fails

**Deployment:** Self-hosted in Docker on Oracle Cloud free tier VPS, accessed securely via Caddy reverse proxy over HTTPS.

---

### 🔹 Caddy — Reverse Proxy & HTTPS
**What it is:** A lightweight, open-source web server that automatically handles SSL certificate provisioning and renewal.

**What it does in this project:**
- Sits in front of n8n and handles all incoming HTTPS traffic
- Automatically obtained a free SSL certificate from Let's Encrypt on first startup
- Auto-renews the certificate before expiry — zero manual maintenance ever

**Why Caddy instead of Nginx:**
- Zero-config SSL — just specify the domain name and it handles everything
- No manual certificate renewal commands
- Minimal resource usage (~13MB RAM)

---

### 🔹 Tavily — AI-Optimized Search (FFIEC only)
**What it is:** A search engine built specifically for AI/LLM workflows. Unlike traditional web scraping, Tavily returns clean, structured JSON results.

**Why used for FFIEC specifically:**
- FFIEC's website structure is complex to scrape reliably
- Tavily returns pre-structured results (title, URL, date, content) without custom parsing logic
- Reduces code complexity for this one source

**What it returns:**
```json
{
  "title": "FFIEC Releases Revised BSA/AML Examination Manual",
  "url": "https://www.ffiec.gov/press/...",
  "published_date": "2026-04-01",
  "content": "..."
}
```

---

### 🔹 Supabase — Database & Authentication
**What it is:** An open-source Firebase alternative built on PostgreSQL.

**What it stores:**
- All processed regulatory articles
- AI-generated categories and summaries
- Full article text
- Source metadata (regulator, date, URL)
- User decisions (approve/decline/unsure) with notes
- Decision history and timestamps

**Why Supabase:**
- Free tier is generous (500MB database, unlimited API calls)
- Built-in authentication for multi-user login
- Row-Level Security (RLS) for role-based data access
- Real-time API accessible from the frontend

---

### 🔹 Oracle Cloud — Backend Hosting
**What it is:** Oracle Cloud Infrastructure's Always Free tier provides a permanent, always-on virtual server.

**Specs used:**
- VM.Standard.E2.1.Micro (1 OCPU, 1GB RAM)
- Ubuntu 22.04 LTS
- 50GB boot volume
- 4GB swap file

**What runs on it:**
- n8n (Docker container)
- Caddy (reverse proxy + SSL termination)
- All 12 source workflows + error handler

---

### 🔹 Netlify — Frontend Hosting
**What it is:** A free static site hosting platform with GitHub auto-deployment.

**What it hosts:**
- The compliance review web interface
- Connected directly to Supabase for data and auth
- Auto-deploys on every GitHub push

**Important:** Netlify connects directly to Supabase — it does NOT connect to n8n. n8n writes data to Supabase on its own schedule. Netlify independently reads from and writes decisions to the same Supabase database.

---

### 🔹 AI Models — Classification & Summarization

**Primary:** Groq API — `meta-llama/llama-4-scout-17b-16e-instruct`
- Fast inference, free tier with generous limits
- Primary model for all 12 workflows

**Fallback:** OpenRouter — `google/gemma-4-27b-it:free`
- Activates automatically if Groq hits rate limits
- Different company/infrastructure = fully independent rate limits
- Free tier on OpenRouter

**Why dual providers:** If Groq experiences rate limits or downtime, OpenRouter automatically takes over — no manual intervention needed. Both providers serve capable models for free.

**What the AI does for each article:**
1. Classifies into exactly ONE category:
   - BSA/AML/KYC Fines & Enforcement
   - Regulatory Updates
   - Regulator Speeches & Testimony
   - Industry News & Trends
2. Writes a 2-sentence plain-English summary for compliance officers

**Enhanced AI Classification Prompt:**
The n8n classification prompt uses explicit decision rules, keyword anchors, a priority hierarchy, and a hard exclusions list to reduce misclassification. A KEY TEST heuristic ("Would a compliance officer need to change a policy, procedure, or control because of this article?") appears at the top of the Regulatory Updates section to prevent over-classification. A dedicated speech-vs-rule distinction rule prevents named officials' statements about rules from being miscategorised as Regulatory Updates. Worked examples drawn from real articles anchor the model's behaviour for the most common failure patterns.

Manual reclassification — users can correct AI-assigned categories from the UI; the original AI classification is retained in the database for audit purposes and flagged visually in the review queue.

---

## ⚠️ Failure Handling

The system is designed to fail safely and visibly:

- Duplicate articles → skipped automatically  
- AI failures → fallback provider is triggered  
- Source errors → workflow fails and sends Telegram alert  
- Parsing failures → article is skipped (no partial data stored)  

A global error workflow ensures **no silent failures**.

---

### 🔹 Telegram — Error Monitoring
A dedicated n8n error handler workflow sends real-time alerts to a Telegram bot whenever any workflow fails, including:
- Which workflow failed
- Which node failed
- The error message

This ensures failures are immediately visible and actionable.

---

## ⚠️ Known Limitations

- URL-based deduplication assumes one URL per unique article  
- HTML parsing depends on regulator website structure (can break if layout changes)  
- FFIEC relies on Tavily search rather than direct scraping  
- Classification can be deterministic (manual override option is available)
- OCC PDF extraction depends on n8n's built-in Extract From File node; scanned/image PDFs will not extract correctly

These are acceptable trade-offs for a lightweight, free-tier system.

---

## 🔁 Deduplication Logic

The system uses a **two-layer deduplication strategy**:

1. **Pre-processing deduplication (workflow level)**  
   Each workflow checks whether an article URL already exists in Supabase before sending it to AI. This prevents unnecessary API calls and reduces token usage.

2. **Database-level protection (Supabase)**  
   The `url` field is enforced as unique, ensuring no duplicate records are ever stored, even if multiple workflows run concurrently.

---

## 🗃️ Database Schema (Supabase)

### `items` table
| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid | Primary key |
| `title` | text | Article title |
| `url` | text | Source URL (unique) |
| `source` | text | Regulator name (e.g., "FDIC") |
| `date_published` | date | Publication date |
| `category` | text | AI-assigned category (current truth — reflects any manual override) |
| `summary` | text | AI-generated 2-sentence summary |
| `full_text` | text | Cleaned article content |
| `ai_category` | text | Original AI-assigned category before any manual reclassification |
| `manually_reclassified` | boolean | True if a user has overridden the AI category |
| `added_by` | uuid | FK to auth.users — who added the item |
| `created_at` | timestamp | When added to DB |

### `decisions` table (frontend)
| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid | Primary key |
| `item_id` | uuid | FK to items |
| `user_id` | uuid | FK to auth.users |
| `action` | text | approve / decline / unsure |
| `notes` | text | User notes |
| `decided_at` | timestamp | When decision was made |
| `created_at` | timestamp | Record creation timestamp |

---

## 👥 User Roles & Review Workflow

### 🟢 Doer
- Sees **ALL** incoming articles
- For each article must select:
  - ✅ **Approve** — relevant, should be escalated
  - ❌ **Decline** — not relevant
  - ❓ **Unsure** — needs second opinion
- Can add notes to any item
- Primary focus: approve or unsure (decline is available but secondary)

### 🔵 Reviewer
- Sees **ONLY** items marked `Approve` or `Unsure` by Doer
- Cannot see `Declined` items
- Can add their own notes
- Validates Doer decisions

### Business Rule
For each of the regulatory categories, the Doer must make a decision. The system enforces that `Approve` and `Unsure` items flow to the Reviewer queue. All decisions are stored permanently with timestamps and user attribution.

---

## 💻 Frontend (Web Interface)

**Hosting:** Netlify (free, auto-deploys from GitHub)  
**Auth:** Supabase Auth (email/password, multi-user)  
**Data:** Direct Supabase API connection

### Features:
- Multi-user login with role-based access
- Dashboard with analytics and charts
- Article list with category badges and AI summaries
- Per-article decision panel (Approve / Decline / Unsure buttons)
- Notes field per article
- Manual category reclassification from the UI (original AI category preserved in DB)
- **Dual-row filter system** — filter by Category (Enforcement, Regulatory, Speeches, News) and by Regulator (FDIC, FinCEN, OCC, Treasury, FINRA, Federal Reserve, SEC, FFIEC) independently; both filters apply together with AND logic
- Date range filters with presets (Today, Last 7 days, Last 30 days, This month) and custom date picker
- Reviewer queue with stat cards and the same dual-row filter system as the Doer queue
- **Analytics tab** including:
  - KPI cards (total items, decisions by type, pending, unique sources)
  - Monthly volume chart, category donut, source breakdown table
  - Decision rate bars, weekly trend, day-of-week distribution, yearly volume
  - Decisions by category (stacked bar), recent activity feed
  - Classification intelligence section (AI vs manually reclassified, reclassification rate by category)
  - **Summary tables** — monthly breakdown of articles by regulator (Jan–Dec columns, regulator rows, total column), with a second table filtered to Regulatory Updates only; both tables update with the dashboard's date period filter
- Fully responsive (works on mobile)

---

## 🔧 Optimizations

### Server RAM Optimizations (Oracle Free Tier)

| Optimization | RAM Saved | Safe? | Notes |
|---|---|---|---|
| Remove Snap | ~150MB | ✅ | Largest single win |
| Remove oracle-cloud-agent | ~50MB | ✅ | Removed automatically with snap; Oracle metrics stop but instance runs fine |
| Disable multipathd | ~27MB | ✅ | Multi-path disk manager, not needed on single-disk VPS |
| Disable apport + whoopsie | ~15MB | ✅ | Ubuntu crash reporters, useless on a server |
| Limit journald to 50MB | variable | ✅ | Prevents log files from consuming unbounded RAM |
| 4GB swap file | prevents crashes | ✅ | Emergency RAM overflow buffer; n8n spills into swap instead of crashing |
| `restart: always` in Docker | reliability | ✅ | n8n auto-restarts after any crash or server reboot |
| RAM logging cron job | visibility | ✅ | Logs `free -m` every minute to `~/ram_log.txt` |
| Do NOT limit Node.js memory | n/a | ✅ | Swap handles overflow; capping Node causes workflow crashes |

### Workflow Optimizations

- **Staggered triggers** — workflows run every 5 minutes from 5:00–5:35 AM to avoid simultaneous API hammering
- **Deduplication before AI** — AI is only called for genuinely new articles (saves API costs and rate limits)
- **Dynamic year in URLs** — all listing URLs use `new Date().getFullYear()` expression so no code changes are needed each January
- **Dual AI provider** — Groq primary + OpenRouter fallback prevents rate limit failures; two independent providers with separate limits
- **OCC PDF routing** — URL-based If node splits PDF and HTML articles before fetching, avoiding binary/text confusion downstream; n8n's built-in Extract From File node handles PDF text extraction with no external dependencies

---

## 🔗 Integrations Summary

| Integration | Purpose | Cost |
|-------------|---------|------|
| **n8n** (self-hosted) | Workflow automation engine | Free |
| **Oracle Cloud** | VPS hosting for n8n | Free (Always Free tier) |
| **Caddy** | Reverse proxy + automatic SSL | Free (open source) |
| **Let's Encrypt** | SSL certificate authority | Free |
| **Supabase** | PostgreSQL database + Auth | Free tier |
| **Netlify** | Frontend hosting | Free tier |
| **Groq** | Primary AI (llama-4-scout-17b) | Free tier |
| **OpenRouter** | Fallback AI (google/gemma-4-27b-it) | Free tier |
| **Tavily** | FFIEC search (1 workflow only) | Free tier |
| **Telegram** | Error monitoring alerts | Free |

---

## 📈 Key Advantages

- ✅ **TOTALLY FREE** — entire stack runs on free tiers permanently
- ✅ **Fully automated** — runs every morning without human intervention
- ✅ **8 agencies, 12 workflows** — comprehensive U.S. financial regulator coverage
- ✅ **HTTPS secured** — custom domain with auto-renewing SSL certificate via Let's Encrypt
- ✅ **Workplace accessible** — HTTPS domain works on corporate networks that block raw IP addresses
- ✅ **AI-powered** — automatic classification and plain-English summaries
- ✅ **Duplicate-free** — built-in deduplication prevents redundant data
- ✅ **Fault-tolerant** — dual AI providers + Telegram error alerts + Docker auto-restart
- ✅ **Audit-ready** — all decisions stored with timestamps and user attribution
- ✅ **Role-based** — structured Doer → Reviewer workflow
- ✅ **Real-time monitoring** — Telegram alerts for any workflow failures
- ✅ **PDF-capable** — OCC speeches published as PDFs are automatically detected, fetched, and extracted using n8n's built-in tooling
- ✅ **Regulator filtering** — filter any queue by specific regulator (FDIC, FinCEN, OCC, Treasury, FINRA, Federal Reserve, SEC, FFIEC) alongside category filters, with AND logic
- ✅ **Monthly summary tables** — analytics tab shows article volume by regulator and month at a glance, with a dedicated Regulatory Updates view

---

## 🚀 Future Improvements

- Email/SMS digest alerts for new high-priority items
- Multi-tenant support for multiple compliance teams
- Integration with existing compliance management systems
- Mobile app for on-the-go review
- Expanded international regulator coverage
- Upgrade Oracle VM to A1.Flex (4 OCPU / 24GB RAM) when free tier capacity becomes available

---

## 📋 Business Summary

This platform continuously monitors all major U.S. financial regulators, automatically collects and AI-processes regulatory updates every morning, and routes them through a structured compliance review workflow. Analysts (Doers) review AI-summarized articles and make initial decisions; Reviewers validate those decisions. Every action is permanently stored for audit purposes.

The system is secured with HTTPS via a custom domain — accessible from any network including corporate environments — and eliminates hours of daily manual monitoring across 8 agencies at zero ongoing cost.

---

*Built with n8n · Supabase · Oracle Cloud · Caddy · Let's Encrypt · Netlify · Groq · OpenRouter · Tavily · Telegram*
