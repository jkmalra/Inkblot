# Inkblot — System Architecture

> **Inkblot** is an automated censorship observatory that detects when content disappears from Indian social media platforms, creates cryptographic proof that the content existed, and reveals patterns of suppression over time.

---

## Table of Contents

1. [Problem Statement](#1-problem-statement)
2. [Project Philosophy](#2-project-philosophy)
3. [High-Level Architecture](#3-high-level-architecture)
4. [System Layers](#4-system-layers)
   - [Layer 1 — Sources](#layer-1--sources)
   - [Layer 2 — The Watcher](#layer-2--the-watcher)
   - [Layer 3 — The Archive](#layer-3--the-archive)
   - [Layer 4 — The Database](#layer-4--the-database)
   - [Layer 5 — The Intelligence Layer](#layer-5--the-intelligence-layer)
   - [Layer 6 — The Public API](#layer-6--the-public-api)
   - [Layer 7 — The Dashboard](#layer-7--the-dashboard)
5. [Data Flow](#5-data-flow)
6. [Database Schema](#6-database-schema)
7. [Technology Stack](#7-technology-stack)
8. [Infrastructure and Deployment](#8-infrastructure-and-deployment)
9. [Security and Legal Considerations](#9-security-and-legal-considerations)
10. [Priority Tier System](#10-priority-tier-system)
11. [Anti-Scraping Strategy](#11-anti-scraping-strategy)
12. [Proof System — How Cryptographic Evidence Works](#12-proof-system--how-cryptographic-evidence-works)
13. [AI Classification Pipeline](#13-ai-classification-pipeline)
14. [API Reference](#14-api-reference)
15. [Build Phases](#15-build-phases)
16. [Target Users](#16-target-users)
17. [Project Constraints and Known Hard Problems](#17-project-constraints-and-known-hard-problems)

---

## 1. Problem Statement

In India, thousands of pieces of social media content are removed every month under the IT Rules 2021. Platforms are legally required to comply with government takedown orders within 36 hours or face significant fines. As a result, platforms over-remove and self-censor at scale — quietly, with no transparency.

When content disappears:

- The creator has no proof it ever existed
- The public cannot see patterns of who is being silenced
- Journalists have no verifiable, admissible evidence
- Researchers have no systematic dataset
- Courts cannot act without documented evidence

The missing piece is not a platform. It is **evidence infrastructure** — a system that watches, records, proves, and reveals.

---

## 2. Project Philosophy

Inkblot is built on four principles:

**Evidence over opinion.** Every claim the system makes is backed by verifiable, cryptographically-signed data. We do not editorialize — we record and reveal.

**Infrastructure over product.** Inkblot is designed to be depended on by journalists, lawyers, researchers, and civil society organizations. It exposes a public API so others can build on top of it.

**Transparency by design.** All data collected is public. All methodology is documented. All code is open source. The system cannot be accused of operating in the dark.

**Minimal footprint, maximum credibility.** The system does not host content permanently. It archives existence proofs, not full media files. This limits legal exposure while preserving the evidence chain.

---

## 3. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                          SOURCES                                │
│   Twitter/X · YouTube · Instagram · Facebook · Govt. Orders    │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                       THE WATCHER                               │
│         Python scrapers · Playwright · Celery scheduler         │
│         Detects removals by comparing current state vs stored   │
└──────────────┬───────────────────────────────┬──────────────────┘
               │                               │
               ▼                               ▼
┌──────────────────────────┐     ┌─────────────────────────────────┐
│       THE ARCHIVE        │────▶│         THE DATABASE            │
│  SHA-256 hash            │     │  PostgreSQL · SQLAlchemy        │
│  OpenTimestamps proof    │     │  Every incident, fully logged   │
│  Wayback Machine submit  │     │  with hash reference + tags     │
│  IPFS screenshot store   │     └──────────────┬──────────────────┘
└──────────────────────────┘                    │
                                                │
               ┌────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────────────────┐
│                   AI INTELLIGENCE LAYER                         │
│   Claude API · Auto-tagging · Pattern detection · Risk scoring  │
└───────────────────────────┬─────────────────────────────────────┘
                            │
               ┌────────────┴──────────────┐
               │                           │
               ▼                           ▼
┌──────────────────────────┐   ┌───────────────────────────────────┐
│       PUBLIC API         │   │        PUBLIC DASHBOARD           │
│  FastAPI · REST          │   │  React · Live feed · India map    │
│  JWT auth · Open data    │   │  Timeline · Per-creator history   │
└──────────────────────────┘   └───────────────────────────────────┘
```

---

## 4. System Layers

### Layer 1 — Sources

Inkblot monitors content from the following source categories:

| Source | What is Monitored | Method |
|---|---|---|
| Twitter / X | Public tweets, threads from tracked accounts | Playwright browser automation + Twitter API (limited) |
| YouTube | Video pages, channel pages for tracked creators | YouTube Data API v3 + scraping |
| Instagram | Public posts, reels from tracked profiles | Playwright (no official API) |
| Facebook | Public pages and posts | Playwright (no official API) |
| Government orders | TRAI URL blocking list, MeitY transparency reports | HTTP polling of public government endpoints |
| Twitter Transparency | Twitter's own government request disclosures | Twitter Transparency Report API |

**Source selection logic:**
Sources are not crawled blindly. A curated watchlist of high-risk creators is maintained — journalists, activists, comedians, regional-language content creators in politically sensitive areas (UP, Kashmir, Manipur, Assam). New sources can be submitted by users via the API or dashboard.

---

### Layer 2 — The Watcher

The Watcher is the core data collection engine. It is a fleet of Python workers running on a Celery task queue.

**Responsibilities:**
- Load a batch of tracked URLs from the database
- Check each URL for current status (200 OK, 404, redirect, restricted)
- Compare current state to the last known state
- If a change is detected (especially a removal), trigger the Archive pipeline and log the incident
- Submit new content to the Archive pipeline on first discovery

**Detection logic:**

```python
# Simplified detection flow
def check_url(url: str, last_known_state: str) -> ContentStatus:
    current = fetch_url_status(url)  # uses Playwright for JS-rendered pages

    if current.status == 404:
        return ContentStatus.REMOVED
    if current.status == 200 and "content unavailable" in current.body.lower():
        return ContentStatus.REMOVED
    if current.redirects_to_login and not last_known_state.requires_login:
        return ContentStatus.RESTRICTED
    if current.status == 200 and content_hash(current.body) != last_known_state.hash:
        return ContentStatus.MODIFIED

    return ContentStatus.ALIVE
```

**Scheduling (Celery + Redis broker):**

| Tier | Description | Check Frequency |
|---|---|---|
| Tier 1 | High-risk creators, journalists, active news cycles | Every 1 hour |
| Tier 2 | Known creators, regional language accounts | Every 6 hours |
| Tier 3 | General tracked accounts, historical records | Every 24 hours |

---

### Layer 3 — The Archive

The Archive pipeline runs immediately when new content is first discovered. Its job is to create an irrefutable, time-stamped proof that a specific piece of content existed at a specific moment.

**Step 1 — Generate SHA-256 hash**

The full text content (and metadata) of the page is hashed using SHA-256. This fingerprint is unique to that exact content. If even one character changes, the hash changes completely.

```python
import hashlib
import json

def generate_content_hash(content: dict) -> str:
    canonical = json.dumps(content, sort_keys=True, ensure_ascii=False)
    return hashlib.sha256(canonical.encode("utf-8")).hexdigest()
```

**Step 2 — Submit to OpenTimestamps**

The hash is submitted to OpenTimestamps, a free, open protocol that embeds hashes into the Bitcoin blockchain. Bitcoin's blockchain is immutable and globally distributed — no government or organization can alter it. This creates a public, third-party verified timestamp proving that at a specific moment in time, this specific hash (and therefore this specific content) existed.

The OpenTimestamps proof file (`.ots`) is stored alongside the incident record.

**Step 3 — Submit to Wayback Machine**

The Internet Archive's Save Page Now API is called to force an archival snapshot of the URL. This creates a permanent, publicly accessible snapshot at `web.archive.org/web/[timestamp]/[url]`.

```python
import httpx

async def submit_to_wayback(url: str) -> str:
    response = await httpx.post(
        "https://web.archive.org/save/",
        data={"url": url},
        headers={"Accept": "application/json"}
    )
    return response.headers.get("Content-Location", "")
```

**Step 4 — Store screenshot to IPFS**

A full-page screenshot (PNG) is captured using Playwright and stored on IPFS (InterPlanetary File System), a decentralized storage network. The IPFS content hash (CID) is stored in the database. Because IPFS is content-addressed, the file cannot be altered after upload.

**What the Archive proves:**

When content is later removed, Inkblot can present:
1. The original content text
2. The SHA-256 fingerprint of that exact content
3. An OpenTimestamps `.ots` file proving when that hash was recorded on the Bitcoin blockchain
4. A Wayback Machine URL showing the live snapshot
5. An IPFS link to the full-page screenshot

This chain of evidence is verifiable by any independent party without trusting Inkblot itself.

---

### Layer 4 — The Database

PostgreSQL is the single source of truth for all incident data.

**Core design decisions:**
- Every content URL gets a record on first discovery, regardless of whether it is later removed
- Removal is a state change on an existing record, not a new record
- Proof artifacts (hash, OTS file path, Wayback URL, IPFS CID) are part of the core record
- Tags are stored as a PostgreSQL array with a separate tag taxonomy table for normalization
- All timestamps are stored in UTC

See full schema in [Section 6](#6-database-schema).

---

### Layer 5 — The Intelligence Layer

The Intelligence Layer runs as async background workers that operate on records already in the database. It does not block the main collection pipeline.

**Component 1 — Auto-tagger**

When new content is archived, its text is passed to the Claude API with a structured prompt requesting JSON output classifying the content across a defined taxonomy:

```python
CLASSIFICATION_PROMPT = """
You are a content classifier for a censorship monitoring system in India.
Classify the following content into one or more of these categories:
- political_criticism (criticism of government, politicians, or parties)
- police_accountability (coverage of police action, encounters, arrests)
- religious_commentary (content involving religion, communal issues)
- caste_issues (content about caste discrimination or reservations)
- lgbtq (content about LGBTQ+ rights or identity)
- protest_coverage (coverage of protests, demonstrations)
- judicial_criticism (criticism of courts or judicial decisions)
- corporate_criticism (criticism of specific corporations)
- regional_conflict (content about conflict zones: Kashmir, Manipur, etc.)
- other

For each applicable category, provide a confidence score from 0.0 to 1.0.
Respond only with a JSON object. No explanation.

Content: {content_text}
"""
```

Tags with confidence below 0.65 are stored but not surfaced in public queries by default. A human review queue surfaces low-confidence classifications for optional manual verification.

**Component 2 — Pattern Detector**

Runs as a nightly batch job. Queries the database for anomalies:

- Removal rate spike detection: if the 7-day rolling average for any tag category exceeds 2 standard deviations above the 90-day baseline, an alert is generated
- Election correlation: compares removal rates in the 30 days before and after known election dates (stored as a reference table)
- Platform comparison: weekly aggregation of removal rates per platform per tag category
- Creator risk profiling: calculates a per-creator removal rate and topic exposure score

**Component 3 — Semantic Search Index**

All archived content is embedded using `sentence-transformers` (the `paraphrase-multilingual-MiniLM-L12-v2` model, which supports Hindi and other Indian languages). Embeddings are stored in PostgreSQL using the `pgvector` extension.

This enables semantic search across the entire corpus — journalists can search for "content about Adani" and find relevant archived content even if those exact words are not in it.

---

### Layer 6 — The Public API

A FastAPI application exposing the entire dataset as a REST API. Designed for consumption by journalists, researchers, and civil society organizations.

**Authentication:**
- Read-only endpoints: open, no authentication required
- Write endpoints (submitting new URLs for monitoring): API key required, obtained via email registration
- No rate limiting on read endpoints for documented research use

**Base URL:** `https://api.inkblot.in/v1`

See full endpoint reference in [Section 14](#14-api-reference).

---

### Layer 7 — The Dashboard

A public-facing React web application. The primary interface for non-technical users.

**Key views:**

| View | Description |
|---|---|
| Live feed | Real-time stream of newly detected removals |
| India map | Choropleth map colored by removal density per state |
| Timeline | Chart of daily/weekly removal counts, overlaid with political events |
| Platform breakdown | Stacked bar chart comparing removal rates across platforms |
| Creator page | Full history of a specific creator's removal incidents |
| Incident page | Full details of one incident: content, proof chain, timeline |
| Tag explorer | Browse all incidents by topic category |
| Search | Full-text and semantic search across archived content |

---

## 5. Data Flow

### New content discovery flow

```
Watcher detects new URL
        │
        ▼
Fetch content via Playwright
        │
        ▼
Generate SHA-256 hash
        │
        ├──▶ Submit hash to OpenTimestamps ──▶ Store .ots proof file
        │
        ├──▶ Submit URL to Wayback Machine ──▶ Store archive URL
        │
        ├──▶ Capture screenshot ──▶ Upload to IPFS ──▶ Store CID
        │
        ▼
Write ContentRecord to PostgreSQL (status: ALIVE)
        │
        ▼
Enqueue auto-tagging job (async, Claude API)
        │
        ▼
Tags written back to ContentRecord
```

### Removal detection flow

```
Watcher checks URL on schedule
        │
        ▼
URL returns 404 / removal page / login wall
        │
        ▼
Compare against stored ContentRecord
        │
        ▼
Confirm removal (retry 3x over 30 minutes to rule out transient errors)
        │
        ▼
Update ContentRecord: status=REMOVED, removed_at=now()
        │
        ▼
Write RemovalEvent record
        │
        ├──▶ Trigger pattern detection worker
        │
        └──▶ Publish to WebSocket (live feed on dashboard)
```

---

## 6. Database Schema

```sql
-- Platforms reference table
CREATE TABLE platforms (
    id          SERIAL PRIMARY KEY,
    name        VARCHAR(50) UNIQUE NOT NULL,  -- 'twitter', 'youtube', etc.
    display_name VARCHAR(100) NOT NULL
);

-- Creators being monitored
CREATE TABLE creators (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    handle          VARCHAR(200) NOT NULL,
    platform_id     INTEGER REFERENCES platforms(id),
    display_name    VARCHAR(200),
    region          VARCHAR(100),         -- Indian state or 'national'
    language        VARCHAR(50),          -- primary content language
    priority_tier   SMALLINT DEFAULT 2,   -- 1=hourly, 2=6hr, 3=daily
    added_at        TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE (handle, platform_id)
);

-- Core content record — one row per piece of tracked content
CREATE TABLE content_records (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    creator_id      UUID REFERENCES creators(id),
    platform_id     INTEGER REFERENCES platforms(id),
    url             TEXT NOT NULL UNIQUE,
    content_text    TEXT,
    content_hash    CHAR(64) NOT NULL,          -- SHA-256 hex
    content_type    VARCHAR(50),                -- 'tweet', 'video', 'post', 'reel'
    status          VARCHAR(20) DEFAULT 'alive', -- 'alive', 'removed', 'restricted', 'modified'
    first_seen_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    removed_at      TIMESTAMPTZ,
    survival_hours  NUMERIC GENERATED ALWAYS AS (
                        EXTRACT(EPOCH FROM (removed_at - first_seen_at)) / 3600
                    ) STORED,
    -- Archive proof references
    ots_proof_path  TEXT,               -- path to .ots file in object storage
    wayback_url     TEXT,               -- web.archive.org snapshot URL
    ipfs_cid        TEXT,               -- IPFS content identifier for screenshot
    -- Government order linkage
    govt_order_ref  TEXT,               -- MeitY/TRAI order number if traceable
    is_govt_ordered BOOLEAN DEFAULT FALSE,
    -- Metadata
    language        VARCHAR(50),
    embedding       VECTOR(384),        -- sentence-transformers embedding for semantic search
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Tag taxonomy
CREATE TABLE tags (
    id      SERIAL PRIMARY KEY,
    slug    VARCHAR(100) UNIQUE NOT NULL,  -- 'political_criticism', 'caste_issues', etc.
    label   VARCHAR(200) NOT NULL
);

-- Content-tag junction with confidence scores
CREATE TABLE content_tags (
    content_id      UUID REFERENCES content_records(id) ON DELETE CASCADE,
    tag_id          INTEGER REFERENCES tags(id),
    confidence      NUMERIC(3,2) NOT NULL,  -- 0.00 to 1.00
    tagged_by       VARCHAR(20) DEFAULT 'ai',  -- 'ai' or 'human'
    tagged_at       TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (content_id, tag_id)
);

-- Individual removal events (for audit trail)
CREATE TABLE removal_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    content_id      UUID REFERENCES content_records(id),
    detected_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    removal_reason  TEXT,           -- platform-provided reason if any
    http_status     SMALLINT,       -- 404, 302, etc.
    response_body   TEXT,           -- what the page returned at removal time
    confirmed       BOOLEAN DEFAULT FALSE,  -- after 3x retry confirmation
    confirmed_at    TIMESTAMPTZ
);

-- Pattern detection alerts
CREATE TABLE pattern_alerts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    alert_type      VARCHAR(100) NOT NULL,  -- 'spike', 'election_correlation', etc.
    tag_id          INTEGER REFERENCES tags(id),
    platform_id     INTEGER REFERENCES platforms(id),
    region          VARCHAR(100),
    description     TEXT NOT NULL,
    severity        VARCHAR(20),    -- 'low', 'medium', 'high'
    detected_at     TIMESTAMPTZ DEFAULT NOW(),
    is_published    BOOLEAN DEFAULT FALSE
);

-- Election / political event reference table (for correlation analysis)
CREATE TABLE political_events (
    id          SERIAL PRIMARY KEY,
    name        VARCHAR(200) NOT NULL,
    event_type  VARCHAR(50),    -- 'election', 'protest', 'verdict', 'incident'
    region      VARCHAR(100),
    starts_at   DATE NOT NULL,
    ends_at     DATE
);

-- Key indexes
CREATE INDEX idx_content_records_status ON content_records(status);
CREATE INDEX idx_content_records_removed_at ON content_records(removed_at);
CREATE INDEX idx_content_records_creator ON content_records(creator_id);
CREATE INDEX idx_content_records_platform ON content_records(platform_id);
CREATE INDEX idx_content_records_embedding ON content_records USING ivfflat (embedding vector_cosine_ops);
CREATE INDEX idx_content_tags_tag ON content_tags(tag_id);
CREATE INDEX idx_removal_events_detected ON removal_events(detected_at);
```

---

## 7. Infrastructure and Deployment

### Why hosted outside India

The system is intentionally hosted outside Indian jurisdiction. Servers in India are subject to the IT Rules 2021 and can be compelled to hand over data or take down the service under vague "national security" provisions. Hosting on Fly.io (US/EU regions) ensures the infrastructure is protected by stronger legal frameworks.

The domain registrar should also be outside India. Cloudflare DNS provides DDoS protection and an additional legal layer.

### Deployment topology

```
GitHub (source)
    │
    ▼ GitHub Actions CI/CD
    │
    ├──▶ Fly.io (API + Workers)
    │       ├── FastAPI app (2 instances, autoscale)
    │       ├── Celery workers (scraping fleet, autoscale 2–10)
    │       └── Celery beat (scheduler)
    │
    ├──▶ Supabase (PostgreSQL + pgvector)
    │
    ├──▶ Redis (Upstash — managed Redis)
    │
    ├──▶ Cloudflare R2 (proof file storage)
    │
    └──▶ Vercel (React dashboard, static)
```

### Environment variables

```
DATABASE_URL=postgresql+asyncpg://...
REDIS_URL=redis://...
ANTHROPIC_API_KEY=sk-ant-...
CLOUDFLARE_R2_BUCKET=...
CLOUDFLARE_R2_KEY=...
CLOUDFLARE_R2_SECRET=...
PINATA_API_KEY=...   # IPFS via Pinata
SECRET_KEY=...       # JWT signing
ENVIRONMENT=production
```

---

## 8. Security and Legal Considerations

### Data minimization

Inkblot does not permanently store full media files (videos, images). It stores:
- Text content (tweets, captions, descriptions)
- A single screenshot at time of first discovery
- Proof artifacts (hashes, timestamps)
- Metadata (URL, creator, timestamp, platform)

This is the minimum data required to prove existence and detect removal.

### What Inkblot is not

- Not a proxy or VPN — it does not help users access blocked content
- Not a platform — it does not host user-generated content
- Not a circumvention tool — it works entirely with public content

### Legal framing

Inkblot is a journalism and academic research infrastructure tool. Monitoring public content and documenting removals is consistent with journalistic practice and academic research methodology. The closest legal analogues are existing tools like GreatFire.org (China) and Roskomsvoboda (Russia), both of which operate legally.

### Responsible disclosure

If Inkblot's monitoring infrastructure is itself targeted (scrapers blocked, server attacked), this is documented and published as a finding — the targeting of a transparency tool is itself newsworthy.

---

## 9. Priority Tier System

Creators are assigned to tiers based on assessed risk and importance:

| Tier | Criteria | Check Frequency | Max Creators |
|---|---|---|---|
| 1 | Active journalists, court-ordered takedowns in past, breaking news | Every 1 hour | 500 |
| 2 | Known activists, regional language creators, political commentators | Every 6 hours | 5,000 |
| 3 | General tracked creators, historical record maintenance | Every 24 hours | 50,000 |

Tier assignment is automated based on:
- Past removal history (creators who have been removed before are promoted to Tier 1)
- Topic tags on their recent content (content tagged `political_criticism` or `protest_coverage` raises tier)
- User nominations via the public API (requires review)

---

## 10. Anti-Scraping Strategy

Platform anti-scraping is the primary ongoing engineering challenge. The following strategies are layered:

| Strategy | Implementation |
|---|---|
| Browser automation | Playwright renders full JavaScript, appearing as a real browser |
| Rotating proxies | Residential proxy pool (Webshare or Smartproxy) rotated per request |
| Request rate limiting | Per-platform configurable delays between requests (e.g. 3–8 seconds randomized) |
| User-agent rotation | Randomized real browser user-agent strings |
| Session management | Maintain warm browser sessions rather than cold starting per request |
| Retry with backoff | On HTTP 429 or 503, exponential backoff before retry |
| Platform API fallback | Where official APIs exist (YouTube Data API, Twitter API v2), prefer them over scraping |
| Failure logging | All failed checks logged with reason; if failure rate for a platform exceeds 40% over 6 hours, alert raised |

---

## 11. Proof System — How Cryptographic Evidence Works

The proof chain for each piece of content consists of four independent, verifiable artifacts:

### Artifact 1 — SHA-256 Content Hash

```
Input:  { "url": "...", "text": "...", "author": "...", "timestamp": "..." }
         (sorted keys, UTF-8 JSON)
Output: 64-character hex string, e.g.
         "a3f1c9e2b4d7f0e8c1a2b3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4"
```

Anyone can verify: take the original content, produce the same JSON, hash it with SHA-256, compare. If the hashes match, the content is identical to what Inkblot recorded.

### Artifact 2 — OpenTimestamps Proof (.ots file)

The SHA-256 hash is submitted to OpenTimestamps, which batches it with other hashes and embeds the Merkle root into a Bitcoin transaction. The `.ots` file returned contains a cryptographic path proving that this specific hash was included in a specific Bitcoin block.

Verification: `ots verify content_hash.ots` — the OpenTimestamps CLI checks the Bitcoin blockchain directly.

**This timestamp cannot be faked, backdated, or altered by anyone** — including Inkblot.

### Artifact 3 — Wayback Machine Snapshot

URL: `https://web.archive.org/web/[YYYYMMDDHHmmss]/[original_url]`

The Internet Archive is an independent US nonprofit. Their snapshot is timestamped by their own systems, independent of Inkblot. A journalist can cite the Wayback URL directly without citing Inkblot at all.

### Artifact 4 — IPFS Screenshot

CID: `ipfs://Qm...`

The screenshot is content-addressed on IPFS — the CID is derived from the file's content, so the file cannot be changed without the CID changing. Anyone can retrieve it via any IPFS gateway.

### Reading the complete proof chain

```
Content hash:   a3f1c9e2...        ← you generate this yourself to verify
OTS proof:      content.ots        ← verify against Bitcoin blockchain
Wayback URL:    web.archive.org/web/20260412154700/https://twitter.com/...
IPFS CID:       ipfs://QmXyz123...
Removal time:   2026-04-13 09:22 UTC
Survival:       17.3 hours
```

No single point of trust. No need to trust Inkblot. Every artifact is independently verifiable.

---

## 12. AI Classification Pipeline

### Classification taxonomy

```python
CATEGORIES = {
    "political_criticism":    "Criticism of government, politicians, or political parties",
    "police_accountability":  "Coverage of police actions, encounters, or custodial deaths",
    "religious_commentary":   "Content involving religious groups or communal tension",
    "caste_issues":           "Content about caste discrimination or reservation policy",
    "lgbtq":                  "Content about LGBTQ+ rights, identity, or discrimination",
    "protest_coverage":       "Coverage of protests, demonstrations, or public gatherings",
    "judicial_criticism":     "Criticism of courts, judges, or judicial decisions",
    "corporate_criticism":    "Criticism of specific corporations or business figures",
    "regional_conflict":      "Content about conflict zones: Kashmir, Manipur, Assam, etc.",
    "press_freedom":          "Content about media censorship or journalist targeting",
    "other":                  "Does not fit above categories"
}
```

### Classification pipeline

```
New content archived
        │
        ▼
Celery task enqueued: classify_content(content_id)
        │
        ▼
Fetch content_text from database
        │
        ▼
Build prompt with content + taxonomy + JSON output instruction
        │
        ▼
Call Claude API (claude-sonnet-4-6)
        │
        ▼
Parse JSON response
        │
        ├── confidence >= 0.65 → write to content_tags (ai-tagged)
        │
        └── 0.40 <= confidence < 0.65 → write to content_tags + add to human_review_queue
```

### Multilingual support

The classifier prompt explicitly instructs Claude to handle content in Hindi, Tamil, Telugu, Bengali, Punjabi, and other Indian languages without requiring translation. The multilingual sentence-transformers model handles embedding generation natively for these languages.

### Evaluation and accuracy tracking

A reference set of 500 manually-tagged incidents is maintained. Weekly, the AI classifier is run against this set and precision/recall metrics are computed per category. If any category drops below 0.75 F1, the prompt is revised and re-evaluated before deployment.

---

## 13. API Reference

Base URL: `https://api.inkblot.in/v1`

All responses return JSON. Timestamps are ISO 8601 UTC.

### Incidents

```
GET  /incidents
     ?platform=twitter|youtube|instagram|facebook
     ?region=UP|MH|DL|...        (Indian state code)
     ?tag=political_criticism|...
     ?from=2026-01-01
     ?to=2026-03-31
     ?status=removed|alive|restricted
     ?page=1&limit=50

GET  /incidents/{id}             Full incident detail

GET  /incidents/{id}/proof       Returns proof artifact URLs
     Response: {
       "content_hash": "a3f1c9...",
       "ots_proof_url": "https://storage.inkblot.in/proofs/...",
       "wayback_url": "https://web.archive.org/web/.../...",
       "ipfs_cid": "QmXyz..."
     }
```

### Creators

```
GET  /creators/{handle}          Creator profile + removal history
     ?platform=twitter

GET  /creators/{handle}/incidents All incidents for this creator
```

### Statistics

```
GET  /stats/summary              Total counts by platform and status
GET  /stats/by-platform          Removal counts per platform (timeseries)
GET  /stats/by-region            Removal counts per Indian state
GET  /stats/by-tag               Removal counts per topic category
GET  /stats/timeline             Daily removal counts, optional filters
```

### Search

```
GET  /search?q=adani+coal        Full-text search across archived content
GET  /search/semantic?q=...      Semantic similarity search (vector)
```

### Submission (requires API key)

```
POST /submit
     Body: { "url": "https://twitter.com/...", "notes": "optional" }
     Response: { "submitted": true, "queued_at": "..." }
```

---

## 14. Build Phases

### Phase 1 — Core Engine (Weeks 1–4)

**Goal:** A working watcher that detects removals and creates proof records.

Deliverables:
- Python scraper monitoring a list of 50 Twitter handles
- SHA-256 hashing on first content discovery
- OpenTimestamps proof generation
- Wayback Machine submission
- PostgreSQL logging of all incidents
- Celery scheduler running checks every 6 hours
- Deployed to Fly.io

**Definition of done:** A real tweet is removed, and Inkblot has a logged record with proof artifacts before the removal.

---

### Phase 2 — Full Collection + API (Weeks 5–8)

**Goal:** Multi-platform coverage and a queryable public API.

Deliverables:
- Add YouTube and Instagram scrapers
- Add government order monitoring (TRAI list)
- FastAPI REST API with full incident query endpoints
- Basic public web page listing recent incidents
- IPFS screenshot storage

**Definition of done:** A journalist can call the API and retrieve a filtered list of incidents with proof links.

---

### Phase 3 — Intelligence Layer (Weeks 9–12)

**Goal:** AI-powered classification and pattern detection.

Deliverables:
- Claude API auto-tagging pipeline
- Multilingual sentence-transformers embeddings + pgvector search
- Nightly pattern detection job with spike alerts
- Dashboard v1: live feed, India map, timeline chart
- Creator pages with full history

**Definition of done:** The dashboard shows a visual spike in removals during a recent political event, sourced entirely from automatically-tagged data.

---

### Phase 4 — Scale and Partnerships (Ongoing)

Deliverables:
- Reach out to IFF (Internet Freedom Foundation), SFLC.in, MediaNama, The Wire
- Public launch with press outreach
- Expand watchlist to 10,000+ creators
- Add Facebook scraper
- API key registration for research partners
- Creator submission portal (creators can add their own content for monitoring)
- Export to CSV/JSON for researchers

---

## 15. Target Users

| User | How They Use Inkblot | What They Need |
|---|---|---|
| **Journalists** | Verify that takedowns happened; cite proof in articles | Cryptographic proof links, fast API access |
| **Lawyers / PILs** | Evidence of systematic suppression in court | OTS proof files, exportable incident logs |
| **Civil rights orgs** (IFF, SFLC.in) | Policy briefs, annual reports on internet freedom | Statistics API, bulk data export |
| **Academic researchers** | Datasets for studying internet censorship in India | Full dataset download, semantic search |
| **Creators** | Know if their content is at risk; document their own removal history | Creator page, submission API |
| **International media** (BBC, Reuters, Al Jazeera) | Source data for India internet freedom coverage | Dashboard, embeddable charts |

---

## 16. Project Constraints and Known Hard Problems

### Constraint 1 — You can only archive what you've already seen

If content is removed before the Watcher first discovers it, no proof can be generated. Mitigation: users can submit URLs proactively via the API before content is at risk. Creator submission portal (Phase 4) addresses this more broadly.

### Constraint 2 — Platform anti-scraping is an arms race

Twitter, Instagram, and Facebook actively evolve their anti-scraping measures. Maintaining working scrapers requires ongoing engineering effort. This is expected and budgeted for — it is not a reason to use official APIs exclusively, because official APIs often don't expose the data needed.

### Constraint 3 — Classification accuracy affects credibility

A false positive — labeling a spam removal as political suppression — is worse than a missed detection. The confidence threshold (0.65) and human review queue exist to protect against this. The evaluation reference set must be maintained actively.

### Constraint 4 — Scale of the internet vs. budget

At Tier 1 (hourly checks), 500 creators with average 20 pieces of tracked content each = 10,000 URL checks per hour. At Tier 2 (6-hourly), 5,000 creators = 25,000 checks per 6 hours. Playwright-based checks are slow (~3–8 seconds each). Worker autoscaling and intelligent batching are essential to keep infrastructure costs manageable.

### Constraint 5 — Legal exposure if targeted

If the Indian government views Inkblot as a threat, they could pressure hosting providers, pursue the operators legally, or attempt to block access to the dashboard. Mitigations: host outside India, maintain anonymity of operators initially, partner with international press freedom organizations for cover, keep the tool framed as journalism infrastructure.

---

## Repository Structure

```
inkblot/
├── README.md
├── SYSTEM_ARCHITECTURE.md          ← this file
├── .env.example
├── docker-compose.yml              ← local development
│
├── backend/
│   ├── app/
│   │   ├── main.py                 ← FastAPI app entry point
│   │   ├── config.py               ← settings from environment
│   │   ├── database.py             ← SQLAlchemy async engine + session
│   │   │
│   │   ├── models/                 ← SQLAlchemy ORM models
│   │   │   ├── content.py
│   │   │   ├── creator.py
│   │   │   ├── removal.py
│   │   │   └── tags.py
│   │   │
│   │   ├── api/                    ← FastAPI routers
│   │   │   ├── incidents.py
│   │   │   ├── creators.py
│   │   │   ├── stats.py
│   │   │   ├── search.py
│   │   │   └── submit.py
│   │   │
│   │   ├── watcher/                ← scraping and monitoring
│   │   │   ├── scheduler.py        ← Celery beat schedule
│   │   │   ├── tasks.py            ← Celery task definitions
│   │   │   ├── scrapers/
│   │   │   │   ├── base.py
│   │   │   │   ├── twitter.py
│   │   │   │   ├── youtube.py
│   │   │   │   └── instagram.py
│   │   │   └── detector.py         ← removal detection logic
│   │   │
│   │   ├── archive/                ← proof generation
│   │   │   ├── hasher.py           ← SHA-256
│   │   │   ├── timestamps.py       ← OpenTimestamps
│   │   │   ├── wayback.py          ← Wayback Machine API
│   │   │   └── ipfs.py             ← IPFS via Pinata
│   │   │
│   │   └── intelligence/           ← AI and pattern detection
│   │       ├── classifier.py       ← Claude API tagging
│   │       ├── embedder.py         ← sentence-transformers
│   │       └── patterns.py         ← spike detection, correlation
│   │
│   ├── migrations/                 ← Alembic database migrations
│   ├── tests/
│   └── requirements.txt
│
├── frontend/
│   ├── src/
│   │   ├── pages/
│   │   ├── components/
│   │   └── api/                    ← API client
│   └── package.json
│
└── infrastructure/
    ├── fly.toml                    ← Fly.io deployment config
    └── github-actions/
        └── deploy.yml
```

---

*Last updated: May 2026*
*Status: Architecture v1.1 — pre-implementation - updated*
