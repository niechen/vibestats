# VibeStats PRD v1

**Status:** Draft  
**Owner:** Chen  
**Date:** 2026-06-07  
**Version:** 1.0 (MVP Scope)

---

## 1. Executive Summary

VibeStats measures how fast open-source projects are really being built in the AI coding era. It scans any public GitHub repository and reconstructs historical engineering activity from commits, pull requests, reviews, releases, issues, and repository metadata.

**Core thesis:** The product is not an AI-code detector. It is **repo velocity intelligence**: how fast a project is changing, how many humans are involved, whether governance keeps pace, and where the project shows unusual acceleration.

**MVP wedge:** Reconstruct and explain repository acceleration over time from public GitHub metadata, without overclaiming. Prove that VibeStats can produce trustworthy historical acceleration timelines before adding deep file analysis, public rankings, or AI/vibe signals.

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

### 2.4 Differentiation

| Existing Tools | Focus | VibeStats Focus |
|----------------|-------|-----------------|
| vibereport.dev, slopcodemonitor.ai, codeorigin.dev | AI-code provenance detection | Longitudinal engineering behavior: acceleration, leverage, governance coverage |
| Star History, GStars, GitPulse, OSS Insight | Star growth, activity dashboards | Acceleration relative to repo's own baseline, with explicit confidence labels |
| OpenSSF Scorecard, CHAOSS | Security/health metrics | Velocity + governance + contributor leverage over time |
| GitHub Insights/Pulse | Built-in repo stats | Cross-repo comparison, reconstructed history, shareable reports |

**Key differentiators:**
- Acceleration over time (not static totals).
- Velocity per contributor (human leverage).
- Review/governance density (process keeping pace?).
- Reconstructed historical timelines + observed future snapshots.
- Clear confidence labels for every metric.
- Decentralized data ingestion via GitHub Actions.

---

## 3. Target Users

### 3.1 Open-Source Users (Primary)

**Who:** Developers evaluating whether to adopt a dependency or contribute to a project.

**Jobs:**
- Check whether a repo is moving quickly or abandoned.
- Understand whether fast growth is matched by review and tests.
- Compare alternatives before adopting a dependency.

### 3.2 Maintainers (Secondary)

**Who:** Project maintainers who want a public health page for their repository.

**Jobs:**
- Show project momentum to users/investors.
- Detect review bottlenecks.
- Identify acceleration periods and governance gaps.
- Add a badge to README (V1).

### 3.3 Investors, Researchers, Tool Builders (Tertiary)

**Who:** People tracking AI-era software production patterns.

**Jobs:**
- Find one-person or small-team projects shipping unusually fast.
- Study how AI coding changes open-source development velocity.
- Build lists and indexes of high-acceleration repositories.

### 3.4 Security and Platform Teams (Future)

**Who:** Teams deciding whether to trust or adopt fast-moving dependencies.

**Jobs:**
- Identify high-velocity, low-review repositories.
- Track quality drift.
- Surface dependency and CI hygiene risks before adoption.

---

## 4. User Stories

### MVP Stories

- As a developer, I can paste any public GitHub repository URL and get a historical velocity report.
- As a maintainer, I can see whether my review process is keeping pace with project acceleration.
- As a user evaluating a dependency, I can see whether recent development is healthy or chaotic.
- As a researcher, I can browse recently scanned repositories with acceleration timelines.
- As a maintainer, I can install a GitHub Action to automatically report my project's stats to VibeStats.

### V1 Stories (Post-MVP)

- As a maintainer, I can add a public badge showing velocity, governance, and confidence to my README.
- As a researcher, I can filter fastest-accelerating repos by language, category, and timeframe.
- As a platform team, I can compare a repository against similar repos before approving it.
- As a user, I can watch a repo and get notified when acceleration changes sharply.

---

## 5. Core Concepts & Metrics

### 5.1 Velocity

**Definition:** Engineering activity volume over a time window.

**MVP Components (GitHub API-only):**
- Commits (count, unique authors).
- Pull requests opened.
- Pull requests merged.
- Issues opened/closed.
- Releases published.
- Reviews submitted.
- Review comments posted.
- Active GitHub actors (unique logins with any activity).

**Out of Scope for MVP:**
- LOC added/deleted (requires per-commit detail or git clone).
- Files changed (requires per-commit/detail API calls).

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

**Interpretation:**
- "2 contributors produced 150 commits in 30 days (75 commits/contributor)."
- "Top contributor accounts for 80% of activity."

### 5.4 Governance Coverage (Not "Score")

**Definition:** Whether collaboration and review keep pace with code velocity.

**MVP Components:**
- Reviews per merged PR (average).
- Review comments per merged PR (average).
- Share of merged PRs with at least one non-author review.
- Self-merge ratio (% of PRs merged by author).
- Median time from PR open to merge.

**Important Caveats:**
- Some projects review outside GitHub (mailing lists, Discord, internal tools).
- Low GitHub review density ≠ poor governance.
- MVP labels this as "GitHub review coverage" with confidence notes.

### 5.5 Confidence Labels (Per-Metric)

**Definition:** Reliability and completeness of each metric.

**MVP Levels:**
- **High:** Full data available, large sample size.
- **Medium:** Partial data or small sample size.
- **Low:** Significant gaps (e.g., reviews disabled, short history).
- **Unavailable:** Data not accessible via GitHub API.

**Per-Metric Examples:**
- Commit timeline: High (if repo age > 6 months).
- Review coverage: Low (if project uses external review).
- Acceleration: Medium (if < 12 months history).

---

## 6. Primary Product Surface

### 6.1 Repository Report Page

**Input:** GitHub owner/repo URL or shorthand (e.g., `niechen/vibestats`).

**Top Summary Card:**
- Repository name, description, language, stars, forks, age.
- **Acceleration:** "3.1x above baseline (last 30d vs prior 90d)."
- **Human Leverage:** "2 contributors, 75 commits/contributor."
- **Governance Coverage:** "0.4 reviews/PR, 60% self-merge."
- **Confidence:** "Medium (9 months history, reviews enabled)."
- **One-sentence interpretation:** "This repo is moving 3.1x faster than its historical baseline, driven by 2 active contributors, while GitHub review coverage remains low."

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
- Reviews (if available).
- Review comments (if available).

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

**Interpretations:**
- "Velocity increased but contributor count stayed at 2."
- "New contributors joined in March; output diversified."

### 6.5 Governance Coverage View

**Purpose:** Show whether review activity keeps pace with code velocity.

**Metrics:**
- Reviews per merged PR (over time).
- Review comments per PR (over time).
- Self-merge ratio (over time).
- Median PR open-to-merge time (over time).

**Caveats UI:**
- "This project may use external review tools."
- "Low GitHub review data does not imply poor governance."

### 6.6 Public Index Pages (MVP: Minimal)

**MVP Approach:** No ranked leaderboards. Show "Recently Scanned Repos" as examples.

**V1 Plans:**
- Fastest-accelerating repos this week (with confidence threshold).
- One-person repos shipping like teams.
- High-velocity, high-governance repos.
- Filter by language, stars, repo age.

---

## 7. Data Sources & Scan Modes

### 7.1 GitHub REST API (MVP Primary)

**Use For:**
- Repository metadata.
- Commits list (with pagination).
- Pull requests list.
- Issues list.
- Releases list.
- Reviews and review comments (per-PR).
- Contributors list.

**Limitations:**
- Rate limits for unauthenticated requests.
- Per-commit additions/deletions require N+1 calls.
- Review data requires per-PR requests (expensive).

### 7.2 GitHub GraphQL API (MVP Optional)

**Use For:**
- Efficient timeline queries.
- Pagination-heavy historical reconstruction.
- Search/index candidate discovery.

### 7.3 GitHub Actions (Decentralized Ingestion)

**Purpose:** Allow maintainers to voluntarily report stats to VibeStats.

**Action Behavior:**
- Runs on schedule (e.g., weekly) or on release.
- Collects local git stats (commits, PRs, LOC if desired).
- Posts anonymized metrics to VibeStats API.
- Opt-in, self-hosted by repo maintainers.

**Benefits:**
- Reduces VibeStats API rate limit burden.
- Enables deeper stats without cloning on server.
- Decentralized, low-cost data ingestion.

### 7.4 Observed Snapshots

**Purpose:** Store values that cannot be perfectly reconstructed historically.

**Examples:**
- Star count over time.
- Fork count over time.
- Branch protection details.
- Score/report snapshots.

**MVP Approach:**
- Store daily snapshot after first scan.
- Label charts as "Observed" vs "Reconstructed."

---

## 8. History Model

### 8.1 Reconstructed History

**Definition:** History inferred from existing GitHub events and git history.

**Examples:**
- Commit dates.
- PR open/merge dates.
- Review dates.
- Issue dates.
- Release dates.

**Confidence:** High for timestamps, medium for completeness.

### 8.2 Observed History

**Definition:** History collected by VibeStats after the first scan.

**Examples:**
- Star count snapshots.
- Fork count snapshots.
- Report score snapshots.

**UI Requirement:** Label whether a chart is based on reconstructed history, observed history, or a mixture.

---

## 9. MVP Scope

### 9.1 MVP In Scope

- Paste public GitHub repo URL.
- API-only quick scan (no git clone by default).
- Repository summary card.
- Weekly historical timeline (commits, PRs, issues, releases, reviews).
- Acceleration analysis (30d vs 90d, 90d vs 12mo).
- Contributor leverage metrics.
- Governance coverage metrics (with caveats).
- Per-metric confidence labels.
- Report page with shareable URL.
- Store scan result and daily snapshot.
- Public "Recently Scanned" list (no rankings).
- GitHub Action for self-reporting (basic version).

### 9.2 MVP Out of Scope

- Private repositories.
- Full code clone for every repo.
- LOC/file-level history.
- Test-to-code ratio.
- Dependency churn.
- CI/check trend analysis.
- README badge.
- Public ranked leaderboards.
- AI/vibe signal detection.
- Organization-level dashboards.
- Paid team accounts.

### 9.3 MVP Success Criteria

- A user can scan a public repo and get a useful report in under 60 seconds for small/medium repos.
- Reports explain at least one non-obvious insight from history (e.g., acceleration window).
- Charts show time-series changes, not only static totals.
- The system distinguishes reconstructed vs observed history.
- The product can show a list of recently scanned repos with working reports.

---

## 10. Technical Architecture

### 10.1 Stack Choices

**Backend:**
- Runtime: **Bun** (as requested by user).
- Framework: Hono or Elysia (lightweight, Bun-native).
- Database: PostgreSQL (Supabase or self-hosted).
- Queue: BullMQ (Redis) or PostgreSQL-based queue.
- Object Storage: S3-compatible (R2, Supabase Storage) for raw scan payloads.

**Frontend:**
- Framework: Next.js or Remix (SSR for report pages).
- Styling: Tailwind CSS + shadcn/ui.
- Charts: Recharts or ECharts.
- Deployment: Vercel, Cloudflare Pages, or self-hosted.

**Workers:**
- GitHub API ingestion worker (Bun).
- Historical aggregation worker (Bun).
- Snapshot scheduler (cron).
- GitHub Action runner (separate repo).

### 10.2 Core Data Model (Simplified for MVP)

```sql
-- repositories
CREATE TABLE repositories (
  id UUID PRIMARY KEY,
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

-- weekly_metrics (aggregated from API scans)
CREATE TABLE weekly_metrics (
  repo_id UUID REFERENCES repositories(id),
  week_start DATE NOT NULL,
  commits INTEGER,
  prs_opened INTEGER,
  prs_merged INTEGER,
  issues_opened INTEGER,
  issues_closed INTEGER,
  releases INTEGER,
  reviews INTEGER,
  review_comments INTEGER,
  active_contributors INTEGER,
  PRIMARY KEY (repo_id, week_start)
);

-- daily_snapshots (observed history)
CREATE TABLE daily_snapshots (
  repo_id UUID REFERENCES repositories(id),
  observed_at DATE NOT NULL,
  stars INTEGER,
  forks INTEGER,
  open_issues INTEGER,
  scores_json JSONB,
  PRIMARY KEY (repo_id, observed_at)
);

-- scan_jobs (queue management)
CREATE TABLE scan_jobs (
  id UUID PRIMARY KEY,
  repo_id UUID REFERENCES repositories(id),
  status TEXT, -- pending, running, completed, failed
  created_at TIMESTAMPTZ,
  completed_at TIMESTAMPTZ,
  error_message TEXT
);
```

### 10.3 API Endpoints (MVP)

```
POST   /api/scans              # Trigger new scan
GET    /api/scans/:id          # Get scan status/result
GET    /api/repos/:owner/:repo # Get repo summary + latest metrics
GET    /api/repos/:owner/:repo/timeline # Get weekly timeline
GET    /api/index/recent       # Get recently scanned repos
POST   /api/actions/report     # GitHub Action webhook for self-reporting
```

### 10.4 GitHub Action Design

**Action Repo:** `vibestats/action`

**Inputs:**
- `vibestats-api-url`: VibeStats API endpoint.
- `vibestats-api-key`: Optional auth key.
- `report-interval`: Schedule (e.g., weekly, on-release).

**Action Steps:**
1. Checkout repo.
2. Collect local stats (git log, PR data via API if available).
3. Anonymize/aggregates metrics.
4. POST to VibeStats `/api/actions/report`.

**Example Workflow:**
```yaml
name: VibeStats Report
on:
  schedule:
    - cron: '0 0 * * 0'  # Weekly
jobs:
  report:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: vibestats/action@v1
        with:
          vibestats-api-url: https://vibestats.io/api/actions/report
```

---

## 11. Execution Plan

### Phase 1: Prototype (Week 1-2)

**Goal:** Prove reconstruction of acceleration timeline from GitHub API.

**Deliverables:**
- CLI tool to scan one repo via GitHub API.
- Weekly timeline for commits, PRs, issues, releases.
- Basic acceleration calculation (30d vs 90d).
- Static HTML report generation.
- No database yet; JSON file storage.

**Success Metric:** Can scan a medium-sized repo (e.g., `vercel/next.js`) and produce a timeline with acceleration insight.

### Phase 2: MVP Backend (Week 3-4)

**Goal:** Persistent storage, API, queue system.

**Deliverables:**
- Bun backend with Hono/Elysia.
- PostgreSQL schema + migrations.
- Scan job queue (BullMQ or PG-based).
- API endpoints for scan, timeline, summary.
- Daily snapshot scheduler.

**Success Metric:** Can trigger scan via API, store results, retrieve timeline.

### Phase 3: MVP Frontend (Week 5-6)

**Goal:** Public-facing report pages.

**Deliverables:**
- Next.js/Remix app.
- Repo input form.
- Report page with timeline charts.
- Acceleration view.
- Contributor leverage view.
- Governance coverage view.
- Recently scanned list.

**Success Metric:** User can paste repo URL, see interactive report in browser.

### Phase 4: GitHub Action + Polish (Week 7-8)

**Goal:** Decentralized ingestion, method docs, launch prep.

**Deliverables:**
- GitHub Action for self-reporting.
- Methodology page (explain metrics, confidence, limitations).
- Error handling, rate limit handling.
- Performance optimization (caching, pagination).
- Launch landing page.

**Success Metric:** Maintainer can install action, VibeStats receives automated reports.

---

## 12. Risks & Mitigations

### 12.1 GitHub API Rate Limits

**Risk:** Large repos require many paginated requests; unauthenticated limits are low.

**Mitigation:**
- Use authenticated API (bot account) for backend scans.
- Progressive enrichment: quick scan first, deep scan queued.
- Encourage GitHub Action self-reporting.
- Cache raw API responses.

### 12.2 Squash Merge / Rebase Distortion

**Risk:** Commit history may hide PR development activity.

**Mitigation:**
- Prefer PR timelines where available.
- Show confidence notes: "Commit history may be squashed."
- Compare commit-based and PR-based velocity.

### 12.3 Missing Review Data

**Risk:** Some repos do not use GitHub PRs or reviews.

**Mitigation:**
- Governance coverage shows "Unavailable" instead of low score.
- Confidence score explains missing review data.
- Add caveat: "This project may use external review tools."

### 12.4 Bot Activity Distortion

**Risk:** Dependabot, Renovate, release bots inflate contributor counts.

**Mitigation:**
- Detect known bot accounts.
- Exclude bot activity from "human leverage" metrics.
- Show bot activity separately if significant.

### 12.5 Misinterpretation of Metrics

**Risk:** Users treat "low governance coverage" as "bad project."

**Mitigation:**
- Use "coverage" not "score."
- Add explicit caveats in UI.
- Avoid public rankings with negative labels in MVP.
- Publish methodology page explaining limitations.

---

## 13. Trust & Safety

### 13.1 Principles

- Do not shame or roast projects.
- Do not claim AI authorship without evidence.
- Do not expose private repo data.
- Allow maintainers to annotate/dispute reports (V1).

### 13.2 Takedown Policy

- Maintainers can request report removal or annotation.
- Clear contact method for disputes.
- Documented process for corrections.

---

## 14. Open Questions

- Should MVP require GitHub login for higher rate limits?
- What is the minimum repo age/history needed for acceleration metric?
- How should monorepos be handled?
- Should public reports exclude repos below a confidence threshold?
- What default timeframe should be used for top-line acceleration?

---

## 15. Appendix: Metric Formulas (MVP)

### Acceleration Multiplier

```
acceleration = (velocity_last_30d) / (median_velocity_prior_90d)
```

Where:
- `velocity_last_30d` = commits + prs_merged in last 30 days.
- `median_velocity_prior_90d` = median weekly velocity in prior 90 days.

### Human Leverage

```
leverage = (commits_last_30d) / (active_contributors_last_30d)
top_contributor_share = (commits_by_top_author) / (total_commits)
```

### Governance Coverage

```
reviews_per_pr = (total_reviews) / (prs_merged)
self_merge_ratio = (prs_merged_by_author) / (prs_merged)
```

---

**End of PRD v1**
