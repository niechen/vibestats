# VibeStats GitHub Issues Plan

**Generated:** 2026-06-07  
**Linked PRD:** `docs/vibe-stats-prd-v2.md`  
**Linked Design:** `docs/vibestats-engineering-design-v1.md`

---

## Phase 1: Prototype (Week 1-2)

### Issue #1: CLI Scan Tool - Basic GitHub API Integration

**Labels:** `phase-1`, `cli`, `github-api`  
**Estimate:** 3 days  
**Assignee:** TBD

**Description:**

Build a Node.js CLI tool that can scan a single public GitHub repository and output a JSON report with:
- Repository metadata (owner, name, stars, forks, created_at, primary_language)
- Weekly timeline for commits, PRs, issues, releases (last 90 days)
- Basic acceleration calculation (30d vs 90d)

**Requirements:**
- Use GraphQL API as primary, REST as fallback
- Authenticate with GitHub token from env var
- Output JSON to stdout or file
- Handle rate limits gracefully (throttle + retry)
- No database yet - JSON file storage only

**Acceptance Criteria:**
- [ ] Can scan `vercel/next.js` and produce timeline in under 5 minutes
- [ ] JSON output includes all required fields
- [ ] Handles 429 rate limit with exponential backoff
- [ ] Works with unauthenticated requests (low limits) and authenticated (higher limits)

**Technical Notes:**
- Use `@octokit/graphql` for GraphQL queries
- Use `commander` or `cac` for CLI parsing
- Store raw API responses in `.vibestats/cache/` for debugging

---

### Issue #2: Acceleration Calculation Logic

**Labels:** `phase-1`, `metrics`, `algorithm`  
**Estimate:** 1 day  
**Assignee:** TBD

**Description:**

Implement acceleration calculation algorithm:
- Compare last 30 days vs prior 90-day median
- Detect burst windows (>3x baseline)
- Generate interpretation text

**Requirements:**
- Function: `calculateAcceleration(weeklyMetrics)` returns multiplier + interpretation
- Handle edge cases: no baseline data, zero activity, insufficient history
- Return confidence level based on data completeness

**Acceptance Criteria:**
- [ ] Unit tests for normal case (has baseline)
- [ ] Unit tests for edge cases (no baseline, zero activity)
- [ ] Interpretation text is human-readable and neutral
- [ ] Confidence level matches operational definitions from PRD

**Formula:**
```typescript
acceleration = velocity_last_30d / median_velocity_prior_90d

confidence = 
  High: >12 months history AND >100 data points AND <10% missing
  Medium: >6 months OR >50 points OR <30% missing
  Low: <6 months AND <50 points AND >30% missing
```

---

### Issue #3: Bot Detection & Exclusion

**Labels:** `phase-1`, `metrics`, `bot-detection`  
**Estimate:** 0.5 days  
**Assignee:** TBD

**Description:**

Implement bot detection to exclude automated accounts from contributor counts:
- Hardcoded bot list: `dependabot[bot]`, `renovate[bot]`, `github-actions[bot]`, etc.
- Exclude bot commits from human leverage metrics
- Show bot activity separately if significant (>20%)

**Requirements:**
- Function: `isBot(login: string): boolean`
- Apply exclusion in weekly aggregation
- Track bot activity count separately

**Acceptance Criteria:**
- [ ] Known bots excluded from contributor counts
- [ ] Bot commits still counted in total commits (for velocity)
- [ ] Report shows "X commits from bots excluded" if >20%

---

## Phase 2: MVP Backend (Week 3-4)

### Issue #4: Database Schema & Migrations

**Labels:** `phase-2`, `database`, `schema`  
**Estimate:** 1 day  
**Assignee:** TBD

**Description:**

Set up PostgreSQL database with Drizzle ORM migrations:
- Tables: `repositories`, `weekly_metrics`, `daily_snapshots`, `scan_jobs`, `scan_cache`
- Indexes as specified in engineering design
- Seed script for local development

**Requirements:**
- Use Supabase for MVP (free tier)
- Drizzle ORM for type-safe queries
- Migration files version-controlled
- Local dev setup documented

**Acceptance Criteria:**
- [ ] All tables created with correct schema
- [ ] Indexes in place for performance
- [ ] Migration rollback works
- [ ] README updated with setup instructions

**Files:**
- `db/schema.ts` - Drizzle schema definitions
- `db/migrations/0001_initial.sql` - Initial migration
- `db/seed.ts` - Seed script for local dev

---

### Issue #5: Scan Job Queue System

**Labels:** `phase-2`, `backend`, `queue`  
**Estimate:** 2 days  
**Assignee:** TBD

**Description:**

Implement scan job queue using PostgreSQL-based queue:
- Create job: `POST /api/scans`
- Poll status: `GET /api/scans/:id`
- Process pending jobs with worker
- Retry logic with exponential backoff

**Requirements:**
- Job states: `pending`, `running`, `completed`, `failed`
- Max 3 retries with 1min, 5min, 30min backoff
- Progress tracking (stage + percent complete)
- Concurrency limit: max 10 jobs running

**Acceptance Criteria:**
- [ ] Can create job and poll until completion
- [ ] Failed jobs retry automatically
- [ ] Progress updates visible during scan
- [ ] Rate limiting enforced (100 req/min to GitHub)

---

### Issue #6: API Response Cache Layer

**Labels:** `phase-2`, `backend`, `cache`  
**Estimate:** 1 day  
**Assignee:** TBD

**Description:**

Implement caching layer for GitHub API responses:
- Cache key: `github:{method}:{path}:{params_hash}`
- TTL: 24 hours
- Redis (Upstash) for MVP

**Requirements:**
- Cache lookup before making API call
- Store successful responses automatically
- Invalidate on schema change (manual trigger)
- Fallback to DB cache if Redis unavailable

**Acceptance Criteria:**
- [ ] Cache hit rate >50% for repeated scans
- [ ] Expired entries auto-cleaned
- [ ] Graceful degradation if Redis down

---

### Issue #7: Hono API Server Setup

**Labels:** `phase-2`, `backend`, `api`  
**Estimate:** 2 days  
**Assignee:** TBD

**Description:**

Set up Hono API server on Node.js 20:
- Endpoints: `/api/scans`, `/api/repos/:owner/:repo`, `/api/repos/:owner/:repo/timeline`
- Middleware: CORS, rate limiting, error handling
- Deploy to Vercel

**Requirements:**
- TypeScript + strict mode
- Zod validation for request bodies
- Structured JSON logging
- Health check endpoint

**Acceptance Criteria:**
- [ ] All endpoints functional
- [ ] Rate limiting working (5 scans/hour per IP)
- [ ] Error responses follow RFC 7807 (problem JSON)
- [ ] Deploys to Vercel successfully

---

## Phase 3: MVP Frontend (Week 5-6)

### Issue #8: Next.js App Setup

**Labels:** `phase-3`, `frontend`, `setup`  
**Estimate:** 1 day  
**Assignee:** TBD

**Description:**

Set up Next.js 14 app with App Router:
- TypeScript + ESLint + Prettier
- Tailwind CSS + shadcn/ui
- TanStack Query for data fetching
- Deploy to Vercel

**Requirements:**
- Monorepo structure (or separate repos)
- shadcn/ui components initialized
- Environment variables documented
- CI/CD pipeline (Vercel auto-deploy)

**Acceptance Criteria:**
- [ ] `npm run dev` works locally
- [ ] Deploys to Vercel on push to main
- [ ] ESLint passes with no errors
- [ ] Basic layout with header/footer

---

### Issue #9: Home Page with Repo Input

**Labels:** `phase-3`, `frontend`, `page`  
**Estimate:** 1 day  
**Assignee:** TBD

**Description:**

Build home page (`/`) with:
- Large repo input form (owner/repo)
- Example reports section (hardcoded 5-10 repos)
- Brief positioning statement
- Link to methodology

**Requirements:**
- Form validation (valid GitHub owner/repo format)
- Submit triggers scan via API
- Redirect to report page or poll status
- Responsive design (mobile-friendly)

**Acceptance Criteria:**
- [ ] Can input `vercel/next.js` and start scan
- [ ] Shows examples section with working links
- [ ] Form errors are clear and helpful
- [ ] Mobile layout works

---

### Issue #10: Report Page - Summary Card

**Labels:** `phase-3`, `frontend`, `page`, `chart`  
**Estimate:** 2 days  
**Assignee:** TBD

**Description:**

Build report page (`/repos/[owner]/[repo]`) summary section:
- Repository metadata (name, description, stars, forks, age)
- Acceleration card (multiplier + interpretation)
- Human leverage card (contributors, commits/contributor)
- Confidence card (level + reason)

**Requirements:**
- Fetch data from `/api/repos/:owner/:repo`
- Loading skeleton while fetching
- Error state with retry button
- Share button (copy URL)

**Acceptance Criteria:**
- [ ] Summary cards display correctly
- [ ] Loading state shows skeleton
- [ ] Error state allows retry
- [ ] Mobile layout stacks cards vertically

---

### Issue #11: Report Page - Timeline Chart

**Labels:** `phase-3`, `frontend`, `page`, `chart`  
**Estimate:** 2 days  
**Assignee:** TBD

**Description:**

Build timeline chart section:
- Weekly commits, PRs, issues, releases
- Toggle between series
- Time window selector (30d, 90d, 12mo, all)
- Recharts implementation

**Requirements:**
- Fetch from `/api/repos/:owner/:repo/timeline`
- Zoom/pan not required for MVP
- Tooltip shows exact values
- Responsive chart (resize on window change)

**Acceptance Criteria:**
- [ ] Chart renders with 90 days of data
- [ ] Toggle switches series correctly
- [ ] Window selector updates data
- [ ] Tooltip shows date + values

---

### Issue #12: Report Page - Acceleration View

**Labels:** `phase-3`, `frontend`, `page`  
**Estimate:** 1 day  
**Assignee:** TBD

**Description:**

Build acceleration analysis section:
- Rolling velocity line (7-day MA, 30-day MA)
- Historical baseline band
- Acceleration multiplier annotation
- Burst markers

**Requirements:**
- Clear visual distinction between recent vs baseline
- Interpretation text from API
- Confidence note

**Acceptance Criteria:**
- [ ] Baseline band visible
- [ ] Multiplier displayed prominently
- [ ] Burst windows marked
- [ ] Interpretation text is readable

---

### Issue #13: Report Page - Contributor Leverage View

**Labels:** `phase-3`, `frontend`, `page`  
**Estimate:** 1 day  
**Assignee:** TBD

**Description:**

Build contributor leverage section:
- Active contributors over time
- Top contributor share (%)
- Commits per contributor
- Bot activity note (if significant)

**Requirements:**
- Bar chart or line chart
- Show bot exclusion note if >20% bot activity

**Acceptance Criteria:**
- [ ] Contributor trend visible
- [ ] Top contributor % displayed
- [ ] Bot note shown when applicable

---

### Issue #14: Methodology Page

**Labels:** `phase-3`, `frontend`, `page`, `docs`  
**Estimate:** 1 day  
**Assignee:** TBD

**Description:**

Build methodology documentation page (`/methodology`):
- How metrics are calculated
- Data sources (GitHub API)
- Confidence level definitions
- Limitations and caveats
- FAQ

**Requirements:**
- Markdown or MDX content
- Table of contents
- Link to dispute process

**Acceptance Criteria:**
- [ ] All sections from PRD included
- [ ] Formulas are clear
- [ ] Caveats are prominent
- [ ] Dispute link works

---

### Issue #15: Dispute Form

**Labels:** `phase-3`, `frontend`, `form`  
**Estimate:** 1 day  
**Assignee:** TBD

**Description:**

Build dispute/correction form:
- Fields: repo, issue type, description, evidence URL, contact email
- Submit to `/api/disputes`
- Confirmation message with ticket ID

**Requirements:**
- Form validation (required fields, email format)
- Success state with ticket ID
- Error state with retry

**Acceptance Criteria:**
- [ ] Form submits successfully
- [ ] Validation errors are clear
- [ ] Ticket ID shown on success
- [ ] Email confirmation mentioned

---

## Phase 4: Polish + Launch (Week 7-8)

### Issue #16: Error Handling & Edge Cases

**Labels:** `phase-4`, `backend`, `error-handling`  
**Estimate:** 2 days  
**Assignee:** TBD

**Description:**

Comprehensive error handling:
- GitHub API failures (timeout, 403, 404)
- Database connection errors
- Rate limit exhaustion
- Invalid repo format
- Partial data scenarios

**Requirements:**
- User-friendly error messages
- Retry options where applicable
- Logging for debugging
- Graceful degradation

**Acceptance Criteria:**
- [ ] All error states have clear messages
- [ ] Retry buttons where appropriate
- [ ] Errors logged with correlation ID
- [ ] No crashes on edge cases

---

### Issue #17: Performance Optimization

**Labels:** `phase-4`, `frontend`, `backend`, `performance`  
**Estimate:** 2 days  
**Assignee:** TBD

**Description:**

Optimize performance:
- Frontend: code splitting, image optimization, cache headers
- Backend: query optimization, index usage, cache hit rate
- Target: FCP <1.5s, TTI <3s

**Requirements:**
- Lighthouse score >90
- API response time <500ms for cached reports
- Chart render <500ms for 100 data points

**Acceptance Criteria:**
- [ ] Lighthouse audit passes
- [ ] Cached reports load instantly
- [ ] Charts render smoothly

---

### Issue #18: Launch Landing Page

**Labels:** `phase-4`, `frontend`, `marketing`  
**Estimate:** 1 day  
**Assignee:** TBD

**Description:**

Polish home page for launch:
- Hero section with value prop
- Example reports showcase
- Social proof (if any)
- Footer with links (methodology, contact, terms)

**Requirements:**
- Compelling copy
- Clean design
- Mobile responsive

**Acceptance Criteria:**
- [ ] Value prop is clear in 5 seconds
- [ ] Examples section looks polished
- [ ] Footer has all required links

---

### Issue #19: Social Card Generation

**Labels:** `phase-4`, `backend`, `og-image`  
**Estimate:** 1 day  
**Assignee:** TBD

**Description:**

Generate Open Graph images for report pages:
- Dynamic OG image with repo name, acceleration multiplier, confidence
- Use Vercel OG Image or @vercel/og
- Cache generated images

**Requirements:**
- OG image at `/api/og/[owner]/[repo]`
- Includes: repo name, stars, acceleration, confidence badge
- Consistent branding

**Acceptance Criteria:**
- [ ] OG image generates correctly
- [ ] Twitter/Discord preview shows correct image
- [ ] Images cached (not regenerated every request)

---

### Issue #20: Final Testing & Bug Fixes

**Labels:** `phase-4`, `testing`, `bugfix`  
**Estimate:** 3 days  
**Assignee:** TBD

**Description:**

Comprehensive testing before launch:
- Test with 10+ diverse repos (small, medium, large, old, new, active, inactive)
- Fix bugs found during testing
- Validate accuracy against manual inspection
- Load testing (simulate concurrent scans)

**Requirements:**
- Test repo list documented
- Bug tracker maintained
- Performance benchmarks recorded

**Acceptance Criteria:**
- [ ] All test repos produce valid reports
- [ ] No critical bugs open
- [ ] Performance meets targets
- [ ] Ready for public launch

---

## Backlog (V1 Post-MVP)

### Issue #21: GitHub Action for Self-Reporting

**Labels:** `v1`, `github-action`, `backlog`  
**Note:** Moved from MVP due to security complexity

**Description:**

Create GitHub Action that repos can install to self-report stats:
- Runs on schedule (weekly) or on release
- Collects local git stats
- Posts to VibeStats API with authentication
- Ownership verification required

**Requirements:**
- Secure auth model (app installation or token)
- Prevent spam/fake reports
- Rate limit per repo

---

### Issue #22: Public Index Pages

**Labels:** `v1`, `frontend`, `backlog`  
**Note:** Moved from MVP due to gaming risk

**Description:**

Dynamic index pages:
- Recently scanned repos
- Fastest accelerating (with confidence threshold)
- Filter by language, stars, age

**Requirements:**
- Anti-gaming measures (rate limit visibility)
- Confidence thresholds for rankings
- Abuse reporting mechanism

---

### Issue #23: Review Metrics Integration

**Labels:** `v1`, `metrics`, `backlog`  
**Note:** Moved from MVP due to API cost

**Description:**

Add review/comment metrics:
- Reviews per PR
- Review comments per PR
- Self-merge ratio
- PR open-to-merge time

**Requirements:**
- GraphQL mandatory for efficiency
- Caveats about external review workflows
- Governance coverage interpretation

---

### Issue #24: README Badge

**Labels:** `v1`, `badge`, `backlog`  
**Note:** Moved from MVP

**Description:**

Generate badges for maintainers:
- SVG badge with acceleration, confidence
- Embeddable in README
- Opt-in only

**Requirements:**
- Badge endpoint: `/badge/[owner]/[repo].svg`
- Multiple styles (flat, plastic, etc.)
- Links back to report page

---

## Meta Issues

### Issue #25: Set Up Project Monorepo

**Labels:** `setup`, `infrastructure`  
**Estimate:** 0.5 days

**Description:**

Create monorepo structure:
```
/vibestats
  /apps
    /web          # Next.js frontend
    /api          # Hono backend
  /packages
    /shared       # Shared types, utilities
    /scanner      # GitHub API scanner library
  /db             # Database schema, migrations
  /docs           # PRD, design, methodology
```

**Requirements:**
- Turborepo or Nx for build orchestration
- Shared TypeScript config
- Consistent linting/prettier

---

### Issue #26: CI/CD Pipeline

**Labels:** `setup`, `infrastructure`  
**Estimate:** 0.5 days

**Description:**

Set up CI/CD:
- GitHub Actions for tests, lint, type check
- Vercel auto-deploy on merge to main
- Preview deployments for PRs

**Requirements:**
- Tests run on every PR
- Type check passes before merge
- Preview URL posted to PR

---

### Issue #27: Monitoring & Alerting Setup

**Labels:** `setup`, `observability`  
**Estimate:** 1 day

**Description:**

Set up monitoring:
- APM (Sentry or similar)
- Uptime monitoring
- Error alerting
- Dashboard for key metrics

**Requirements:**
- Error tracking configured
- Uptime checks for API and frontend
- Alert channel (Slack, email)

---

## Issue Template

Use this template for creating issues:

```markdown
## Description

[What and why]

## Requirements

- [ ] Requirement 1
- [ ] Requirement 2

## Acceptance Criteria

- [ ] Criterion 1
- [ ] Criterion 2

## Technical Notes

[Any additional context]

## Files

- `path/to/file.ts` - Description
```

---

**End of Issues Plan**
