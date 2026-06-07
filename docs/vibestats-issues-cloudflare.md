# VibeStats GitHub Issues Plan (Cloudflare Stack)

**Generated:** 2026-06-07  
**Linked PRD:** `docs/vibe-stats-prd-v2.md`  
**Linked Design:** `docs/vibestats-engineering-design-v2-cloudflare.md`

---

## Pre-flight Setup

### Issue #0: GitHub OAuth App + Bot Account Setup

**Labels:** `setup`, `infrastructure`  
**Estimate:** 0.5 days

**Description:**

Create dedicated GitHub account and OAuth app for authenticated API access.

**Requirements:**

- [ ] Create GitHub account for VibeStats bot (e.g., `@vibestats-bot`)
- [ ] Create OAuth app under bot account
- [ ] Generate personal access token with scopes: `repo`, `read:org`
- [ ] Store token in Cloudflare Workers secrets (`wrangler secret put GITHUB_TOKEN`)
- [ ] Document token rotation procedure

**Scopes Required:**

- `repo` (full control of private repositories) - for reading all public repo data
- `read:org` (read org membership) - for org-owned repos

**Acceptance Criteria:**

- [ ] Bot account created
- [ ] OAuth app registered
- [ ] Token stored as Workers secret
- [ ] Token works in CLI test

---

## Phase 1: Prototype (Week 1-2)

### Issue #1: CLI Scan Tool - Basic GitHub API Integration

**Labels:** `phase-1`, `cli`, `github-api`  
**Estimate:** 3 days

**Description:**

Build a Node.js CLI tool that can scan a single public GitHub repository and output a JSON report.

**Requirements:**

- [ ] Use GraphQL API as primary, REST as fallback
- [ ] Authenticate with GitHub token from env var
- [ ] Output JSON to stdout or file
- [ ] Handle rate limits gracefully (throttle + retry)
- [ ] No database yet - JSON file storage only

**Acceptance Criteria:**

- [ ] Can scan `vercel/next.js` and produce timeline in under 5 minutes
- [ ] JSON output includes all required fields
- [ ] Handles 429 rate limit with exponential backoff
- [ ] Works with unauthenticated and authenticated requests

**Technical Notes:**

- Use `@octokit/graphql` for GraphQL queries
- Use `commander` or `cac` for CLI parsing
- Store raw API responses in `.vibestats/cache/` for debugging

---

### Issue #2: Acceleration Calculation Logic

**Labels:** `phase-1`, `metrics`, `algorithm`  
**Estimate:** 1 day

**Description:**

Implement acceleration calculation algorithm.

**Requirements:**

- [ ] Function: `calculateAcceleration(weeklyMetrics)` returns multiplier + interpretation
- [ ] Handle edge cases: no baseline, zero activity, insufficient history
- [ ] Return confidence level based on data completeness

**Acceptance Criteria:**

- [ ] Unit tests for normal case (has baseline)
- [ ] Unit tests for edge cases (no baseline, zero activity)
- [ ] Interpretation text is human-readable and neutral

**Formula:**

```typescript
acceleration = velocity_last_30d / median_velocity_prior_90d

confidence:
  High: >12 months AND >100 points AND <10% missing
  Medium: >6 months OR >50 points OR <30% missing
  Low: <6 months AND <50 points AND >30% missing
```

---

### Issue #3: Bot Detection & Exclusion

**Labels:** `phase-1`, `metrics`, `bot-detection`  
**Estimate:** 0.5 days

**Description:**

Implement bot detection to exclude automated accounts from contributor counts.

**Requirements:**

- [ ] Function: `isBot(login: string): boolean`
- [ ] Hardcoded list: `dependabot[bot]`, `renovate[bot]`, `github-actions[bot]`, etc.
- [ ] Exclude bot commits from human leverage metrics
- [ ] Show bot activity separately if >20%

**Acceptance Criteria:**

- [ ] Known bots excluded from contributor counts
- [ ] Bot commits still counted in total commits
- [ ] Report shows "X commits from bots excluded" if >20%

---

## Phase 2: MVP Backend on Cloudflare (Week 3-4)

### Issue #4: D1 Database Schema & Migrations

**Labels:** `phase-2`, `database`, `cloudflare`  
**Estimate:** 1 day

**Description:**

Set up Cloudflare D1 database with SQLite migrations.

**Requirements:**

- [ ] Tables: `repositories`, `weekly_metrics`, `daily_snapshots`, `scan_jobs`, `disputes`
- [ ] Indexes as specified in engineering design v2
- [ ] Use TEXT for UUIDs (no native UUID in SQLite)
- [ ] Raw SQL migrations (Drizzle optional for V1)
- [ ] Seed script for local development

**Acceptance Criteria:**

- [ ] All tables created with correct SQLite schema
- [ ] Indexes in place for performance
- [ ] Migration rollback works
- [ ] README updated with setup instructions

**Files:**

- `db/migrations/0001_initial.sql` - Initial migration
- `db/seed.sql` - Seed script for local dev

---

### Issue #5: Cloudflare Queues + Scan Job System

**Labels:** `phase-2`, `backend`, `queue`, `cloudflare`  
**Estimate:** 2 days

**Description:**

Implement scan job queue using Cloudflare Queues.

**Requirements:**

- [ ] Create job: `POST /api/scans`
- [ ] Poll status: `GET /api/scans/:id`
- [ ] Job states: `pending`, `running`, `completed`, `failed`
- [ ] Max 3 retries with 1min, 5min, 30min backoff
- [ ] Progress tracking (stage + percent complete)
- [ ] Concurrency limit: max 10 jobs per batch
- [ ] Break large scans into multiple queue messages (avoid 10s timeout)
- [ ] Cron trigger every minute to poll pending jobs

**Acceptance Criteria:**

- [ ] Can create job and poll until completion
- [ ] Failed jobs retry automatically
- [ ] Progress updates visible during scan
- [ ] Rate limiting enforced (100 req/min to GitHub)

---

### Issue #6: Cloudflare KV Cache Layer

**Labels:** `phase-2`, `backend`, `cache`, `cloudflare`  
**Estimate:** 1 day

**Description:**

Implement caching layer using Cloudflare KV.

**Requirements:**

- [ ] Cache key: `github:{method}:{path}:{params_hash}`
- [ ] TTL: 24 hours (automatic via KV expiration)
- [ ] Fallback to D1 if KV unavailable
- [ ] Cache lookup before making API call
- [ ] Store successful responses automatically with TTL
- [ ] Track cache hit rate for monitoring

**Acceptance Criteria:**

- [ ] Cache hit rate >50% for repeated scans
- [ ] Expired entries auto-deleted by KV
- [ ] Graceful degradation if KV down

---

### Issue #7: Hono API on Cloudflare Workers

**Labels:** `phase-2`, `backend`, `api`, `cloudflare`  
**Estimate:** 2 days

**Description:**

Set up Hono API server on Cloudflare Workers.

**Requirements:**

- [ ] Endpoints: `/api/scans`, `/api/repos/:owner/:repo`, `/api/repos/:owner/:repo/timeline`, `/api/examples`, `/api/disputes`
- [ ] Middleware: CORS, rate limiting, error handling
- [ ] Bindings: D1, KV, R2, Queues
- [ ] TypeScript + strict mode
- [ ] Zod validation for request bodies
- [ ] Structured JSON logging
- [ ] Health check endpoint
- [ ] Wrangler configuration for all bindings

**Acceptance Criteria:**

- [ ] All endpoints functional
- [ ] Rate limiting working (5 scans/hour per IP)
- [ ] Error responses follow RFC 7807 (problem JSON)
- [ ] Deploys via `wrangler deploy`

---

## Phase 3: MVP Frontend on Pages (Week 5-6)

### Issue #8: Next.js on Cloudflare Pages

**Labels:** `phase-3`, `frontend`, `setup`, `cloudflare`  
**Estimate:** 1 day

**Description:**

Set up Next.js 14 app on Cloudflare Pages.

**Requirements:**

- [ ] TypeScript + ESLint + Prettier
- [ ] Tailwind CSS + shadcn/ui
- [ ] TanStack Query for data fetching
- [ ] Static export mode for Pages compatibility
- [ ] shadcn/ui components initialized
- [ ] Environment variables documented
- [ ] CI/CD pipeline (`wrangler pages deploy`)

**Acceptance Criteria:**

- [ ] `npm run dev` works locally
- [ ] Deploys via `wrangler pages deploy` on push to main
- [ ] ESLint passes with no errors
- [ ] Basic layout with header/footer

---

### Issue #9: Home Page with Repo Input

**Labels:** `phase-3`, `frontend`, `page`  
**Estimate:** 1 day

**Description:**

Build home page (`/`) with repo input form and examples.

**Requirements:**

- [ ] Large repo input form (owner/repo)
- [ ] Example reports section (hardcoded 5-10 repos)
- [ ] Form validation (valid GitHub owner/repo format)
- [ ] Submit triggers scan via API
- [ ] Responsive design (mobile-friendly)

**Acceptance Criteria:**

- [ ] Can input `vercel/next.js` and start scan
- [ ] Shows examples section with working links
- [ ] Form errors are clear and helpful

---

### Issue #10: Report Page - Summary Card

**Labels:** `phase-3`, `frontend`, `page`  
**Estimate:** 2 days

**Description:**

Build report page (`/repos/[owner]/[repo]`) summary section.

**Requirements:**

- [ ] Repository metadata (name, description, stars, forks, age)
- [ ] Acceleration card (multiplier + interpretation)
- [ ] Human leverage card (contributors, commits/contributor)
- [ ] Confidence card (level + reason)
- [ ] Fetch data from `/api/repos/:owner/:repo`

**Acceptance Criteria:**

- [ ] Summary cards display correctly
- [ ] Loading state shows skeleton
- [ ] Error state allows retry

---

### Issue #11: Report Page - Timeline Chart

**Labels:** `phase-3`, `frontend`, `page`, `chart`  
**Estimate:** 2 days

**Description:**

Build timeline chart section with Recharts.

**Requirements:**

- [ ] Weekly commits, PRs, issues, releases
- [ ] Toggle between series
- [ ] Time window selector (30d, 90d, 12mo, all)
- [ ] Fetch from `/api/repos/:owner/:repo/timeline`

**Acceptance Criteria:**

- [ ] Chart renders with 90 days of data
- [ ] Toggle switches series correctly
- [ ] Window selector updates data
- [ ] Tooltip shows date + values

---

### Issue #12: Report Page - Acceleration View

**Labels:** `phase-3`, `frontend`, `page`  
**Estimate:** 1 day

**Description:**

Build acceleration analysis section.

**Requirements:**

- [ ] Rolling velocity line (7-day MA, 30-day MA)
- [ ] Historical baseline band
- [ ] Acceleration multiplier annotation
- [ ] Burst markers
- [ ] Interpretation text from API

**Acceptance Criteria:**

- [ ] Baseline band visible
- [ ] Multiplier displayed prominently
- [ ] Burst windows marked

---

### Issue #13: Report Page - Contributor Leverage View

**Labels:** `phase-3`, `frontend`, `page`  
**Estimate:** 1 day

**Description:**

Build contributor leverage section.

**Requirements:**

- [ ] Active contributors over time
- [ ] Top contributor share (%)
- [ ] Commits per contributor
- [ ] Bot activity note (if >20%)

**Acceptance Criteria:**

- [ ] Contributor trend visible
- [ ] Top contributor % displayed
- [ ] Bot note shown when applicable

---

### Issue #14: Methodology Page

**Labels:** `phase-3`, `frontend`, `page`, `docs`  
**Estimate:** 1 day

**Description:**

Build methodology documentation page (`/methodology`).

**Requirements:**

- [ ] How metrics are calculated
- [ ] Data sources (GitHub API)
- [ ] Confidence level definitions
- [ ] Limitations and caveats
- [ ] FAQ
- [ ] Link to dispute process

**Acceptance Criteria:**

- [ ] All sections from PRD included
- [ ] Formulas are clear
- [ ] Caveats are prominent

---

### Issue #15: Dispute Form

**Labels:** `phase-3`, `frontend`, `form`  
**Estimate:** 1 day

**Description:**

Build dispute/correction form.

**Requirements:**

- [ ] Fields: repo, issue type, description, evidence URL, contact email
- [ ] Submit to `/api/disputes`
- [ ] Form validation (required fields, email format)
- [ ] Success state with ticket ID

**Acceptance Criteria:**

- [ ] Form submits successfully
- [ ] Validation errors are clear
- [ ] Ticket ID shown on success

---

## Phase 4: Polish + Launch (Week 7-8)

### Issue #16: Error Handling & Edge Cases

**Labels:** `phase-4`, `backend`, `error-handling`  
**Estimate:** 2 days

**Description:**

Comprehensive error handling.

**Requirements:**

- [ ] GitHub API failures (timeout, 403, 404)
- [ ] D1 connection errors
- [ ] Rate limit exhaustion
- [ ] Invalid repo format
- [ ] Partial data scenarios
- [ ] User-friendly error messages

**Acceptance Criteria:**

- [ ] All error states have clear messages
- [ ] Retry buttons where appropriate
- [ ] Errors logged with correlation ID

---

### Issue #17: Performance Optimization

**Labels:** `phase-4`, `frontend`, `backend`, `performance`  
**Estimate:** 2 days

**Description:**

Optimize performance for frontend and backend.

**Requirements:**

- [ ] Frontend: code splitting, image optimization, cache headers
- [ ] Backend: D1 query optimization, index usage, KV cache hit rate
- [ ] Target: FCP <1.5s, TTI <3s
- [ ] Lighthouse score >90

**Acceptance Criteria:**

- [ ] Lighthouse audit passes
- [ ] Cached reports load instantly
- [ ] Charts render smoothly (<500ms)

---

### Issue #18: Launch Landing Page

**Labels:** `phase-4`, `frontend`, `marketing`  
**Estimate:** 1 day

**Description:**

Polish home page for launch.

**Requirements:**

- [ ] Hero section with value prop
- [ ] Example reports showcase
- [ ] Footer with links (methodology, contact, terms)
- [ ] Mobile responsive

**Acceptance Criteria:**

- [ ] Value prop is clear in 5 seconds
- [ ] Examples section looks polished

---

### Issue #19: Social Card Generation

**Labels:** `phase-4`, `backend`  
**Estimate:** 1 day

**Description:**

Generate Open Graph images for report pages.

**Requirements:**

- [ ] Dynamic OG image with repo name, acceleration multiplier, confidence
- [ ] Use @vercel/og or custom implementation
- [ ] Badge endpoint: `/api/og/[owner]/[repo].svg`
- [ ] Cache generated images in R2

**Acceptance Criteria:**

- [ ] OG image generates correctly
- [ ] Twitter/Discord preview shows correct image

---

### Issue #20: Final Testing & Bug Fixes

**Labels:** `phase-4`, `testing`, `bugfix`  
**Estimate:** 3 days

**Description:**

Comprehensive testing before launch.

**Requirements:**

- [ ] Test with 10+ diverse repos (small, medium, large, old, new, active, inactive)
- [ ] Fix bugs found during testing
- [ ] Validate accuracy against manual inspection
- [ ] Load testing (simulate concurrent scans)

**Acceptance Criteria:**

- [ ] All test repos produce valid reports
- [ ] No critical bugs open
- [ ] Performance meets targets

---

## Backlog (V1 Post-MVP)

### Issue #21: [V1] GitHub Action for Self-Reporting

**Labels:** `v1`, `github-action`, `backlog`

**Description:**

Create GitHub Action that repos can install to self-report stats.

**Moved from MVP due to security complexity.**

**Requirements:**

- [ ] Runs on schedule (weekly) or on release
- [ ] Collects local git stats
- [ ] Posts to VibeStats API with authentication
- [ ] Ownership verification required
- [ ] Prevent spam/fake reports

---

### Issue #22: [V1] Public Index Pages

**Labels:** `v1`, `frontend`, `backlog`

**Description:**

Dynamic index pages for recently scanned and fastest accelerating repos.

**Moved from MVP due to gaming risk.**

**Requirements:**

- [ ] Recently scanned repos
- [ ] Fastest accelerating (with confidence threshold)
- [ ] Filter by language, stars, age
- [ ] Anti-gaming measures (rate limit visibility)
- [ ] Confidence thresholds for rankings

---

### Issue #23: [V1] Review Metrics Integration

**Labels:** `v1`, `metrics`, `backlog`

**Description:**

Add review/comment metrics for governance coverage.

**Moved from MVP due to API cost.**

**Requirements:**

- [ ] Reviews per PR
- [ ] Review comments per PR
- [ ] Self-merge ratio
- [ ] PR open-to-merge time
- [ ] GraphQL mandatory for efficiency
- [ ] Caveats about external review workflows

---

### Issue #24: [V1] README Badge

**Labels:** `v1`, `badge`, `backlog`

**Description:**

Generate badges for maintainers to embed in README.

**Moved from MVP.**

**Requirements:**

- [ ] SVG badge with acceleration, confidence
- [ ] Badge endpoint: `/badge/[owner]/[repo].svg`
- [ ] Multiple styles (flat, plastic, etc.)
- [ ] Links back to report page
- [ ] Opt-in only

---

## Meta Infrastructure

### Issue #25: Set Up Project Monorepo

**Labels:** `setup`, `infrastructure`  
**Estimate:** 0.5 days

**Description:**

Create monorepo structure.

**Structure:**

```
/vibestats
  /apps
    /web          # Next.js frontend (Cloudflare Pages)
    /api          # Hono backend (Cloudflare Workers)
  /packages
    /shared       # Shared types, utilities
    /scanner      # GitHub API scanner library
  /db             # D1 schema, migrations
  /docs           # PRD, design, methodology
```

**Requirements:**

- [ ] Turborepo or Nx for build orchestration
- [ ] Shared TypeScript config
- [ ] Consistent linting/prettier
- [ ] Wrangler configuration for all apps

---

### Issue #26: CI/CD Pipeline

**Labels:** `setup`, `infrastructure`  
**Estimate:** 0.5 days

**Description:**

Set up CI/CD pipeline.

**Requirements:**

- [ ] GitHub Actions for tests, lint, type check
- [ ] `wrangler pages deploy` on merge to main
- [ ] Preview deployments for PRs
- [ ] Tests run on every PR
- [ ] Type check passes before merge

---

### Issue #27: Monitoring + Sentry Setup

**Labels:** `setup`, `infrastructure`, `observability`  
**Estimate:** 1 day

**Description:**

Set up monitoring and observability.

**Requirements:**

- [ ] Sentry for error tracking (free tier: 5K errors/month)
- [ ] Cloudflare Analytics Engine for custom metrics
- [ ] Uptime monitoring (UptimeRobot or similar)
- [ ] Dashboard for key metrics (scan success rate, cache hit rate, etc.)
- [ ] Alert channel (Slack webhook or email)

**Metrics to Track:**

- Scan job success/failure rate
- Average scan duration by repo size
- GitHub API rate limit headroom
- KV cache hit rate
- Report page views
- Dispute ticket volume

---

### Issue #28: Domain + DNS + Email via Cloudflare

**Labels:** `phase-4`, `infrastructure`, `cloudflare`  
**Estimate:** 0.5 days

**Description:**

Set up domain, DNS, and email via Cloudflare.

**Requirements:**

- [ ] Register `vibestats.io` via Cloudflare Registrar
- [ ] Configure DNS records (auto-configured by Pages)
- [ ] Set up Email Routing (`hello@vibestats.io` → personal email)
- [ ] Configure custom domain on Cloudflare Pages
- [ ] Enable SSL/TLS (automatic via Cloudflare)

**Acceptance Criteria:**

- [ ] Domain registered
- [ ] DNS configured (auto via Pages)
- [ ] Email forwarding works
- [ ] Custom domain live on Pages

---

## Critical Path

```
#0 (OAuth) → #25 (Monorepo) → #1 (CLI) → #4 (D1) → #5 (Queues) → #7 (Workers) → #8 (Pages) → #9-13 (Report pages) → #20 (Testing)
```

**Parallelizable:**

- #2 (acceleration) and #3 (bots) parallel with #1
- #6 (KV cache) parallel with #5 (Queues)
- #14 (methodology) and #15 (dispute form) parallel with frontend
- #18 (landing) and #19 (OG images) parallel with #16-17 (errors/perf)
- #28 (domain) can start anytime

---

**End of Issues Plan (Cloudflare Stack)**
