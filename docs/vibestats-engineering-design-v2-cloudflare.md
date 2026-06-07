# VibeStats Engineering Design v2 (Cloudflare Stack)

**Status:** Draft  
**Date:** 2026-06-07  
**Version:** 2.0 (Cloudflare-native)  
**Linked PRD:** `docs/vibe-stats-prd-v2.md`

---

## 1. System Overview

### 1.1 Architecture Diagram

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│  Cloudflare     │────▶│  Cloudflare      │────▶│  Cloudflare D1  │
│  Pages          │◀────│  Workers         │◀────│  (SQLite)       │
│  (Next.js)      │     │  (Hono API)      │     │                 │
└─────────────────┘     └──────────────────┘     └─────────────────┘
                               │
                               ▼
                        ┌──────────────────┐
                        │  Cloudflare KV   │
                        │  (Cache)         │
                        └──────────────────┘
                               │
                               ▼
                        ┌──────────────────┐
                        │  Cloudflare R2   │
                        │  (Raw payloads)  │
                        └──────────────────┘
                               │
                               ▼
                        ┌──────────────────┐
                        │  GitHub API      │
                        │  (GraphQL+REST)  │
                        └──────────────────┘
```

### 1.2 Component Responsibilities

| Component | Responsibility | Tech Choice |
|-----------|----------------|-------------|
| Frontend | Report pages, input form, charts | Next.js 14 on Cloudflare Pages |
| API Server | Scan orchestration, data aggregation | Hono on Cloudflare Workers |
| Database | Persistent storage, job queue | Cloudflare D1 (SQLite) |
| Cache | API response cache, report cache | Cloudflare KV |
| Object Storage | Raw API payloads, artifacts | Cloudflare R2 |
| Background Jobs | Scan worker | Cloudflare Queues + Cron Triggers |
| GitHub Integration | Data ingestion | GraphQL (primary) + REST (fallback) |
| Domain/DNS | Domain registration, DNS | Cloudflare Registrar + DNS |
| Email | Dispute notifications | Cloudflare Email Routing |

### 1.3 Why Cloudflare Stack

**Advantages:**
- **Unified platform:** Single dashboard, single bill
- **Higher free tiers:**
  - D1: 5GB storage, 5M reads/day, 100K writes/day
  - KV: 100K reads/day, 1K writes/day free
  - Workers: 100K requests/day free
  - R2: 10GB storage, 10M reads/month free
  - Queues: 1M operations/month free
- **No cold starts:** Workers are always warm at the edge
- **Global edge:** Low latency worldwide
- **No egress fees:** R2 has no egress charges

**Trade-offs:**
- **D1 is SQLite:** Not PostgreSQL — schema adjustments needed (no UUID native, use TEXT)
- **Worker timeout:** 10s (free), 30s (paid) — long scans must be async via Queues
- **KV limitations:** No Redis-style TTL expiration, no pub/sub
- **No native WebSocket:** Need to use Cloudflare Tunnel or external service

### 1.4 Deployment Topology

- **Frontend + API:** Cloudflare Pages (Next.js SSR via Functions)
- **Backend Workers:** Cloudflare Workers (Hono)
- **Database:** Cloudflare D1 (global replicated SQLite)
- **Cache:** Cloudflare KV (edge-cached)
- **Object Storage:** Cloudflare R2
- **Queue:** Cloudflare Queues + Cron Triggers (every minute)
- **Domain:** `vibestats.io` via Cloudflare Registrar
- **Email:** `hello@vibestats.io` via Cloudflare Email Routing → personal email

---

## 2. Data Model (D1/SQLite)

### 2.1 Core Tables

**Note:** D1 uses SQLite. Key differences from PostgreSQL:
- No native `UUID` type → use `TEXT` with `lower(hex(randomblob(16)))`
- No `TIMESTAMPTZ` → use `TEXT` with ISO 8601 strings
- No `JSONB` → use `TEXT` with `json()` function
- No `ON DELETE CASCADE` in all cases → handle in application logic
- No `GENERATED ALWAYS` → compute in application

#### `repositories`

```sql
CREATE TABLE repositories (
  id TEXT PRIMARY KEY, -- UUID as TEXT
  owner TEXT NOT NULL,
  name TEXT NOT NULL,
  default_branch TEXT,
  created_at TEXT, -- ISO 8601
  first_scanned_at TEXT,
  last_scanned_at TEXT,
  stars_current INTEGER,
  forks_current INTEGER,
  primary_language TEXT,
  UNIQUE(owner, name)
);

CREATE INDEX idx_repositories_owner_name ON repositories(owner, name);
CREATE INDEX idx_repositories_last_scanned ON repositories(last_scanned_at DESC);
```

**UUID Generation (in Worker):**
```typescript
function generateUUID(): string {
  return crypto.randomUUID(); // Workers support this
}
```

#### `weekly_metrics`

```sql
CREATE TABLE weekly_metrics (
  repo_id TEXT NOT NULL,
  week_start TEXT NOT NULL, -- ISO 8601 date (Monday)
  commits INTEGER DEFAULT 0,
  prs_opened INTEGER DEFAULT 0,
  prs_merged INTEGER DEFAULT 0,
  issues_opened INTEGER DEFAULT 0,
  issues_closed INTEGER DEFAULT 0,
  releases INTEGER DEFAULT 0,
  active_contributors INTEGER DEFAULT 0,
  PRIMARY KEY (repo_id, week_start),
  FOREIGN KEY (repo_id) REFERENCES repositories(id)
);

CREATE INDEX idx_weekly_metrics_repo_week ON weekly_metrics(repo_id, week_start DESC);
```

#### `daily_snapshots`

```sql
CREATE TABLE daily_snapshots (
  repo_id TEXT NOT NULL,
  observed_at TEXT NOT NULL, -- ISO 8601 date
  stars INTEGER,
  forks INTEGER,
  open_issues INTEGER,
  acceleration_score REAL,
  leverage_score REAL,
  confidence_level TEXT,
  PRIMARY KEY (repo_id, observed_at),
  FOREIGN KEY (repo_id) REFERENCES repositories(id)
);

CREATE INDEX idx_daily_snapshots_repo_date ON daily_snapshots(repo_id, observed_at DESC);
```

#### `scan_jobs`

```sql
CREATE TABLE scan_jobs (
  id TEXT PRIMARY KEY,
  repo_id TEXT NOT NULL,
  status TEXT NOT NULL, -- pending, running, completed, failed
  created_at TEXT DEFAULT (datetime('now')),
  started_at TEXT,
  completed_at TEXT,
  error_message TEXT,
  api_calls_used INTEGER DEFAULT 0,
  cache_hit INTEGER DEFAULT 0, -- SQLite uses INTEGER for boolean
  retry_count INTEGER DEFAULT 0,
  next_retry_at TEXT,
  progress_json TEXT, -- JSON string: {"stage": "...", "percent": 45}
  FOREIGN KEY (repo_id) REFERENCES repositories(id)
);

CREATE INDEX idx_scan_jobs_status ON scan_jobs(status);
CREATE INDEX idx_scan_jobs_repo ON scan_jobs(repo_id);
CREATE INDEX idx_scan_jobs_next_retry ON scan_jobs(next_retry_at) WHERE status = 'pending';
```

**Progress JSON Schema:**
```typescript
{
  stage: "validate" | "check_cache" | "fetch_metadata" | "fetch_timeline" | "aggregate" | "complete",
  percent_complete: number,
  current_entity?: "commits" | "prs" | "issues" | "releases"
}
```

#### `scan_cache` (KV Alternative)

**Decision:** Use **Cloudflare KV** instead of D1 table for cache.

**Rationale:**
- KV is edge-cached (faster than D1 for simple key-value)
- Built-in TTL support
- Cheaper for high-frequency reads

**KV Namespace:** `SCAN_CACHE`

**Key Format:**
- `github:graphql:{query_hash}`
- `github:rest:{method}:{path}:{params_hash}`

**TTL:** 86400 seconds (24 hours)

**Code Example:**
```typescript
// Write
await SCAN_CACHE.put(key, JSON.stringify(value), { expirationTtl: 86400 });

// Read
const cached = await SCAN_CACHE.get(key, 'json');
```

#### `disputes`

```sql
CREATE TABLE disputes (
  id TEXT PRIMARY KEY,
  repo_id TEXT NOT NULL,
  issue_type TEXT NOT NULL, -- incorrect_data, missing_data, unfair_interpretation, takedown_request
  description TEXT NOT NULL,
  evidence_url TEXT,
  contact_email TEXT NOT NULL,
  status TEXT NOT NULL DEFAULT 'submitted', -- submitted, under_review, resolved, closed
  created_at TEXT DEFAULT (datetime('now')),
  reviewed_at TEXT,
  reviewed_by TEXT,
  resolution_notes TEXT,
  resolved_at TEXT,
  FOREIGN KEY (repo_id) REFERENCES repositories(id)
);

CREATE INDEX idx_disputes_status ON disputes(status);
CREATE INDEX idx_disputes_repo ON disputes(repo_id);
CREATE INDEX idx_disputes_created ON disputes(created_at DESC);
```

### 2.2 Derived Metrics (Computed at Query Time)

Same formulas as PRD v2, implemented in Worker TypeScript code.

---

## 3. API Design (Hono on Workers)

### 3.1 Project Structure

```
/apps
  /web          # Next.js frontend (Cloudflare Pages)
  /api          # Hono API (Cloudflare Workers)
/packages
  /shared       # Shared types, utilities
  /scanner      # GitHub API scanner library
/db
  /migrations   # D1 migration SQL files
  /schema.ts    # Drizzle schema (optional, or use raw SQL)
/docs           # PRD, design, methodology
```

### 3.2 Public Endpoints

All endpoints implemented in Hono on Workers.

#### `POST /api/scans`

Same spec as v1. Implementation uses D1 + Queues.

#### `GET /api/scans/:id`

Same spec as v1. Polls D1 for job status.

#### `GET /api/repos/:owner/:repo`

Same spec as v1. Fetches from D1.

#### `GET /api/repos/:owner/:repo/timeline`

Same spec as v1. Queries D1 `weekly_metrics`.

#### `GET /api/examples`

Hardcoded list in Worker.

#### `POST /api/disputes`

Inserts into D1 `disputes` table, triggers Email Routing notification.

---

## 4. GitHub API Strategy

Same as v1 (GraphQL primary, REST fallback).

**Rate Limit Management:**
- Throttle: 100 req/min
- Backoff: Exponential on 429
- Cache: KV with 24h TTL

**Implementation in Workers:**
```typescript
// Use Workers' built-in fetch with caching
async function cachedFetch(url: string, options: RequestInit) {
  const key = `github:${url}`;
  const cached = await SCAN_CACHE.get(key, 'json');
  if (cached) return cached;

  const response = await fetch(url, options);
  if (response.ok) {
    const data = await response.json();
    await SCAN_CACHE.put(key, JSON.stringify(data), { expirationTtl: 86400 });
    return data;
  }
  throw new Error(`GitHub API error: ${response.status}`);
}
```

---

## 5. Background Job Execution (Queues + Cron)

### 5.1 Architecture

```
Cloudflare Cron (every minute)
       ↓
Cloudflare Worker (scan-worker)
       ↓
Cloudflare Queues (scan-jobs queue)
       ↓
Worker Consumer (process scans, update D1)
```

### 5.2 Cron Trigger Configuration

**File:** `wrangler.toml`

```toml
[triggers]
crons = ["* * * * *"]  # Every minute
```

### 5.3 Queue Configuration

**File:** `wrangler.toml`

```toml
[[queues.producers]]
queue = "scan-jobs"
binding = "SCAN_QUEUE"

[[queues.consumers]]
queue = "scan-jobs"
max_batch_size = 10
max_retries = 3
dead_letter_queue = "scan-jobs-dlq"
```

### 5.4 Worker Code (Simplified)

```typescript
// cron trigger
export default {
  async scheduled(event, env, ctx) {
    // Poll D1 for pending jobs
    const pending = await env.DB.prepare(
      'SELECT * FROM scan_jobs WHERE status = ? ORDER BY created_at LIMIT 10'
    ).bind('pending').all();

    // Send to queue
    const messages = pending.results.map(job => ({
      body: JSON.stringify({ jobId: job.id })
    }));
    await env.SCAN_QUEUE.sendBatch(messages);
  }
};

// queue consumer
export default {
  async queue(batch, env, ctx) {
    for (const message of batch.messages) {
      const { jobId } = JSON.parse(message.body);
      await processScanJob(env, jobId);
    }
  }
};
```

### 5.5 Timeout Handling

**Problem:** Workers have 10s timeout (free) / 30s (paid).

**Solution:**
- Break scan into multiple queue messages (one per entity: commits, PRs, issues, releases)
- Each message processes independently
- Aggregate at the end

**Alternative:** Use a dedicated VPS (Fly.io, Railway) for heavy scans if needed.

---

## 6. Frontend (Next.js on Pages)

### 6.1 Setup

**File:** `pages/web/next.config.js`

```javascript
/** @type {import('next').NextConfig} */
const nextConfig = {
  output: 'export', // Static export for Pages
  images: { unoptimized: true }, // No Image Optimization on Pages
};

module.exports = nextConfig;
```

**Deployment:**
```bash
npx wrangler pages deploy ./out --project-name=vibestats
```

### 6.2 API Calls

Frontend calls Workers endpoints directly:
```typescript
const res = await fetch('https://vibestats.io/api/repos/' + repo);
```

**CORS:** Configure in Worker:
```typescript
app.use('*', cors());
```

---

## 7. Domain, DNS, Email

### 7.1 Domain Registration

**Action:** Register `vibestats.io` via Cloudflare Registrar.

**Cost:** ~$10/year for `.io` (at-cost pricing).

### 7.2 DNS Configuration

**Records:**
```
Type    Name              Content                    Proxy
----    ----              -------                    -----
A       vibestats.io      76.76.21.21 (Vercel IP)   No
CNAME   www               vibestats.io               No
TXT     _github-challenge xyz                        No (for OAuth)
MX      @                 mx1.emailserver.com        Yes (if using Email Routing)
```

**Note:** If using Cloudflare Pages, DNS is auto-configured.

### 7.3 Email Routing

**Setup:**
1. Enable Email Routing in Cloudflare Dashboard
2. Create rule: `hello@vibestats.io` → `[your-personal-email]`
3. Verify destination address

**Cost:** Free.

---

## 8. Environment Variables

### Required

| Variable | Description | Example |
|----------|-------------|---------|
| `GITHUB_TOKEN` | GitHub OAuth token | `ghp_xxxx` |
| `D1_DATABASE_ID` | D1 database UUID | `xxxx-xxxx-xxxx` |
| `KV_NAMESPACE_ID` | KV namespace ID | `xxxx` |
| `R2_BUCKET_NAME` | R2 bucket name | `vibestats-cache` |
| `R2_ACCESS_KEY_ID` | R2 access key | `xxxx` |
| `R2_SECRET_ACCESS_KEY` | R2 secret key | `xxxx` |
| `NODE_ENV` | Environment | `production` |

### Wrangler Configuration

**File:** `wrangler.toml`

```toml
name = "vibestats-api"
main = "apps/api/src/index.ts"
compatibility_date = "2026-06-01"

[[d1_databases]]
binding = "DB"
database_name = "vibestats"
database_id = "xxxx-xxxx-xxxx"

[[kv_namespaces]]
binding = "SCAN_CACHE"
id = "xxxx"

[[r2_buckets]]
binding = "R2_STORE"
bucket_name = "vibestats-cache"

[[queues.producers]]
queue = "scan-jobs"
binding = "SCAN_QUEUE"

[[queues.consumers]]
queue = "scan-jobs"
max_batch_size = 10

[triggers]
crons = ["* * * * *"]

[vars]
GITHUB_TOKEN = "ghp_xxxx" # Use secrets, not vars!
```

**Secrets (not in wrangler.toml):**
```bash
wrangler secret put GITHUB_TOKEN
wrangler secret put R2_ACCESS_KEY_ID
wrangler secret put R2_SECRET_ACCESS_KEY
```

---

## 9. Cost Estimates (Cloudflare Stack)

| Service | Free Tier | MVP Usage | Monthly Cost |
|---------|-----------|-----------|--------------|
| Pages | 100K requests/day | 10K page views | $0 |
| Workers | 100K requests/day | 1K scans × 10 calls = 10K | $0 |
| D1 | 5GB storage, 5M reads/day | 1GB, 100K reads | $0 |
| KV | 100K reads/day | 50K reads | $0 |
| R2 | 10GB storage, 10M reads/month | 1GB, 100K reads | $0 |
| Queues | 1M operations/month | 10K jobs | $0 |
| Domain | - | `vibestats.io` | ~$10/year |
| **Total** | | | **~$1/month** (just domain) |

**Assumptions:**
- 1,000 scans/month
- 10,000 report page views/month
- All within free tiers

**If growth exceeds free tiers:**
- Workers Paid: $5/month for 10M requests
- D1 Paid: $0.75/GB storage, $0.50/M reads
- Still significantly cheaper than Vercel + Supabase + Upstash combo

---

## 10. Migration from PostgreSQL to SQLite

### Key Changes

| PostgreSQL | SQLite (D1) | Migration Note |
|------------|-------------|----------------|
| `UUID` | `TEXT` | Generate UUID in app code |
| `TIMESTAMPTZ` | `TEXT` | Store ISO 8601 strings |
| `JSONB` | `TEXT` | Use `json()` function for queries |
| `BOOLEAN` | `INTEGER` | 0/1 instead of true/false |
| `ON DELETE CASCADE` | Limited support | Handle in app logic |
| `RETURNING` clause | Supported | Works in D1 |
| `GENERATED ALWAYS` | Not supported | Compute in app |

### Migration Commands

```bash
# Create D1 database
wrangler d1 create vibestats

# Run migrations
wrangler d1 execute vibestats --file=db/migrations/0001_initial.sql

# Query locally
wrangler d1 execute vibestats --command="SELECT * FROM repositories LIMIT 5"
```

---

## 11. Open Technical Decisions

### 11.1 Worker Timeout for Scans

**Problem:** 10s timeout insufficient for large repo scans.

**Options:**
1. **Async via Queues (chosen):** Break scan into multiple messages, each <10s
2. **Use longer timeout (paid):** $5/month for 30s timeout
3. **External worker:** Fly.io/Railway for heavy scans

**Decision:** Start with Queues (option 1). Upgrade to paid if needed.

### 11.2 Drizzle ORM vs Raw SQL

**Options:**
1. **Drizzle ORM:** Type-safe, migrations, but adds complexity
2. **Raw SQL:** Simpler, full control, but more boilerplate

**Decision:** Use **raw SQL** for MVP (simpler for D1). Add Drizzle in V1 if needed.

### 11.3 Next.js on Pages vs Workers

**Options:**
1. **Pages Functions:** Easier deployment, but limited to 10s timeout
2. **Workers + Hono:** More control, but manual routing

**Decision:** Use **Pages Functions** for Next.js API routes, separate **Workers** for scan jobs.

---

## 12. Data Retention Policy

Same as v1:
- Raw API cache (KV): 24 hours (automatic TTL)
- Aggregated metrics (D1): Indefinite
- Daily snapshots (D1): 12 months
- Scan job history (D1): 90 days
- Dispute records (D1): 2 years

---

## 13. Timezone Handling

**All date aggregations use UTC.**

- `week_start`: Monday 00:00:00 UTC (stored as ISO 8601 string)
- `observed_at`: Midnight UTC
- Scan timestamps: ISO 8601 with timezone

SQLite doesn't have native timezone handling — all conversions done in TypeScript.

---

## 14. Monitoring & Observability

### 14.1 Cloudflare Analytics

- **Workers Analytics Engine:** Track custom metrics (scans, errors, duration)
- **D1 Dashboard:** Monitor query performance
- **KV Metrics:** Cache hit rate

### 14.2 Error Tracking

**Option A:** Cloudflare Logpush → Splunk/Datadog (paid)  
**Option B:** Sentry (free tier)  
**Option C:** Simple error logging to R2 (DIY)

**Decision:** Use **Sentry** for MVP (free up to 5K errors/month).

---

## 15. Next Steps

1. **Register domain** via Cloudflare Registrar
2. **Create D1 database** + run migrations
3. **Set up KV namespace** + R2 bucket
4. **Configure Wrangler** + deploy hello world Worker
5. **Implement CLI scanner** (Phase 1 prototype)
6. **Build API endpoints** on Workers
7. **Deploy Next.js** to Pages
8. **Test end-to-end** with example repos

---

**End of Engineering Design v2 (Cloudflare)**
