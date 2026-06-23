# RegWave
**Regulatory Intelligence Platform for Financial Services Compliance**

RegWave aggregates, classifies, and summarizes regulatory updates from major US regulators — helping compliance teams stay current without manually monitoring sources.

---

## 🧭 Overview

This platform fully automates the collection, AI-processing, and structured review of regulatory updates and enforcement actions from all major U.S. financial regulators. It monitors for:

- **BSA** — Bank Secrecy Act
- **AML** — Anti-Money Laundering
- **KYC** — Know Your Customer
- **ABC** — Anti-Bribery & Corruption
- Fraud fines & enforcement actions
- Regulatory developments across financial services

The system runs completely automatically, every morning before business hours, and routes processed data through a structured compliance review workflow.

**💰 Cost: $0** under current free-tier limits. The system is designed to operate entirely within free tiers, though higher volume or API usage may require upgrades.

---

## 🏗️ Architecture

```
10 Regulatory Agencies → 18 Workflows
        ↓
n8n Workflows
   ├── 15 source workflows
   ├── 1 email digest delivery workflow
   ├── 1 on-demand share workflow (webhook-triggered)
   └── 1 global error handler
        ↓
AI Classification & Summarization
(Groq llama-4-scout-17b + OpenRouter Google Gemma 4.0 fallback)
        ↓
HuggingFace Inference API
(all-MiniLM-L6-v2 → 384-dimension embedding vector per article)
        ↓
Supabase (PostgreSQL + pgvector)
   ├── Article content, metadata & decisions
   ├── Embedding vectors for semantic search
   └── User subscriptions for email digest
        ↓
Caddy Reverse Proxy (HTTPS + auto SSL)
        ↓
Custom Domain (HTTPS secured, no port number)
        ↓
Web Frontend (Netlify)
   ├── Netlify Serverless Function (embed.js)
   │   └── Converts search queries → vectors via HuggingFace
   ├── Netlify Serverless Function (share.js)
   │   └── Proxies "share filtered" requests → n8n webhook (secret held server-side)
   └── Direct Supabase connection (data + auth)
        ↓
User Review Workflow
(Queue → Smart/Exact search → filter → Relevant / Irrelevant / Review Later → Stored decisions)
        ↓
Output & Distribution
   ├── Email Digest Delivery (scheduled daily/weekly per subscriber via Gmail)
   ├── Share Filtered (on-demand email of the current filtered set via Gmail)
   ├── Per-article Share (mailto to user's own mail client)
   └── CSV Export (current filtered view, with triage status & notes)
        ↓
Telegram Error Alerts (real-time)
```

---

## 🏛️ Regulatory Sources

The platform monitors 10 major U.S. regulatory agencies across 18 automated workflows (15 source workflows + 1 email digest + 1 error handler, plus 1 webhook-triggered share workflow):

| Source | Type | Collection Method |
|--------|------|-------------------|
| FDIC | News & Speeches | Manual crawl |
| FinCEN | News & Guidance | Manual crawl |
| OCC | News, Bulletins & Speeches | RSS + parse + PDF extraction |
| U.S. Treasury | News | Manual crawl |
| FINRA | News & Enforcement | Manual crawl |
| Federal Reserve | News & Speeches | RSS + parse |
| SEC | Press Releases | RSS + parse |
| FFIEC | Press Releases | Tavily search |
| DOJ | Press Releases | RSS + parse |
| CFTC | Press Releases & Speeches | Manual crawl |

> **Note:** Tavily is used exclusively for FFIEC. All other sources use direct crawling, HTML parsing, and content cleaning pipelines. OCC speeches are published as PDFs — these are handled via a dedicated PDF extraction branch. DOJ publishes the same announcement across both press release and video URLs — the video workflow filters these out at ingestion to prevent duplicates.

---

## ⚙️ Core Components

### 🔹 n8n — Automation Engine

**What it is:** An open-source workflow automation platform (like Zapier but self-hosted and free).

**What it does in this project:**
- Triggers all scheduled workflows daily (5:00–5:45 AM ET, staggered every 5 minutes)
- Fetches and parses regulatory content from all 10 agencies across 14 source workflows
- Calls AI APIs for classification and summarization
- Calls HuggingFace to generate a 384-dimension embedding vector for each article
- Checks Supabase for duplicates before processing
- Writes processed articles + embeddings to the database
- Delivers personalized email digests to subscribers at 7 AM daily
- Handles on-demand "share filtered" requests via a dedicated webhook workflow (triggered by the frontend)
- Sends Telegram alerts when any workflow fails

**Deployment:** Self-hosted in Docker on Oracle Cloud free tier VPS, accessed securely via Caddy reverse proxy over HTTPS.

---

### 🔹 HuggingFace — Semantic Embeddings

**What it is:** HuggingFace's Inference API, used to run the `sentence-transformers/all-MiniLM-L6-v2` model — a lightweight model that converts text into a 384-number vector representing its semantic meaning.

**How it fits into the system:**

There are three completely separate HuggingFace calls serving different purposes:

**Call 1 — Write path (n8n, at ingest time):**
For every new article scraped, n8n sends the article title + AI-generated summary to HuggingFace. The model returns 384 numbers — a "meaning fingerprint" of the article. These numbers are stored permanently in Supabase alongside the article content in a `vector(384)` column.

```
New article scraped
    → AI generates summary
    → HuggingFace: title + summary → [384 numbers]
    → Supabase: stores article + embedding permanently
```

**Call 2 — Read path (Netlify function, at search time):**
When a user types a search query in the frontend, a Netlify serverless function calls HuggingFace to convert the query into 384 numbers. Supabase then uses pgvector to find articles whose stored vectors are closest to the query vector — returning results sorted by semantic similarity, not keyword match.

```
User types "money laundering"
    → Netlify function → HuggingFace: query → [384 numbers]
    → Supabase search_items() RPC: compare vs all stored embeddings
    → Returns top matches sorted by semantic relevance
```

**Call 3 — Email digest path (n8n, at delivery time):**
When the email digest workflow runs, it converts each subscriber's saved themes (e.g. "Fraud BSA AML") into a 384-number vector via HuggingFace. Supabase then finds the articles most semantically similar to those themes from the relevant time window, and those are the articles delivered in the digest.

```
Subscriber themes: ["Fraud", "BSA", "AML"]
    → n8n → HuggingFace: "Fraud BSA AML" → [384 numbers]
    → Supabase match_items_for_digest() RPC: compare vs stored embeddings
    → Returns top matches from past 1 or 7 days
    → Formatted and delivered via Gmail
```

**Why this matters:** Searching or subscribing to "money laundering" returns articles about BSA compliance, illicit finance, and sanctions evasion — even if those exact words don't appear in the title. The system understands *meaning*, not just keywords.

**Think of it as a map:** Each article gets plotted at a location on a 384-dimensional map based on what it's *about*. Articles on similar topics land near each other. A search query or digest theme gets plotted on the same map, and the closest articles are returned — regardless of exact wording.

---

### 🔹 Search — Smart (Hybrid) & Exact Modes

The queue search bar offers two modes via a toggle, with **Smart** as the default:

**Smart (hybrid) — default.** Combines semantic recall with keyword precision. The query is embedded via the Netlify function and matched against stored vectors (pgvector), *and* simultaneously scored against article text by keyword. The two ranked lists are then fused using reciprocal-rank fusion (RRF), so an article that's both semantically related *and* contains the literal terms rises to the top. This fixes the common failure of pure-semantic search burying an obvious exact-term match.

```
User types "insider trading prediction markets"
    → Semantic ranking (meaning)  ─┐
    → Keyword ranking (literal terms) ─┤ → RRF fusion → final ranked results
```

**Exact (keyword).** Pure literal matching across title, summary, and full text — useful for exact names, codes, short queries, and exhaustive literal sweeps where you want only documents containing the exact terms.

**Automatic bypass.** Queries of 4 characters or fewer, and exact regulator-name searches (e.g. "SEC", "FDIC"), skip semantic processing entirely and use fast keyword matching — the mode toggle dims to indicate it has no effect for those queries.

**Filter intersection.** Search results are always scoped to whatever items are currently visible through active filters (status, category, regulator, date) — search and filters apply together.

---

### 🔹 Netlify Serverless Functions

Two Node.js serverless functions are deployed automatically alongside the frontend on Netlify.

#### `embed.js` — Embedding Proxy

**Why it exists:** The original implementation ran the embedding model directly in the browser using `@xenova/transformers` — downloading ~30MB of model weights on every page load and running inference client-side. This caused two problems:
1. Corporate firewalls block `huggingface.co` at the network level, breaking search on work computers
2. A 30MB model download on every session is wasteful and slow on mobile

**What it does:** Acts as a lightweight proxy — the browser sends a search query string to `/.netlify/functions/embed`, the function calls HuggingFace server-side, and returns the resulting vector to the browser. The browser then passes that vector directly to Supabase for the similarity search. The same function is also called by the n8n email digest workflow to embed subscriber themes.

```
Browser: "insider trading" → /.netlify/functions/embed
    → HuggingFace API (server-side, no firewall issues)
    → Returns [384 numbers] to browser
    → Browser → Supabase pgvector search
    → Results
```

**Key advantages over the browser-based approach:**
- Works on any network including corporate firewalls (request comes from Netlify's servers, not the user's browser)
- Zero download — no model weights loaded client-side
- First search responds in ~200ms instead of waiting for model warmup
- Works on any device including older mobile browsers
- Free — Netlify's free tier includes 125,000 function invocations per month

#### `share.js` — Share Proxy (frontend → n8n)

**Why it exists:** The "Share filtered" feature needs to send email on demand, but the n8n webhook requires a shared secret. Because the frontend is a public static site, that secret must never appear in page source. `share.js` holds the secret server-side as a Netlify environment variable and injects it when forwarding the request.

**What it does:** The browser POSTs the filtered articles + recipient to `/.netlify/functions/share` (no secret). The function validates the payload (valid email, at least one article, max 50), adds the `x-regwave-secret` header, and forwards to the n8n webhook, which checks the secret and sends the email.

```
Browser: { recipient, articles[] } → /.netlify/functions/share
    → adds secret header (from Netlify env var)
    → n8n webhook → secret check → format HTML → Gmail send
```

**Environment variables required (set in Netlify):**
- `SHARE_WEBHOOK_URL` — the n8n production webhook URL
- `SHARE_WEBHOOK_SECRET` — shared secret, must match the value checked in the n8n workflow

---

### 🔹 Supabase + pgvector — Database & Vector Search

**What it is:** Supabase is an open-source Firebase alternative built on PostgreSQL. pgvector is a PostgreSQL extension that adds native vector storage and similarity search.

**What it stores:**
- All processed regulatory articles (title, URL, source, date, category, summary, full text)
- AI-generated categories and summaries
- 384-dimension embedding vectors (one per article)
- User triage decisions (relevant / irrelevant / review later) with notes and timestamps
- User subscriptions for the email digest (frequency, regulators, themes)

**How vector search works:**
A PostgreSQL function called `search_items()` is stored inside the database. When called with a query vector, it uses pgvector's `<=>` operator to compute cosine similarity between the query and every stored article embedding, returning the closest matches:

```sql
SELECT * FROM items
ORDER BY embedding <=> query_embedding
LIMIT 200;
```

A second RPC function `match_items_for_digest()` handles the email digest path — filtering by regulator, time window, and similarity threshold before returning matched articles.

This all happens inside Postgres — no data travels to an external search service.

**Access control (Row-Level Security):**
- `items` is read-only for anonymous users (public regulatory news); inserts/updates require an authenticated user, and ingestion runs via the service-role key
- `decisions`, `subscriptions`, and `profiles` are protected per-user — a signed-in user can only read and write their own rows

**Why Supabase:**
- Free tier is generous (500MB database, unlimited API calls)
- Built-in authentication for multi-user login
- Row-Level Security (RLS) for access control
- pgvector extension available natively
- Real-time API accessible directly from the frontend

---

### 🔹 Sharing & Export

RegWave provides three independent ways to get articles out of the platform, each suited to a different need.

#### Email Digest (scheduled) — Personalized Regulatory Intelligence Delivery

**What it is:** An automated daily or weekly email sent to each subscriber containing the regulatory articles most relevant to their chosen themes and regulators, matched using AI semantic search.

**How it works:**

The email digest workflow runs at 7 AM daily and processes each active subscriber independently:

```
7 AM trigger
    → Fetch all active subscriptions from Supabase
    → Filter: who is due today? (daily = always, weekly = only on their chosen day)
    → For each qualifying subscriber:
        → Embed their themes via HuggingFace ("Fraud BSA AML" → [384 numbers])
        → Call match_items_for_digest() RPC:
            - days_back: 1 (daily) or 7 (weekly)
            - regulator_filter: subscriber's chosen regulators
            - match_threshold: 0.3 cosine similarity
            - match_count: up to 100 results
        → If no themes set: return all recent articles by date (no semantic filter)
        → Format HTML email with matched articles
        → Send via Gmail
        → Loop to next subscriber
```

**What subscribers configure (Alerts page):**
- **Frequency** — daily or weekly
- **Send day** — for weekly, which day of the week
- **Regulators** — which of the 10 agencies to monitor (leave empty = all)
- **Themes** — compliance topics to prioritize (e.g. BSA, AML, Fraud, Sanctions, KYC); leave empty = all recent content

**Why semantic matching matters for the digest:**
Two subscribers can have completely different digests from the same article pool. A subscriber focused on "Sanctions OFAC" will receive different articles than one focused on "Fraud enforcement" — even if both watch the same regulators — because the semantic matching finds articles by meaning, not by keyword overlap.

**Deduplication:** DOJ video URLs are filtered at ingestion so the same announcement never appears twice in the digest.

**Email design:** The digest email matches the RegWave brand — navy header with red accent stripe, category color badges (enforcement, regulatory, speeches, news), similarity score shown per article, and a direct link to update preferences.

#### Share Filtered (on-demand email)

**What it is:** A one-click way to email the **current filtered/searched set** of articles to any recipient — built for the workflow where an analyst narrows the queue (e.g. *Last 7 days + Relevant*) down to a handful of items, then a reviewer sends those to a stakeholder.

**How it works:** A "✉ Share filtered" button opens a modal to enter the recipient's email and an optional note. On send, the frontend posts the filtered articles to the `share.js` Netlify function, which forwards them (with the secret header) to a dedicated n8n webhook workflow that formats a branded HTML email and sends it via Gmail.

```
Filter to relevant set → ✉ Share filtered → modal (recipient + note)
    → /.netlify/functions/share (adds secret)
    → n8n webhook → secret check → format HTML digest → Gmail send
```

**Why it routes through n8n:** Unlike per-article share (which uses the user's own mail client), this sends a properly formatted HTML digest directly, with no per-message length limits, reusing the same Gmail sending path as the scheduled digest. A server-side limit caps a single send at 50 articles.

#### Per-Article Share (mailto)

**What it is:** A ✉ Share button on each article card that opens the user's own email client with that single article pre-filled — title, source, date, summary, and link. Best for quickly forwarding one item to a colleague. Requires no backend and leaves a copy in the sender's own Sent folder.

#### CSV Export

**What it is:** A "↓ Export CSV" button that downloads the **current filtered view** as a spreadsheet. Columns: Title, Source, Category, AI Category, Date Published, URL, Summary, Triage Status, and Notes. Values are CSV-escaped (quotes, commas, newlines handled) and the file includes a UTF-8 BOM so special characters render correctly in Excel. Triage status and notes reflect the logged-in user's own decisions. Best for large batches, offline review, or attaching to a report.

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
- All workflows

---

### 🔹 Netlify — Frontend Hosting

**What it is:** A free static site hosting platform with GitHub auto-deployment.

**What it hosts:**
- The compliance review web interface
- The `embed.js` serverless function (embedding proxy)
- The `share.js` serverless function (share proxy → n8n webhook)
- Connected directly to Supabase for data and auth
- Auto-deploys on every GitHub push

> **Important:** Netlify connects directly to Supabase for all data and auth — n8n writes ingestion data to Supabase independently on its own schedule. The frontend also calls n8n in exactly one place: the `share.js` function forwards "Share filtered" requests to a single n8n webhook for on-demand email sending. Aside from that one outbound webhook call, the frontend and n8n remain decoupled (they meet at the Supabase database).

---

### 🔹 AI Models — Classification & Summarization

**Primary:** Groq API — `meta-llama/llama-4-scout-17b-16e-instruct`
- Fast inference, free tier with generous limits
- Primary model for all source workflows

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

**Enhanced AI Classification Prompt:** The classification prompt uses explicit decision rules, keyword anchors, a priority hierarchy, and a hard exclusions list to reduce misclassification. A KEY TEST heuristic ("Would a compliance officer need to change a policy, procedure, or control because of this article?") appears at the top of the Regulatory Updates section to prevent over-classification. A dedicated speech-vs-rule distinction rule prevents named officials' statements about rules from being miscategorized as Regulatory Updates. Worked examples drawn from real articles anchor the model's behavior for the most common failure patterns.

**Manual reclassification** — users can correct AI-assigned categories from the UI; the original AI classification is retained in the database and flagged visually in the review queue.

---

## ⚠️ Failure Handling

The system is designed to fail safely and visibly:

- Duplicate articles → skipped automatically (URL-level and video URL filtering)
- AI failures → fallback provider is triggered
- Source errors → workflow fails and sends Telegram alert
- Parsing failures → article is skipped (no partial data stored)
- HuggingFace embedding failure → article is skipped; no partial records stored
- Empty digest → no email sent (zero-match guard in Format Email node)
- Share request with invalid email / no articles → rejected by the `share.js` function before reaching n8n

A global error workflow ensures no silent failures.

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
- Classification can be non-deterministic (manual override option is available)
- OCC PDF extraction depends on n8n's built-in Extract From File node; scanned/image PDFs will not extract correctly
- HuggingFace free inference tier may queue requests under heavy load (~200–400ms additional latency)
- Email delivery (scheduled digest and Share filtered) currently sends via a personal Gmail account through n8n; this is fine for low volume but is the main candidate for a dedicated transactional email service if usage scales
- Triage decisions are stored with user attribution and timestamp, but the system does not yet retain a full change-history of every status change (no immutable audit log)

These are acceptable trade-offs for a lightweight, free-tier system.

---

## 🔁 Deduplication Logic

The system uses a two-layer deduplication strategy:

**Pre-processing deduplication (workflow level)**
Each workflow checks whether an article URL already exists in Supabase before sending it to AI. This prevents unnecessary API calls and reduces token usage. DOJ workflows additionally filter out `/video/` URLs at the parsing stage, preventing the same announcement from being stored twice under different URLs.

**Database-level protection (Supabase)**
The `url` field is enforced as unique, ensuring no duplicate records are ever stored, even if multiple workflows run concurrently.

---

## 💻 Frontend (Web Interface)

**Hosting:** Netlify (free, auto-deploys from GitHub)
**Auth:** Supabase Auth (email/password, invite-only registration)
**Data:** Direct Supabase API connection

### Features

**Search & Discovery**
- Two search modes via a toggle: **Smart** (hybrid) and **Exact** (keyword), with Smart as the default
- Smart mode fuses semantic similarity (HuggingFace embeddings + pgvector) with keyword scoring via reciprocal-rank fusion, so conceptually related *and* literal-term matches both surface
- Exact mode does pure keyword matching across title, summary, and full text — for names, codes, and exhaustive literal sweeps
- Short queries (≤4 chars) and regulator-name searches (e.g. "SEC", "FDIC") bypass semantic search and use fast keyword matching; the mode toggle dims to indicate it has no effect there
- Search works from any network including corporate environments (routed via Netlify serverless function, not directly to HuggingFace)

**Filtering**
- Collapsible sidebar filter panel on desktop (click "‹ hide" to collapse, "› show" to expand — state persists per session)
- Mobile filter drawer — tap ⚙ Filters to open a full-screen drawer with chip-style filter controls
- Filter by Status, Category (Enforcement, Regulatory, Speeches, News), and Regulator independently; filters apply together with AND logic
- Date range filters with presets (Today, Last 7 days, Last 30 days, This month) and custom date picker
- Filters and search intersect — semantic results are scoped to whichever items are currently visible through active filters

**Review Workflow**
- Invite-only user registration — new users receive a magic link via email; no public sign-up
- Per-article triage panel with three decisions: **Relevant**, **Irrelevant**, **Review Later**
- Notes field per article, saveable independently of triage decision
- Manual category reclassification from the UI (original AI category preserved in DB, flagged visually)
- Triage decisions stored with user attribution and timestamp

**Sharing & Export**
- **Share filtered** — email the current filtered/searched set to any recipient as a branded HTML digest, sent on demand via the `share.js` Netlify function → n8n webhook → Gmail; modal for recipient + optional note
- **Per-article share** — ✉ Share button on each card opens the user's own mail client with that article pre-filled (title, source, date, summary, link)
- **CSV export** — download the current filtered view as a spreadsheet (Title, Source, Category, AI Category, Date, URL, Summary, Triage Status, Notes), CSV-escaped with a UTF-8 BOM for clean Excel rendering

**Email Digest (Alerts)**
- Each user configures their own digest preferences from the Alerts page
- Choose frequency (daily or weekly), regulators to monitor, and key compliance themes
- Themes drive semantic matching — articles are ranked by meaning, not keyword overlap
- Leave themes empty to receive all recent content; leave regulators empty to monitor all 10 agencies
- Digest emails are sent automatically at 7 AM, formatted to match the RegWave brand

**Analytics Dashboard**
- KPI cards (total items, decisions by type, pending, unique sources)
- Monthly volume chart, category donut, source breakdown table
- Decision rate bars, weekly trend, day-of-week distribution, yearly volume
- Decisions by category (stacked bar), recent activity feed
- Classification intelligence section (AI vs manually reclassified, reclassification rate by category)
- Summary tables — monthly breakdown of articles by regulator (Jan–Dec columns, regulator rows, total column), with a second table filtered to Regulatory Updates only; both tables update with the dashboard's date period filter
- Filterable by period (All time, 12 months, 90 days, 30 days) or custom date range

**UI / UX**
- Dark mode toggle in the nav bar — persists across sessions via localStorage, with flash-prevention so the correct theme loads before the page renders
- Fully responsive — optimised for both desktop and mobile:
  - Desktop: collapsible sidebar, full nav, all controls visible
  - Mobile: filter drawer, compressed nav (name hidden, sign out minimised), "+" button instead of "+ Add item", search count hidden to preserve row space, summary tables horizontally scrollable

---

## 🔧 Optimizations

### Workflow Optimizations

- **Staggered triggers** — workflows run every 5 minutes from 5:00–5:45 AM to avoid simultaneous API hammering
- **Deduplication before AI** — AI is only called for genuinely new articles (saves API costs and rate limits)
- **Embedding after classification** — HuggingFace is only called after the AI successfully classifies an article, so no embeddings are generated for skipped or failed items
- **Dynamic year in URLs** — all listing URLs use `new Date().getFullYear()` expression so no code changes are needed each January
- **Dual AI provider** — Groq primary + OpenRouter fallback prevents rate limit failures; two independent providers with separate limits
- **OCC PDF routing** — URL-based If node splits PDF and HTML articles before fetching, avoiding binary/text confusion downstream
- **DOJ video filtering** — `/video/` URLs filtered at parse time, before duplicate check or AI call, keeping the database clean at zero extra cost
- **Empty digest guard** — if zero articles match a subscriber's criteria, no email is sent

### Frontend Optimizations

- **Server-side embedding** — search query vectorization moved from browser (30MB model download + WebAssembly inference) to a Netlify serverless function; zero client-side model loading
- **Hybrid search fusion** — Smart mode merges semantic and keyword rankings client-side via reciprocal-rank fusion, improving precision without an extra round-trip
- **Short query bypass** — queries of 4 characters or fewer and exact regulator name matches skip the semantic search entirely and stay as fast keyword filtering
- **Debounced search** — semantic search fires only after 600ms of typing inactivity, avoiding redundant API calls mid-query
- **Server-side secret for sharing** — the n8n webhook secret lives in a Netlify environment variable inside `share.js`, never in public page source
- **Dark mode flash prevention** — `localStorage` is read and `body.dark` applied in an inline script before the DOM renders, so dark mode users never see a white flash

---

## 🔗 Integrations Summary

| Integration | Purpose | Cost |
|---|---|---|
| n8n (self-hosted) | Workflow automation engine | Free |
| Oracle Cloud | VPS hosting for n8n | Free (Always Free tier) |
| Caddy | Reverse proxy + automatic SSL | Free (open source) |
| Let's Encrypt | SSL certificate authority | Free |
| HuggingFace | Embedding model (all-MiniLM-L6-v2) | Free tier |
| Supabase | PostgreSQL + pgvector + Auth | Free tier |
| Netlify | Frontend hosting + serverless functions | Free tier |
| Groq | Primary AI (llama-4-scout-17b) | Free tier |
| OpenRouter | Fallback AI (google/gemma-4-27b-it) | Free tier |
| Tavily | FFIEC search (1 workflow only) | Free tier |
| Gmail | Email delivery (digest + share filtered) | Free |
| Telegram | Error monitoring alerts | Free |

---

## 📈 Key Advantages

- ✅ **TOTALLY FREE** — entire stack runs on free tiers permanently
- ✅ **Fully automated** — runs every morning without human intervention
- ✅ **10 agencies, 18 workflows** — comprehensive U.S. financial regulator coverage
- ✅ **Smart + Exact search** — hybrid semantic-plus-keyword ranking finds conceptually related articles without burying exact matches; Exact mode for literal lookups
- ✅ **Personalized email digest** — daily or weekly delivery matched to each subscriber's regulators and compliance themes via semantic search
- ✅ **Flexible sharing** — share the whole filtered set by email on demand, forward single articles via your own mail client, or export the filtered view to CSV
- ✅ **Corporate firewall friendly** — semantic search routed via Netlify function; no direct browser-to-HuggingFace calls
- ✅ **HTTPS secured** — custom domain with auto-renewing SSL certificate via Let's Encrypt
- ✅ **AI-powered** — automatic classification and plain-English summaries
- ✅ **Duplicate-free** — URL-level deduplication + DOJ video URL filtering at ingestion
- ✅ **Fault-tolerant** — dual AI providers + Telegram error alerts + Docker auto-restart
- ✅ **Decisions attributed** — all triage decisions stored with user attribution and timestamp
- ✅ **Invite-only access** — no public registration; users join via magic link only
- ✅ **Real-time monitoring** — Telegram alerts for any workflow failures
- ✅ **PDF-capable** — OCC speeches published as PDFs are automatically detected, fetched, and extracted
- ✅ **Dark mode** — system-wide dark theme with localStorage persistence and flash-free loading
- ✅ **Mobile responsive** — collapsible sidebar on desktop, filter drawer on mobile, fully usable on any device
- ✅ **Monthly summary tables** — analytics tab shows article volume by regulator and month at a glance

---

## 📋 Business Summary

This platform continuously monitors all major U.S. financial regulators, automatically collects and AI-processes regulatory updates every morning, and routes them through a structured compliance review workflow. Every user sees the same unified queue and triages articles as Relevant, Irrelevant, or Review Later. Every action is stored with user attribution and a timestamp.

Smart search — a hybrid of HuggingFace semantic embeddings (pgvector) and keyword ranking — lets compliance teams find relevant articles by meaning without losing exact-term matches, with an Exact mode for literal lookups. The same semantic engine powers the personalized email digest: each subscriber receives a curated set of articles matched to their chosen regulators and compliance themes, delivered daily or weekly without any manual curation. For ad-hoc distribution, reviewers can email the current filtered set to any stakeholder on demand, forward individual articles, or export the filtered view to CSV.

The system is secured with HTTPS via a custom domain, accessible from any network including corporate environments, and eliminates hours of daily manual monitoring across 10 agencies at zero ongoing cost.

---

*Built with n8n · Supabase · pgvector · Oracle Cloud · Caddy · Let's Encrypt · Netlify · HuggingFace · Groq · OpenRouter · Tavily · Gmail · Telegram*
