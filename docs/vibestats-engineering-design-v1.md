# VibeStats Engineering Design v1

**Status:** Draft  
**Date:** 2026-06-07  
**Version:** 1.0  
**Linked PRD:** `docs/vibe-stats-prd-v2.md`

---

## 1. System Overview

### 1.1 Architecture Diagram

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│   Next.js UI    │────▶│  Hono API Server │────▶│   PostgreSQL    │
│   (Vercel)      │◀────│  (Node.js 20)    │◀────│   (Supabase)    │
└─────────────────┘     └──────────────────┘     └─────────────────┘
                               │
                               ▼
                        ┌──────────────────┐
                        │   Redis Cache    │
                        │   (Upstash)      │
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
| Frontend | Report pages, input form, charts | Next.js 14 App Router |
| API Server | Scan orchestration, data aggregation | Hono on Node.js 20 |
| Database | Persistent storage, job queue | PostgreSQL (Supabase) |
| Cache | API response cache, report cache | Redis (Upstash) |
| Workers | Background scan jobs | Node.js + `setInterval` polling |
| GitHub Integration | Data ingestion | GraphQL (primary) + REST (fallback) |

### 1.3 Deployment Topology

- **Frontend + API:** Vercel (serverless functions for API routes)
- **Database:** Supabase (free tier for MVP)
- **Cache:** Upstash Redis (free tier for MVP)
- **Background Jobs:** Vercel cron triggers or GitHub Actions scheduled workflow
- **Domain:** `vibestats.io` (TBD)

---

## 2. Data Model

### 2.1 Core Tables

#### `repositories`

```sql
CREATE TABLE repositories (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  owner TEXT NOT NULL,
  name TEXT NOT NULL,
  default_branch TEXT,
  created_at TIMESTAMPTZ,
  first_scanned_at TIMESTAMPTZ,
  last_scanned_at TIMESTAMPTZ,
  stars_current INTEGER,
  forks_current INTEGER,
  primary_language TEXT,
  UNIQUE(owner, name)
);

CREATE INDEX idx_repositories_owner_name ON repositories(owner, name);
CREATE INDEX idx_repositories_last_scanned ON repositories(last_scanned_at DESC);
```

**Purpose:** Master record for each scanned repository.

**Fields:**
- `owner`, `name`: GitHub owner/repo (e.g., `vercel`, `next.js`)
- `default_branch`: From GitHub API
- `created_at`: Repo creation date (from GitHub)
- `first_scanned_at`: First time VibeStats scanned this repo
- `last_scanned_at`: Most recent scan completion
- `stars_current`, `forks_current`: Latest observed values
- `primary_language`: Top language from GitHub metadata

#### `weekly_metrics`

```sql
CREATE TABLE weekly_metrics (
  repo_id UUID REFERENCES repositories(id) ON DELETE CASCADE,
  week_start DATE NOT NULL,
  commits INTEGER DEFAULT 0,
  prs_opened INTEGER DEFAULT 0,
  prs_merged INTEGER DEFAULT 0,
  issues_opened INTEGER DEFAULT 0,
  issues_closed INTEGER DEFAULT 0,
  releases INTEGER DEFAULT 0,
  active_contributors INTEGER DEFAULT 0,
  PRIMARY KEY (repo_id, week_start)
);

CREATE INDEX idx_weekly_metrics_repo_week ON weekly_metrics(repo_id, week_start DESC);
```

**Purpose:** Aggregated weekly activity metrics.

**Aggregation Logic:**
- `week_start`: Monday 00:00:00 UTC
- `commits`: Count of commits with `committed_date` in week
- `prs_opened`: Count of PRs with `created_at` in week
- `prs_merged`: Count of PRs with `merged_at` in week
- `issues_opened`: Count of issues with `created_at` in week
- `issues_closed`: Count of issues with `closed_at` in week
- `releases`: Count of releases with `published_at` in week
- `active_contributors`: Count of unique commit/PR authors (excluding bots)

**Bot Exclusion List:**
- `dependabot[bot]`
- `renovate[bot]`
- `github-actions[bot]`
- `pre-commit-ci[bot]`
- `allcontributors[bot]`

#### `daily_snapshots`

```sql
CREATE TABLE daily_snapshots (
  repo_id UUID REFERENCES repositories(id) ON DELETE CASCADE,
  observed_at DATE NOT NULL,
  stars INTEGER,
  forks INTEGER,
  open_issues INTEGER,
  acceleration_score DECIMAL,
  leverage_score DECIMAL,
  confidence_level TEXT,
  PRIMARY KEY (repo_id, observed_at)
);

CREATE INDEX idx_daily_snapshots_repo_date ON daily_snapshots(repo_id, observed_at DESC);
```

**Purpose:** Observed point-in-time snapshots (not reconstructed).

**When Created:**
- After each successful scan
- Daily scheduled snapshot for active repos (V1)

#### `scan_jobs`

```sql
CREATE TABLE scan_jobs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  repo_id UUID REFERENCES repositories(id) ON DELETE CASCADE,
  status TEXT NOT NULL, -- pending, running, completed, failed
  created_at TIMESTAMPTZ DEFAULT NOW(),
  started_at TIMESTAMPTZ,
  completed_at TIMESTAMPTZ,
  error_message TEXT,
  api_calls_used INTEGER DEFAULT 0,
  cache_hit BOOLEAN DEFAULT FALSE,
  retry_count INTEGER DEFAULT 0,
  next_retry_at TIMESTAMPTZ
);

CREATE INDEX idx_scan_jobs_status ON scan_jobs(status);
CREATE INDEX idx_scan_jobs_repo ON scan_jobs(repo_id);
CREATE INDEX idx_scan_jobs_next_retry ON scan_jobs(next_retry_at) WHERE status = 'pending';
```

**Purpose:** Track scan job lifecycle, retries, and errors.

**State Machine:**
```
pending → running → completed
              ↓
            failed → pending (retry)
```

**Retry Logic:**
- Max 3 retries
- Exponential backoff: 1min, 5min, 30min
- `next_retry_at` indexed for efficient polling

#### `scan_cache`

```sql
CREATE TABLE scan_cache (
  key TEXT PRIMARY KEY,
  value JSONB NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  expires_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_scan_cache_expires ON scan_cache(expires_at);
CREATE INDEX idx_scan_cache_key_prefix ON scan_cache(key text_pattern_ops);
```

**Purpose:** Cache raw GitHub API responses (24h TTL).

**Cache Key Format:**
- `github:graphql:{query_hash}`
- `github:rest:{method}:{path}:{query_params_hash}`

**Invalidation:**
- Automatic on expiry
- Manual invalidation on schema change

### 2.2 Derived Metrics (Computed at Query Time)

#### Acceleration Score

```sql
-- Pseudo-SQL for acceleration calculation
WITH recent_velocity AS (
  SELECT SUM(commits + prs_merged) as velocity
  FROM weekly_metrics
  WHERE repo_id = $1
    AND week_start >= CURRENT_DATE - INTERVAL '30 days'
),
baseline_velocity AS (
  SELECT MEDIAN(commits + prs_merged) as baseline
  FROM weekly_metrics
  WHERE repo_id = $1
    AND week_start BETWEEN CURRENT_DATE - INTERVAL '120 days'
                       AND CURRENT_DATE - INTERVAL '30 days'
)
SELECT 
  CASE 
    WHEN baseline_velocity.baseline > 0 
    THEN recent_velocity.velocity / baseline_velocity.baseline
    ELSE NULL
  END as acceleration_multiplier
FROM recent_velocity, baseline_velocity;
```

#### Human Leverage

```sql
-- Computed from weekly_metrics
SELECT 
  week_start,
  commits / NULLIF(active_contributors, 0) as commits_per_contributor
FROM weekly_metrics
WHERE repo_id = $1;
```

#### Confidence Level

```sql
-- Logic implemented in application layer
CASE
  WHEN history_months > 12 
       AND total_data_points > 100 
       AND missing_pct < 0.10 
  THEN 'High'
  
  WHEN history_months > 6 
       OR total_data_points > 50 
       OR missing_pct < 0.30 
  THEN 'Medium'
  
  WHEN history_months < 6 
       AND total_data_points < 50 
       AND missing_pct > 0.30 
  THEN 'Low'
  
  ELSE 'Unavailable'
END
```

---

## 3. API Design

### 3.1 Public Endpoints

#### `POST /api/scans`

**Purpose:** Trigger a new scan job.

**Request:**
```json
{
  "repo": "vercel/next.js"
}
```

**Response (202 Accepted):**
```json
{
  "job_id": "uuid",
  "status": "pending",
  "estimated_duration_seconds": 180,
  "poll_url": "/api/scans/{job_id}"
}
```

**Behavior:**
1. Validate repo exists on GitHub (lightweight HEAD request)
2. Check cache for recent scan (<24h)
3. Create `scan_jobs` record
4. Return immediately (async processing)

**Errors:**
- `400 Bad Request`: Invalid repo format
- `404 Not Found`: Repo doesn't exist on GitHub
- `429 Too Many Requests`: Rate limit exceeded

---

#### `GET /api/scans/:id`

**Purpose:** Poll scan job status.

**Response (200 OK, job in progress):**
```json
{
  "job_id": "uuid",
  "status": "running",
  "progress": {
    "stage": "fetching_commits",
    "percent_complete": 45
  },
  "estimated_remaining_seconds": 90
}
```

**Response (200 OK, job completed):**
```json
{
  "job_id": "uuid",
  "status": "completed",
  "completed_at": "2026-06-07T12:34:56Z",
  "result": {
    "repo_url": "/api/repos/vercel/next.js",
    "timeline_url": "/api/repos/vercel/next.js/timeline"
  }
}
```

**Response (200 OK, job failed):**
```json
{
  "job_id": "uuid",
  "status": "failed",
  "error": "GitHub API rate limit exceeded",
  "retry_available": true,
  "retry_at": "2026-06-07T12:40:00Z"
}
```

---

#### `GET /api/repos/:owner/:repo`

**Purpose:** Get repo summary and latest metrics.

**Response (200 OK):**
```json
{
  "repository": {
    "owner": "vercel",
    "name": "next.js",
    "description": "The React Framework",
    "default_branch": "main",
    "created_at": "2016-10-05T00:00:00Z",
    "stars": 123456,
    "forks": 23456,
    "primary_language": "JavaScript"
  },
  "summary": {
    "acceleration": {
      "multiplier": 3.1,
      "comparison": "last 30d vs prior 90d",
      "interpretation": "Activity increased 3.1x over baseline"
    },
    "leverage": {
      "active_contributors_30d": 2,
      "commits_per_contributor": 75,
      "top_contributor_share": 0.80
    },
    "confidence": {
      "level": "Medium",
      "reason": "9 months history, 85% data completeness"
    }
  },
  "last_scan": {
    "completed_at": "2026-06-07T12:34:56Z",
    "scan_job_id": "uuid"
  }
}
```

**Cache TTL:** 1 hour

---

#### `GET /api/repos/:owner/:repo/timeline`

**Purpose:** Get weekly historical timeline.

**Query Params:**
- `window`: `30d`, `90d`, `12mo`, `all` (default: `90d`)

**Response (200 OK):**
```json
{
  "repo": "vercel/next.js",
  "window": "90d",
  "data_source": "reconstructed",
  "confidence": "High",
  "weeks": [
    {
      "week_start": "2026-03-02",
      "commits": 45,
      "prs_opened": 12,
      "prs_merged": 10,
      "issues_opened": 8,
      "issues_closed": 7,
      "releases": 2,
      "active_contributors": 5
    },
    ...
  ]
}
```

**Cache TTL:** 1 hour

---

#### `GET /api/examples`

**Purpose:** Get hardcoded list of example reports.

**Response (200 OK):**
```json
{
  "examples": [
    {
      "repo": "vercel/next.js",
      "label": "Mature project with sustained velocity",
      "report_url": "/repos/vercel/next.js"
    },
    {
      "repo": "anthropics/claude-code",
      "label": "Fast-accelerating new project",
      "report_url": "/repos/anthropics/claude-code"
    },
    ...
  ]
}
```

**Note:** Hardcoded in code, not dynamic.

---

#### `POST /api/disputes`

**Purpose:** Maintainer dispute/correction request.

**Request:**
```json
{
  "repo": "vercel/next.js",
  "issue_type": "incorrect_data",
  "description": "Commit count is missing squash merges from main branch",
  "evidence_url": "https://github.com/vercel/next.js/issues/...",
  "contact_email": "maintainer@example.com"
}
```

**Response (202 Accepted):**
```json
{
  "ticket_id": "uuid",
  "status": "submitted",
  "expected_response_hours": 48
}
```

**Backend Behavior:**
- Create record in `disputes` table (not shown in schema above)
- Send email notification to VibeStats team
- Auto-reply to submitter with ticket ID

---

### 3.2 Internal Worker Endpoints (Not Public)

#### `POST /internal/workers/scan`

**Purpose:** Triggered by cron to process pending scan jobs.

**Auth:** Bearer token (internal only)

**Behavior:**
1. Poll `scan_jobs` for `status = 'pending'`
2. Process up to 10 jobs concurrently
3. Update job status and metrics

---

## 4. GitHub API Strategy

### 4.1 GraphQL Primary Queries

#### Repository Metadata + Commits (Single Query)

```graphql
query RepoTimeline($owner: String!, $name: String!, $since: GitTimestamp!) {
  repository(owner: $owner, name: $name) {
    id
    createdAt
    pushedAt
    stargazerCount
    forkCount
    defaultBranchRef {
      name
    }
    primaryLanguage {
      name
    }
    commits: object(expression: "HEAD") {
      ... on Commit {
        history(since: $since, first: 100) {
          totalCount
          nodes {
            committedDate
            author {
              user {
                login
              }
            }
          }
          pageInfo {
            hasNextPage
            endCursor
          }
        }
      }
    }
    pullRequests(states: MERGED, orderBy: {field: UPDATED_AT, direction: DESC}, first: 100, since: $since) {
      totalCount
      nodes {
        createdAt
        mergedAt
        author {
          login
        }
      }
      pageInfo {
        hasNextPage
        endCursor
      }
    }
    issues(states: [OPEN, CLOSED], orderBy: {field: CREATED_AT, direction: DESC}, first: 100, since: $since) {
      totalCount
      nodes {
        createdAt
        closedAt
        state
      }
      pageInfo {
        hasNextPage
        endCursor
      }
    }
    releases(first: 100, orderBy: {field: CREATED_AT, direction: DESC}, since: $since) {
      totalCount
      nodes {
        publishedAt
      }
      pageInfo {
        hasNextPage
        endCursor
      }
    }
  }
}
```

**Pagination Strategy:**
- Use `endCursor` for subsequent pages
- Parallel fetch commits, PRs, issues, releases
- Stop after 30 pages per entity (safe limit for sync scan)

**Rate Limit Cost:** ~10-20 points per query (vs 100+ for equivalent REST calls)

---

### 4.2 REST Fallback Endpoints

Used when GraphQL times out or fails.

#### Commits

```
GET /repos/{owner}/{repo}/commits?since={since}&per_page=100&page={page}
```

#### Pull Requests

```
GET /repos/{owner}/{repo}/pulls?state=all&sort=updated&direction=desc&per_page=100&page={page}
```

#### Issues

```
GET /repos/{owner}/{repo}/issues?state=all&sort=created&direction=desc&per_page=100&page={page}&since={since}
```

#### Releases

```
GET /repos/{owner}/{repo}/releases?per_page=100&page={page}
```

**Rate Limit:** 5,000 req/hour (authenticated)

---

### 4.3 Rate Limit Management

#### Throttling

```typescript
// Pseudo-code for rate limit throttling
const RATE_LIMIT_PER_MINUTE = 100;
let requestsThisMinute = 0;
let lastResetTime = Date.now();

async function throttledRequest(fn) {
  const now = Date.now();
  if (now - lastResetTime > 60000) {
    requestsThisMinute = 0;
    lastResetTime = now;
  }
  
  if (requestsThisMinute >= RATE_LIMIT_PER_MINUTE) {
    await sleep(60000 - (now - lastResetTime));
  }
  
  requestsThisMinute++;
  return await fn();
}
```

#### Exponential Backoff on 429

```typescript
async function fetchWithBackoff(url, retries = 3) {
  for (let i = 0; i < retries; i++) {
    const response = await fetch(url);
    
    if (response.status === 429) {
      const retryAfter = response.headers.get('Retry-After') || Math.pow(2, i);
      await sleep(retryAfter * 1000);
      continue;
    }
    
    return response;
  }
  
  throw new Error('Rate limit exceeded after retries');
}
```

#### Caching

```typescript
// Cache key generation
function cacheKey(method, path, params) {
  const hash = crypto.createHash('md5')
    .update(`${method}:${path}:${JSON.stringify(params)}`)
    .digest('hex');
  return `github:${method.toLowerCase()}:${hash}`;
}

// Cache lookup with TTL
async function cachedFetch(url, options) {
  const key = cacheKey(options.method || 'GET', url, options.params);
  const cached = await redis.get(key);
  
  if (cached) {
    return JSON.parse(cached);
  }
  
  const result = await fetchWithBackoff(url, options);
  await redis.setex(key, 86400, JSON.stringify(result)); // 24h TTL
  return result;
}
```

---

## 5. Scan Job Lifecycle

### 5.1 State Machine

```
┌─────────┐
│ pending │
└────┬────┘
     │
     ▼
┌─────────┐
│ running │───▶ Update progress every 10s
└────┬────┘
     │
     ├──────────────┐
     │              │
     ▼              ▼
┌──────────┐   ┌────────┐
│completed │   │ failed │
└──────────┘   └───┬────┘
                   │
                   │ (retry logic)
                   ▼
              ┌─────────┐
              │ pending │
              └─────────┘
```

### 5.2 Scan Stages

1. **Validate Repo** (5s)
   - Check repo exists via HEAD request
   - Return 404 if not found

2. **Check Cache** (2s)
   - Look for scan <24h old
   - Return cached result if available

3. **Fetch Metadata** (10s)
   - GraphQL query for repo info
   - Store in `repositories` table

4. **Fetch Timeline Data** (30s-10min)
   - Paginated GraphQL/REST for commits, PRs, issues, releases
   - Progress updates every 10s
   - Store raw data in cache

5. **Aggregate Metrics** (10s)
   - Compute weekly buckets
   - Calculate acceleration, leverage, confidence
   - Insert into `weekly_metrics`, `daily_snapshots`

6. **Complete** (2s)
   - Update `scan_jobs.status = 'completed'`
   - Invalidate report cache

### 5.3 Error Handling

| Error Type | Retry? | Backoff | Max Retries |
|------------|--------|---------|-------------|
| Network timeout | Yes | 1min | 3 |
| GitHub 403/429 | Yes | 5-30min | 3 |
| GitHub 404 | No | - | - |
| Invalid repo format | No | - | - |
| Database error | Yes | 1min | 3 |
| Aggregation crash | Yes | 1min | 1 |

---

## 6. Frontend Design

### 6.1 Page Structure

#### `/` - Home Page

- Repo input form (large, centered)
- Example reports section (3-5 hardcoded examples)
- Brief positioning statement
- Link to methodology

#### `/repos/[owner]/[repo]` - Report Page

Sections:
1. **Summary Card** (acceleration, leverage, confidence)
2. **Timeline Chart** (toggle between series)
3. **Acceleration View** (baseline comparison)
4. **Contributor Leverage** (per-contributor output)
5. **Data Confidence** (explain methodology)
6. **Report an Issue** (dispute form)

#### `/methodology` - Documentation

- How metrics are calculated
- Data sources and limitations
- Confidence level definitions
- FAQ

### 6.2 Component Library

- **UI:** shadcn/ui (Radix primitives + Tailwind)
- **Charts:** Recharts
- **Forms:** React Hook Form + Zod validation
- **Data Fetching:** TanStack Query (React Query)

### 6.3 Performance Targets

| Metric | Target |
|--------|--------|
| First Contentful Paint | <1.5s |
| Time to Interactive | <3s |
| Chart Render (100 data points) | <500ms |
| Scan Poll Interval | 5s |

---

## 7. Security & Trust

### 7.1 API Authentication

- Public endpoints: No auth required
- Internal worker endpoints: Bearer token (env var)
- GitHub API: OAuth token (bot account)

### 7.2 Rate Limiting (User-Facing)

- Unauthenticated: 5 scans/hour per IP
- Future authenticated: 20 scans/hour per user

Implementation: Vercel middleware + Redis counter

### 7.3 Dispute Process

1. User submits form on report page
2. Ticket created in `disputes` table
3. Email notification to `hello@vibestats.io`
4. Manual review within 48 hours
5. Resolution: correct data, add annotation, or remove report
6. Respond to submitter

### 7.4 Takedown Policy

- Repo owner can request full removal
- No questions asked for owner requests
- Removal within 48 hours
- Add to blocklist to prevent re-scan

---

## 8. Monitoring & Observability

### 8.1 Metrics to Track

- Scan job success/failure rate
- Average scan duration by repo size
- GitHub API rate limit headroom
- Cache hit rate
- Report page views
- Dispute ticket volume

### 8.2 Alerting

| Condition | Severity | Action |
|-----------|----------|--------|
| Scan failure rate >20% | High | Page on-call |
| GitHub rate limit <10% remaining | Medium | Slow down workers |
| Database CPU >80% | Medium | Investigate queries |
| Dispute backlog >10 tickets | Low | Assign reviewer |

### 8.3 Logging

- Structured JSON logs
- Correlation ID per scan job
- Log levels: ERROR, WARN, INFO, DEBUG
- Retention: 30 days

---

## 9. Cost Estimates (MVP)

| Service | Tier | Monthly Cost |
|---------|------|--------------|
| Vercel | Pro | $20 |
| Supabase | Free | $0 |
| Upstash Redis | Free | $0 |
| Domain | - | $15/year |
| GitHub OAuth | - | $0 |
| **Total** | | **~$20/month** |

**Assumptions:**
- 1,000 scans/month
- 10,000 report page views/month
- Free tier limits sufficient for MVP

---

## 10. Open Technical Decisions

1. **Background Job Execution:**
   - Option A: Vercel cron + serverless functions
   - Option B: GitHub Actions scheduled workflow
   - Option C: Dedicated worker server (Fly.io, Railway)
   - **Decision:** Start with Vercel cron, migrate if needed

2. **Database Migrations:**
   - Option A: Supabase Dashboard manual
   - Option B: Drizzle ORM migrations
   - Option C: Prisma migrations
   - **Decision:** Drizzle for type safety + version control

3. **Chart Library:**
   - Option A: Recharts (React-native, simple)
   - Option B: ECharts (feature-rich, heavier)
   - Option C: Chart.js (middle ground)
   - **Decision:** Recharts for MVP simplicity

---

## 11. Next Steps

1. **Create Database Schema** - Write SQL migrations
2. **Set Up Project Repo** - Monorepo structure
3. **Implement Scan Worker** - Core GitHub API integration
4. **Build API Endpoints** - Hono server
5. **Create Frontend Pages** - Next.js app
6. **Deploy MVP** - Vercel + Supabase
7. **Test with Example Repos** - Validate accuracy

---

**End of Engineering Design v1**
