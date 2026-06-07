# VibeStats PRD v2

**Status:** Ready for Implementation  
**Owner:** Chen  
**Date:** 2026-06-07  
**Version:** 2.0 (Narrowed MVP)

---

## 1. Executive Summary

VibeStats measures how fast open-source projects are really being built in the AI coding era. It scans any public GitHub repository and reconstructs historical engineering activity from commits, pull requests, releases, and issues.

**Core thesis:** The product is not an AI-code detector. It is **repo acceleration intelligence**: how fast a project is changing, how many humans are involved, and where the project shows unusual acceleration.

**MVP wedge (v2):** Reconstruct and explain repository acceleration over time from public GitHub metadata, **without review metrics, without GitHub Action, without public rankings**. Prove that VibeStats can produce trustworthy historical acceleration timelines before adding complexity.

**Key changes from v1:**
- Removed review/comment metrics (too API-expensive, high misinterpretation risk).
- Removed GitHub Action self-reporting (security model not ready).
- Removed public index pages (gaming risk, defers to V1).
- Added operational definitions for confidence labels.
- Replaced 60-second SLA with tiered scan time expectations.
- Switched runtime from Bun to Node.js LTS for MVP.
- Added rate limit strategy and caching layer.
- Added maintainer dispute process.

---

## 2. Product Positioning

### 2.1 Primary Positioning

> Measure how fast open-source projects are really being built in the AI coding era.

### 2.2 Product Category

Repo acceleration intelligence for open-source software.

### 2.3 What It Is Not

- Not a black-box AI-generated-code detector.
- Not a moral ranking of whether a project is "vibe coded".
- Not a replacement for code review, security scanning, or dependency auditing.
- Not a generic GitHub stats dashboard that only shows commits, stars, and contributors.
- **Not a governance quality judgment** (review metrics removed from MVP).

### 2.4 Differentiation

| Existing Tools | Focus | VibeStats Focus |
|----------------|-------|-----------------|
| vibereport.dev, slopcodemonitor.ai, codeorigin.dev | AI-code provenance detection | Longitudinal engineering behavior: acceleration, leverage |
| Star History, GStars, GitPulse | Star growth, activity dashboards | Acceleration relative to repo's own baseline, with explicit confidence labels |
| GitHub Insights/Pulse | Built-in repo stats | Cross-repo comparison, reconstructed history, shareable reports |

**Key differentiators:**
- Acceleration over time (not static totals).
- Velocity per contributor (human leverage).
- Reconstructed historical timelines + observed future snapshots.
- Clear confidence labels with operational definitions.
- Transparent methodology.

---

## 3. Target Users

### 3.1 Open-Source Users (Primary)

**Who:** Developers evaluating whether to adopt a dependency or contribute to a project.

**Jobs:**
- Check whether a repo is moving quickly or abandoned.
- Understand whether recent activity is a burst or sustained growth.
- Compare alternatives before adopting a dependency.

### 3.2 Maintainers (Secondary)

**Who:** Project maintainers who want to understand their project's momentum.

**Jobs:**
- See project acceleration trends.
- Identify burst windows and contributor changes.
- Share a public report with users/investors (V1).

### 3.3 Investors, Researchers (Tertiary)

**Who:** People tracking AI-era software production patterns.

**Jobs:**
- Find projects with unusual acceleration.
- Study how development velocity changes over time.

---

## 4. User Stories (MVP)

- As a developer, I can paste any public GitHub repository URL and get a historical velocity report.
- As a user evaluating a dependency, I can see whether recent development is a burst or sustained growth.
- As a researcher, I can view example reports with acceleration timelines.
- As a maintainer, I can request a correction if my report has incorrect data.

**Removed from MVP (moved to V1):**
- ~~As a maintainer, I can install a GitHub Action to automatically report stats.~~
- ~~As a user, I can browse recently scanned repositories.~~
- ~~As a maintainer, I can add a badge to my README.~~

---

## 5. Core Concepts & Metrics (v2)

### 5.1 Velocity

**Definition:** Engineering activity volume over a time window.

**MVP Components (GitHub API-only):**
- Commits (count, unique authors).
- Pull requests opened.
- Pull requests merged.
- Issues opened/closed.
- Releases published.
- Active GitHub actors (unique logins with any activity).

**Out of Scope for MVP:**
- LOC added/deleted (requires per-commit detail or git clone).
- Files changed (requires per-commit/detail API calls).
- Reviews and review comments (moved to V1).

### 5.2 Acceleration

**Definition:** Change in velocity relative to the repository's own historical baseline.

**MVP Methodology:**
- Compare last 30 days vs prior 90-day median.
- Compare last 90 days vs prior 12-month median.
- Rolling 7-day and 30-day moving average slopes.
- Burst detection: windows where activity exceeds 3x historical median.

**Output:**
- Acceleration multiplier (e.g., "3.1x above baseline").
- Peak acceleration windows (date ranges).
- Interpretation: "Activity increased sharply starting May 2026, driven by PR merges."

### 5.3 Human Leverage (Contributor Density)

**Definition:** Output normalized by active contributors.

**MVP Components:**
- Active contributors per week/month (unique commit/PR authors).
- Commits per active contributor.
- Merged PRs per active contributor.
- Top contributor share (% of commits/PRs from top author).
- New contributors entering (first-time authors in window).

**Bot Handling:**
- Exclude known bot accounts: `dependabot[bot]`, `renovate[bot]`, `github-actions[bot]`, etc.
- Show bot activity separately if significant (>20% of commits).

### 5.4 Confidence Labels (With Operational Definitions)

**Definition:** Reliability and completeness of each metric, with explicit thresholds.

**MVP Levels:**

| Level | Criteria |
|-------|----------|
| **High** | >12 months history AND >100 data points AND <10% missing fields |
| **Medium** | >6 months history OR >50 data points OR 10-30% missing fields |
| **Low** | <6 months history AND <50 data points AND >30% missing fields |
| **Unavailable** | Data not accessible via GitHub API (e.g., reviews disabled) |

**Per-Metric Examples:**
- Commit timeline: High (if repo age > 12 months).
- Acceleration: Medium (if 6-12 months history).
- Human leverage: High (if >100 commits with author data).

**UI Requirement:** Every metric card shows confidence level with tooltip explaining why.

### 5.5 Scan Time SLA (Tiered)

**Definition:** Expected scan duration based on repo size.

| Repo Size | Commits | Expected Scan Time | Mode |
|-----------|---------|-------------------|------|
| Small | <1k | ~30 seconds | Sync |
| Medium | 1k-10k | 1-3 minutes | Sync with progress |
| Large | >10k | 5-15 minutes | Async job with status polling |

**UI Requirement:** Show expected time before scan starts. For large repos, offer "Scan in background, notify when ready."

---

## 6. Primary Product Surface (v2)

### 6.1 Repository Report Page

**Input:** GitHub owner/repo URL or shorthand (e.g., `niechen/vibestats`).

**Top Summary Card:**
- Repository name, description, language, stars, forks, age.
- **Acceleration:** "3.1x above baseline (last 30d vs prior 90d)."
- **Human Leverage:** "2 contributors, 75 commits/contributor."
- **Confidence:** "Medium (9 months history, 85% data completeness)."
- **One-sentence interpretation:** "This repo is moving 3.1x faster than its historical baseline, driven by 2 active contributors."

**Removed from v1:**
- ~~Governance Coverage score~~ (moved to V1).

### 6.2 Historical Timeline (Core Feature)

**Time Windows:**
- Last 30 days.
- Last 90 days.
- Last 12 months.
- All time (if < 2 years).

**Series (weekly aggregation):**
- Commits.
- PRs opened.
- PRs merged.
- Active contributors.
- Issues opened/closed.
- Releases.

**Removed from v1:**
- ~~Reviews~~ (moved to V1).
- ~~Review comments~~ (moved to V1).

**UI Requirements:**
- Toggle between series.
- Show baseline band for acceleration context.
- Label confidence per chart.
- Explain data source: "Reconstructed from GitHub API."

### 6.3 Acceleration View

**Purpose:** Show when project velocity changed sharply.

**Visualizations:**
- Rolling velocity line (7-day MA, 30-day MA).
- Historical baseline band (median of prior period).
- Acceleration multiplier annotation.
- Burst markers (windows > 3x baseline).

**Interpretations:**
- "Last 14 days are 6.2x above historical median."
- "Acceleration started May 15, driven by PR merges."
- "Contributor count stayed flat; output per contributor increased."

### 6.4 Contributor Leverage View

**Purpose:** Show whether acceleration comes from more people or higher output per person.

**Metrics:**
- Active contributors over time.
- Top contributor share (%).
- New contributor count.
- Commits per contributor.
- Merged PRs per contributor.

**Bot Activity Note:**
- "Bot accounts excluded: dependabot[bot] (15 commits)."

### 6.5 Example Reports (Not Dynamic Index)

**MVP Approach:** Hardcoded list of 5-10 example repos, manually curated.

**Examples:**
- A fast-accelerating new project.
- A mature project with sustained velocity.
- A project with recent burst.
- A low-activity project.

**Why Hardcoded:**
- Avoids gaming incentives.
- Ensures quality examples.
- No rate limit burden from repeated scans.

**V1 Plan:** Dynamic "Recently Scanned" with rate limiting.

---

## 7. Data Sources & Scan Strategy

### 7.1 GitHub REST API (Primary for MVP)

**Use For:**
- Repository metadata.
- Commits list (with pagination).
- Pull requests list.
- Issues list.
- Releases list.
- Contributors list.

**Rate Limit Strategy:**
- Authenticated API token (bot account): 5,000 req/hour.
- Throttle to 100 req/min to stay well under limit.
- Exponential backoff on 429 (start 1s, max 60s).
- Cache raw API responses for 24 hours.
- Per-repo scan budget: max 500 API calls (enforced by pagination limits).

**Pagination Limits:**
- Commits: max 30 pages (3,000 commits) for sync scan.
- PRs: max 30 pages (3,000 PRs) for sync scan.
- Beyond limits: async deep scan queued.

### 7.2 GitHub GraphQL API (Primary for Historical Reconstruction)

**Use For:**
- Efficient timeline queries.
- Pagination-heavy historical reconstruction.
- Getting commits + PRs in single query.

**Why GraphQL is Primary (not optional):**
- Reduces API call count significantly.
- Can fetch commits, PRs, authors in one query.
- Avoids N+1 problem.

**Fallback:** REST API if GraphQL query fails or times out.

### 7.3 Observed Snapshots (Post-Scan)

**Purpose:** Store values that cannot be perfectly reconstructed historically.

**Examples:**
- Star count at scan time.
- Fork count at scan time.
- Report score snapshots.

**MVP Approach:**
- Store one snapshot per scan.
- Label charts as "Observed" vs "Reconstructed."

---

## 8. Technical Architecture (v2)

### 8.1 Stack Choices

**Backend:**
- Runtime: **Node.js 20 LTS** (changed from Bun).
- Framework: Hono (Node runtime) or Express.
- Database: PostgreSQL (Supabase for MVP).
- Cache: Redis (Upstash for MVP) or in-memory for prototype.
- Queue: PostgreSQL-based queue (simpler than BullMQ for MVP).

**Frontend:**
- Framework: Next.js 14 (App Router).
- Styling: Tailwind CSS + shadcn/ui.
- Charts: Recharts.
- Deployment: Vercel (frontend + API), Supabase (DB).

**Workers:**
- GitHub API ingestion worker (Node.js).
- Historical aggregation worker (Node.js).
- Scan job scheduler (cron via Vercel or GitHub Actions).

### 8.2 Core Data Model (v2)

```sql
-- repositories
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

-- weekly_metrics (aggregated from API scans)
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

-- daily_snapshots (observed history)
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

-- scan_jobs (queue management)
CREATE TABLE scan_jobs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  repo_id UUID REFERENCES repositories(id) ON DELETE CASCADE,
  status TEXT NOT NULL, -- pending, running, completed, failed
  created_at TIMESTAMPTZ DEFAULT NOW(),
  completed_at TIMESTAMPTZ,
  error_message TEXT,
  api_calls_used INTEGER DEFAULT 0,
  cache_hit BOOLEAN DEFAULT FALSE
);

CREATE INDEX idx_scan_jobs_status ON scan_jobs(status);
CREATE INDEX idx_scan_jobs_repo ON scan_jobs(repo_id);

-- scan_cache (raw API responses, 24h TTL)
CREATE TABLE scan_cache (
  key TEXT PRIMARY KEY,
  value JSONB NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  expires_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_scan_cache_expires ON scan_cache(expires_at);
```

### 8.3 API Endpoints (MVP)

```
POST   /api/scans              # Trigger new scan (returns job id)
GET    /api/scans/:id          # Get scan status/result
GET    /api/repos/:owner/:repo # Get repo summary + latest metrics
GET    /api/repos/:owner/:repo/timeline # Get weekly timeline
GET    /api/examples           # Get hardcoded example reports
POST   /api/disputes           # Maintainer dispute/correction request
GET    /api/methodology        # Get methodology documentation
```

### 8.4 Caching Strategy

**Layers:**
1. **API Response Cache** (24h TTL): Cache raw GitHub API responses by URL.
2. **Report Cache** (1h TTL): Cache rendered JSON reports for hot repos.
3. **CDN Cache** (1h TTL): Vercel/Cloudflare cache for static report pages.

**Cache Keys:**
- API: `github:GET:/repos/{owner}/{repo}/commits?page={n}`
- Report: `report:{owner}:{repo}:{window}`

**Invalidation:**
- New scan invalidates report cache for that repo.
- API cache invalidates on 429 or schema change.

### 8.5 Rate Limit Strategy

**Per-User Limits:**
- Unauthenticated: 5 scans/hour.
- Authenticated (future): 20 scans/hour.

**Global Limits:**
- Max 100 API req/min to GitHub.
- Max 10 concurrent scan jobs.

**Implementation:**
- Vercel middleware for API rate limiting.
- Redis counter for scan frequency.

---

## 9. MVP Scope (v2)

### 9.1 MVP In Scope

- Paste public GitHub repo URL.
- API-only scan (no git clone).
- Repository summary card.
- Weekly historical timeline (commits, PRs, issues, releases).
- Acceleration analysis (30d vs 90d, 90d vs 12mo).
- Contributor leverage metrics (with bot exclusion).
- Per-metric confidence labels (with operational definitions).
- Report page with shareable URL.
- Hardcoded example reports (5-10 repos).
- Maintainer dispute/correction form.
- Methodology page.

### 9.2 MVP Out of Scope (Moved to V1)

- Private repositories.
- Full code clone / LOC analysis.
- Test-to-code ratio.
- Dependency churn.
- CI/check trend analysis.
- README badge.
- Public ranked leaderboards.
- Dynamic "Recently Scanned" index.
- AI/vibe signal detection.
- GitHub Action self-reporting.
- Organization-level dashboards.
- Review/comment metrics.
- Governance score.

### 9.3 MVP Success Criteria

- A user can scan a small/medium repo and get a report in under 3 minutes.
- Reports explain at least one non-obvious insight (e.g., acceleration window).
- Charts show time-series changes, not only static totals.
- Confidence labels are visible and explain why that level was assigned.
- Maintainer can submit a dispute request from report page.
- System handles rate limits gracefully (no crashes, clear error messages).

---

## 10. Execution Plan (v2)

### Phase 1: Prototype (Week 1-2)

**Goal:** Prove reconstruction of acceleration timeline from GitHub API.

**Deliverables:**
- CLI tool (Node.js) to scan one repo via GitHub API (GraphQL preferred).
- Weekly timeline for commits, PRs, issues, releases.
- Basic acceleration calculation (30d vs 90d).
- JSON report generation (no UI yet).
- No database; JSON file storage.

**Success Metric:** Can scan a medium-sized repo (e.g., `vercel/next.js`) and produce a timeline with acceleration insight in under 5 minutes.

### Phase 2: MVP Backend (Week 3-4)

**Goal:** Persistent storage, API, caching, queue.

**Deliverables:**
- Node.js backend with Hono/Express.
- PostgreSQL schema + migrations (Supabase).
- Scan job queue (PostgreSQL-based).
- API response cache (Redis or in-memory).
- API endpoints for scan, timeline, summary.
- Rate limiting middleware.

**Success Metric:** Can trigger scan via API, store results, retrieve timeline. Handles rate limits gracefully.

### Phase 3: MVP Frontend (Week 5-6)

**Goal:** Public-facing report pages.

**Deliverables:**
- Next.js 14 app (Vercel).
- Repo input form.
- Report page with timeline charts.
- Acceleration view.
- Contributor leverage view.
- Confidence labels with tooltips.
- Example reports page (hardcoded).
- Methodology page.

**Success Metric:** User can paste repo URL, see interactive report in browser.

### Phase 4: Polish + Launch (Week 7-8)

**Goal:** Trust/safety, error handling, launch prep.

**Deliverables:**
- Maintainer dispute form + email integration.
- Error handling (API failures, rate limits, timeouts).
- Performance optimization (caching, query optimization).
- Launch landing page.
- Social card generation for reports.

**Success Metric:** Ready for public launch with 5-10 example reports.

---

## 11. Risks & Mitigations (v2)

### 11.1 GitHub API Rate Limits

**Risk:** Large repos require many paginated requests; could exhaust hourly quota.

**Mitigation:**
- Use authenticated API (bot account) for backend scans.
- GraphQL as primary query method (reduces call count).
- Throttle to 100 req/min.
- Cache raw API responses for 24h.
- Tiered scan SLA (large repos async).

### 11.2 Squash Merge / Rebase Distortion

**Risk:** Commit history may hide PR development activity.

**Mitigation:**
- Prefer PR timelines where available.
- Show confidence note: "Commit history may be squashed."
- Compare commit-based and PR-based velocity.

### 11.3 Missing Data / Incomplete History

**Risk:** Some repos have disabled issues, no releases, or limited metadata.

**Mitigation:**
- Confidence labels reflect data completeness.
- Show "Unavailable" instead of low score for missing data.
- Explain what data is missing in UI.

### 11.4 Bot Activity Distortion

**Risk:** Dependabot, Renovate, release bots inflate contributor counts.

**Mitigation:**
- Exclude known bot accounts from human leverage metrics.
- Show bot activity separately if significant.
- List excluded bots in report.

### 11.5 Misinterpretation of Metrics

**Risk:** Users treat "high acceleration" as inherently good or bad.

**Mitigation:**
- Use neutral language ("acceleration" not "growth").
- Add interpretation examples.
- Provide decision frameworks in methodology.
- Prominent caveats on every metric card.

### 11.6 Maintainer Disputes

**Risk:** Maintainers object to reports or claim data is wrong.

**Mitigation:**
- Dispute form on every report page.
- 48-hour response SLA.
- Documented correction workflow.
- Willingness to annotate or remove reports.

---

## 12. Trust & Safety (v2)

### 12.1 Principles

- Do not shame or roast projects.
- Do not claim AI authorship without evidence.
- Do not expose private repo data.
- Allow maintainers to annotate/dispute reports.

### 12.2 Dispute Process

**How to Dispute:**
1. Click "Report an Issue" on report page.
2. Fill form: what's wrong, evidence (optional), contact email.
3. VibeStats team reviews within 48 hours.
4. If valid: correct data, add annotation, or remove report.
5. Respond to submitter with outcome.

**Contact:** `hello@vibestats.io` (or GitHub issues for MVP).

### 12.3 Takedown Policy

- Maintainers can request full report removal.
- No questions asked for repo owner requests.
- Removal within 48 hours.
- Re-scan prevention (blocklist for removed repos).

---

## 13. Open Questions

- Should MVP require GitHub login for higher rate limits? (Decision: No for MVP, authenticated bot account on backend.)
- What is the minimum repo age/history needed for acceleration metric? (Decision: 3 months minimum for 30d vs 90d comparison.)
- How should monorepos be handled? (Decision: Treat as single repo for MVP; note in methodology.)
- Should public reports exclude repos below a confidence threshold? (Decision: Yes, show "Insufficient data" for Low confidence.)
- What default timeframe should be used for top-line acceleration? (Decision: 30d vs 90d for MVP.)

---

## 14. Appendix: Metric Formulas (v2)

### Acceleration Multiplier

```
acceleration = (velocity_last_30d) / (median_velocity_prior_90d)

where:
  velocity_last_30d = commits + prs_merged in last 30 days
  median_velocity_prior_90d = median weekly velocity in prior 90 days
```

### Human Leverage

```
leverage = (commits_last_30d) / (active_contributors_last_30d)
top_contributor_share = (commits_by_top_author) / (total_commits)

where:
  active_contributors = unique commit/PR authors (excluding bots)
```

### Confidence Level

```
if (history_months > 12 AND data_points > 100 AND missing_pct < 10):
  confidence = "High"
elif (history_months > 6 OR data_points > 50 OR missing_pct < 30):
  confidence = "Medium"
elif (history_months < 6 AND data_points < 50 AND missing_pct > 30):
  confidence = "Low"
else:
  confidence = "Unavailable"
```

---

## 15. Changelog

### v2 (2026-06-07)

**Changes from v1:**
- Removed review/comment metrics (moved to V1).
- Removed GitHub Action self-reporting (moved to V1).
- Removed dynamic "Recently Scanned" index (replaced with hardcoded examples).
- Added operational definitions for confidence labels.
- Replaced 60-second SLA with tiered scan time expectations.
- Switched runtime from Bun to Node.js 20 LTS.
- Added rate limit strategy and caching layer.
- Added maintainer dispute process.
- Made GraphQL primary (not optional) for historical reconstruction.
- Added bot detection/exclusion.
- Added database indexes and partitioning notes.

---

**End of PRD v2**
