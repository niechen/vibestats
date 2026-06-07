# Vibe Stats PRD

Status: Draft
Owner: Chen
Date: 2026-06-07

## 1. Summary

Vibe Stats is a public web product for measuring how open-source GitHub repositories accelerate over time in the AI coding era. It scans any public GitHub repository and reconstructs historical engineering activity from commits, pull requests, reviews, releases, issues, and repository metadata. The product's core is not AI-code detection. The core is serious repo velocity intelligence: how fast a project is changing, how many humans are involved, whether governance keeps pace, and where the project shows unusual acceleration.

The product should help users answer:

- Is this repository accelerating faster than its historical baseline?
- Is the acceleration driven by more contributors, higher per-contributor output, or large bursts from a small team?
- Are reviews, tests, CI, and release discipline keeping pace with the speed?
- Is the observed growth healthy engineering velocity or risky ungoverned churn?
- How does this repo compare with similar open-source projects?

## 2. Product Positioning

### 2.1 Primary Positioning

Measure how fast open-source projects are really being built in the AI coding era.

### 2.2 Product Category

Repo velocity intelligence for open-source software.

### 2.3 What It Is Not

- Not a black-box AI-generated-code detector.
- Not a moral ranking of whether a project is "vibe coded".
- Not a replacement for code review, security scanning, or dependency auditing.
- Not a generic GitHub stats dashboard that only shows commits, stars, and contributors.

### 2.4 Differentiation

Existing AI-code scanners tend to focus on provenance signals such as commit signatures, co-authored-by trailers, AI-tool emails, or code-generation patterns. Vibe Stats should focus on longitudinal engineering behavior:

- Acceleration over time.
- Velocity per contributor.
- Review and governance density.
- Quality guardrail drift.
- Reconstructed historical timelines plus observed future snapshots.
- Clear confidence labels for every metric.

## 3. Target Users

### 3.1 Open-Source Users

Developers evaluating whether an open-source project is active, healthy, and responsibly maintained.

Jobs:

- Check whether a repo is moving quickly or abandoned.
- Understand whether fast growth is matched by review and tests.
- Compare alternatives before adopting a dependency.

### 3.2 Maintainers

Maintainers who want a public health page for their repository.

Jobs:

- Show project momentum.
- Detect review bottlenecks.
- Identify acceleration periods and governance gaps.
- Add a badge to README.

### 3.3 Investors, Researchers, and Tool Builders

People tracking AI-era software production patterns.

Jobs:

- Find one-person or small-team projects shipping unusually fast.
- Study how AI coding changes open-source development velocity.
- Build lists and indexes of high-acceleration repositories.

### 3.4 Security and Platform Teams

Teams deciding whether to trust or adopt fast-moving dependencies.

Jobs:

- Identify high-velocity, low-review repositories.
- Track quality drift.
- Surface dependency and CI hygiene risks before adoption.

## 4. User Stories

- As a developer, I can paste any public GitHub repository URL and get a historical velocity report.
- As a maintainer, I can see whether my review process is keeping pace with project acceleration.
- As a user evaluating a dependency, I can see whether recent development is healthy or chaotic.
- As a researcher, I can browse the fastest-accelerating open-source repositories by language, category, and timeframe.
- As a maintainer, I can add a public badge showing velocity, governance, and confidence.
- As a platform team, I can compare a repository against similar repos before approving it.

## 5. Core Concepts

### 5.1 Velocity

Velocity measures engineering activity over a time window. It is not just commit count. It combines code changes, PR activity, release activity, and issue/maintenance activity.

Candidate components:

- Commits.
- Pull requests opened and merged.
- Files changed.
- Lines added, deleted, and net changed.
- Releases.
- Issues opened and closed.
- Review comments.

### 5.2 Acceleration

Acceleration measures change in velocity relative to the repository's own historical baseline.

Examples:

- Last 14 days vs previous 90-day median.
- Last 30 days vs same-length prior window.
- Rolling 7-day and 30-day moving average slopes.
- Burst detection for sudden spikes in commits, PRs, or LOC churn.

### 5.3 Human Density

Human density normalizes velocity by active contributors.

Candidate components:

- Active contributors per week or month.
- Commits per active contributor.
- Merged PRs per active contributor.
- LOC churn per active contributor.
- Share of output from top contributor.
- New contributors entering the project.

### 5.4 Governance Density

Governance density measures whether collaboration and review keep pace with code velocity.

Candidate components:

- Reviews per merged PR.
- Review comments per merged PR.
- Share of merged PRs with at least one non-author review.
- Self-merge ratio.
- Median time from PR open to merge.
- CI checks present and passing before merge, where available.
- Large PRs merged without review.

### 5.5 Quality Guardrails

Quality guardrails measure whether the repository has engineering discipline around fast change.

Candidate components:

- Test files added or modified alongside source changes.
- CI configuration presence.
- Lint/typecheck configuration presence.
- Dependency count and lockfile churn.
- Release cadence.
- Large generated/scaffold files.
- Secret-like files or unsafe committed artifacts.

### 5.6 AI/Vibe Signals

AI/vibe signals are secondary and should be confidence-labeled.

Candidate components:

- Co-authored-by trailers from known AI tools.
- AI-tool email patterns.
- Commit message signatures from tools such as Claude Code, Cursor, Aider, Codex CLI, Copilot, Gemini CLI.
- Extremely high acceleration from very few contributors.
- Large scaffold-like bursts.

Important: AI/vibe signals are not proof of AI-generated code. They are weak-to-medium provenance and behavior signals.

## 6. Primary Product Surface

### 6.1 Repository Report Page

Input:

- GitHub owner/repo URL or shorthand.

Top summary:

- Repository name, description, language, stars, forks, age.
- Velocity score.
- Acceleration score.
- Governance score.
- Sustainability score.
- Confidence score.
- One-sentence interpretation.

Example interpretation:

This repo is moving 4.8x faster than its historical baseline, driven by 2 active contributors, while review density has fallen 62%.

### 6.2 Historical Timeline

The timeline is the main interaction surface.

Required views:

- Daily.
- Weekly.
- Monthly.
- Last 30 days.
- Last 90 days.
- Last 12 months.
- All time.

Required series:

- Commits.
- PRs opened.
- PRs merged.
- Active contributors.
- LOC added.
- LOC deleted.
- Files changed.
- Review count.
- Review comments.
- Releases.
- Issues opened/closed.

### 6.3 Acceleration View

Purpose:

- Show when project velocity changed sharply.

Required visualizations:

- Rolling velocity line.
- Historical baseline band.
- Acceleration multiplier.
- Burst markers.
- Peak acceleration windows.

Required explanations:

- "Last 14 days are 6.2x above historical median."
- "This acceleration is mostly from one contributor."
- "PR review density did not increase during the acceleration window."

### 6.4 Velocity vs Governance View

Purpose:

- Show whether fast development is disciplined.

Required metrics:

- Velocity over time.
- Reviews per PR over time.
- Median PR size over time.
- Unreviewed merge ratio over time.
- Test-to-code change ratio over time.
- CI presence and failure trend where available.

Classification:

- Fast and governed.
- Fast but under-reviewed.
- Slow and governed.
- Slow and inactive.
- High churn risk.

### 6.5 Contributor Timeline

Purpose:

- Show whether acceleration comes from more people or higher leverage per person.

Required metrics:

- Active contributors over time.
- Top contributor share.
- New contributor count.
- Contributor concentration index.
- Commits per contributor.
- Merged PRs per contributor.
- LOC churn per contributor.

### 6.6 Quality Drift View

Purpose:

- Show whether code velocity is outpacing quality infrastructure.

Required metrics:

- Test file change ratio.
- CI config presence.
- Lint/typecheck config presence.
- Dependency file churn.
- Large PR ratio.
- Generated/scaffold file indicators.
- Secret-risk indicators.

### 6.7 Public Index Pages

Index pages create discovery and distribution.

Initial index ideas:

- Fastest-accelerating repos this week.
- One-person repos shipping like teams.
- High-velocity, low-review repos.
- Fast and well-governed repos.
- New repos that reached meaningful velocity quickly.
- AI-era open-source acceleration map by language.

Filters:

- Language.
- Stars.
- Repo age.
- Contributor count.
- Timeframe.
- Governance score.
- License.

## 7. Scoring Model

### 7.1 Principles

- Scores must be explainable.
- Scores must be decomposable.
- Scores must expose confidence.
- Scores must avoid claiming AI authorship without evidence.
- Scores must be robust against small repos and one-off spikes.

### 7.2 Top-Level Scores

Velocity Score:

- Measures recent activity volume.
- Uses commits, PRs, LOC churn, releases, and issue activity.
- Normalized by repo age and language/category where possible.

Acceleration Score:

- Measures recent activity relative to historical baseline.
- Uses rolling windows and burst detection.
- Penalizes insufficient history with lower confidence, not lower score.

Governance Score:

- Measures review, PR discipline, CI presence, and merge hygiene.
- Higher score means process keeps pace with velocity.

Sustainability Score:

- Measures whether speed is supported by tests, CI, dependency hygiene, and controlled churn.

Human Leverage Score:

- Measures output per active contributor and contributor concentration.
- High leverage is not automatically bad. It becomes risky when governance and sustainability are weak.

Confidence Score:

- Measures completeness and reliability of input data.
- Lower when repo has short history, squash-only history, disabled issues, missing PRs, missing checks, or inaccessible metadata.

### 7.3 Example Interpretation Rules

- High acceleration + high governance + high sustainability:
  AI-era fast, professionally governed.

- High acceleration + low governance:
  Accelerating faster than review capacity.

- High velocity + high contributor concentration:
  Small team or solo maintainer operating with unusually high leverage.

- High churn + low test ratio:
  Rapid change without matching quality guardrails.

- Low confidence:
  Treat scores as directional; not enough history or metadata.

## 8. Data Sources

### 8.1 GitHub REST API

Use for:

- Repository metadata.
- Commits.
- Pull requests.
- Issues.
- Releases.
- Contributors.
- Reviews and review comments.
- Check runs and workflow runs where available.

### 8.2 GitHub GraphQL API

Use for:

- Efficient timeline queries.
- PR review metadata.
- Pagination-heavy historical reconstruction.
- Search/index candidate discovery.

### 8.3 Git Clone / Archive Scan

Use when deeper analysis is needed:

- LOC by language.
- File-level history.
- Generated/scaffold file detection.
- Test/source file classification.
- Dependency file parsing.
- CI/lint/typecheck config detection.

MVP should avoid cloning by default. Deep scan can be queued after initial API-only report.

### 8.4 Observed Snapshots

Some values cannot be perfectly reconstructed historically.

Examples:

- Star count over time.
- Fork count over time.
- Current branch protection details over time.
- Some GitHub Actions/check metadata after retention windows.

The product should store daily or weekly snapshots after first scan.

## 9. History Model

### 9.1 Reconstructed History

History inferred from existing GitHub events and git history.

Examples:

- Commit dates.
- PR open/merge dates.
- Review dates.
- Issue dates.
- Release dates.
- File-level changes if git history is scanned.

### 9.2 Observed History

History collected by Vibe Stats after the first scan.

Examples:

- Star count snapshots.
- Fork count snapshots.
- Branch protection snapshots.
- Score snapshots.
- Cached report deltas.

### 9.3 UI Requirement

The UI must label whether a chart is based on reconstructed history, observed history, or a mixture.

## 10. MVP Scope

### 10.1 MVP In Scope

- Paste public GitHub repo URL.
- API-only scan.
- Repository summary.
- Weekly historical timeline.
- Commits, PRs, merged PRs, reviews, review comments, contributors, issues, releases.
- Acceleration score based on commits, PRs, and merged PRs.
- Governance score based on reviews and merge patterns.
- Basic confidence score.
- Report page with shareable URL.
- Store scan result and daily score snapshot.
- Public index for recently scanned repos.

### 10.2 MVP Out of Scope

- Private repositories.
- Full code clone for every repo.
- Full security scanning.
- Full AI-generated code detection.
- Organization-level dashboards.
- Paid team accounts.
- Browser extension.
- CI integration.

### 10.3 MVP Success Criteria

- A user can scan a public repo and get a useful report in under 60 seconds.
- Reports explain at least one non-obvious insight from history.
- Charts show time-series changes, not only static totals.
- The system distinguishes reconstructed vs observed history.
- The product can produce a public index of high-acceleration repos.

## 11. V1 Scope After MVP

- Deep scan queue with git clone.
- LOC and file-level history.
- Test-to-code ratio over time.
- Dependency churn.
- CI/check trend analysis.
- README badge.
- Language/category benchmarking.
- Saved watchlists.
- Webhook-based refresh.
- Export as JSON/CSV.
- Public methodology page.

## 12. UX Requirements

### 12.1 First Screen

The first screen should be a functional scanner, not a marketing landing page.

Required elements:

- Repo input.
- Recent example reports.
- Index links.
- Concise positioning line.

### 12.2 Report Page Layout

Suggested sections:

1. Summary and interpretation.
2. Historical timeline.
3. Acceleration analysis.
4. Velocity vs governance.
5. Contributor leverage.
6. Quality guardrails.
7. Data confidence and methodology.

### 12.3 Tone

The tone should be serious and analytical. It can use "vibe" for cultural relevance, but the methodology must read like engineering intelligence.

Avoid:

- Roasts.
- Shame language.
- Unsupported AI authorship claims.
- One-number black-box judgments.

Prefer:

- "Acceleration."
- "Governance."
- "Confidence."
- "Observed vs reconstructed."
- "Directional signal."

## 13. Technical Architecture

### 13.1 Suggested Stack

Frontend:

- Next.js or Remix.
- Tailwind or shadcn/ui.
- Recharts, ECharts, or Tremor for charts.

Backend:

- Node.js/TypeScript API workers.
- PostgreSQL for normalized scan data.
- Redis or queue system for scan jobs.
- Object storage for raw scan payloads and deep scan artifacts.

Workers:

- GitHub API ingestion worker.
- Historical aggregation worker.
- Deep git scan worker.
- Snapshot scheduler.

### 13.2 Core Tables

repositories:

- id
- owner
- name
- default_branch
- created_at
- first_scanned_at
- last_scanned_at
- stars_current
- forks_current
- primary_language

repo_snapshots:

- repo_id
- observed_at
- stars
- forks
- open_issues
- default_branch
- scores_json

commit_events:

- repo_id
- sha
- author_login
- committed_at
- additions
- deletions
- files_changed
- ai_signal_json

pull_request_events:

- repo_id
- number
- author_login
- opened_at
- merged_at
- closed_at
- additions
- deletions
- files_changed
- review_count
- review_comment_count
- merged_by_login

issue_events:

- repo_id
- number
- author_login
- opened_at
- closed_at

release_events:

- repo_id
- tag
- published_at

daily_repo_metrics:

- repo_id
- date
- commits
- prs_opened
- prs_merged
- reviews
- review_comments
- active_contributors
- additions
- deletions
- files_changed
- releases
- issues_opened
- issues_closed
- velocity_score
- acceleration_score
- governance_score
- sustainability_score
- confidence_score

### 13.3 API Endpoints

- POST /api/scans
- GET /api/scans/:id
- GET /api/repos/:owner/:repo
- GET /api/repos/:owner/:repo/timeline
- GET /api/repos/:owner/:repo/scores
- GET /api/index/accelerating
- GET /api/index/fast-governed
- GET /api/index/high-velocity-low-review

## 14. Data and Methodology Risks

### 14.1 GitHub API Rate Limits

Risk:

- Large repositories may require many paginated requests.

Mitigation:

- API-only quick scan first.
- Progressive enrichment.
- Cache raw payloads.
- Allow authenticated scans later.

### 14.2 Squash Merge and Rebase Distortion

Risk:

- Commit history may hide PR development activity.

Mitigation:

- Prefer PR timelines where available.
- Show confidence notes.
- Compare commit-based and PR-based velocity.

### 14.3 Missing Review Data

Risk:

- Some repos do not use GitHub PRs or reviews.

Mitigation:

- Governance score should reflect unavailable data separately from low governance.
- Confidence score should explain missing review data.

### 14.4 AI Signal Overclaiming

Risk:

- AI-code detection is unreliable.

Mitigation:

- Keep AI/vibe signal secondary.
- Label as provenance/behavior signals.
- Do not claim authorship.

### 14.5 Language and Repo-Type Bias

Risk:

- Generated projects, docs repos, monorepos, and vendored code can skew metrics.

Mitigation:

- Detect repo type.
- Exclude vendored/generated paths where possible.
- Benchmark by language and repo category.

## 15. Open Questions

- Should the first version require GitHub login for higher rate limits?
- Should deep git scans be opt-in or automatic after quick scan?
- What is the minimum repo age/history needed for acceleration score?
- How should monorepos be handled?
- Should public rankings exclude repos below a confidence threshold?
- Should maintainers be able to dispute or annotate a report?
- What default timeframe should be used for the top-level score?

## 16. Non-Goals

- Do not judge developer skill.
- Do not accuse maintainers of using AI.
- Do not expose private repo data.
- Do not make a joke product with serious-looking numbers.
- Do not collapse the report into one opaque "vibe score".

## 17. Launch Plan

### Phase 1: Prototype

- Scan one repo via GitHub API.
- Build weekly timeline.
- Compute basic acceleration and governance metrics.
- Render a static report page.

### Phase 2: MVP

- Public scanner.
- Persistent reports.
- Daily snapshots.
- Shareable report URLs.
- Methodology page.
- Small public index.

### Phase 3: Index and Distribution

- Scheduled scans for trending repos.
- Weekly acceleration leaderboard.
- README badges.
- Social cards for reports.

### Phase 4: Serious Adoption

- Deep scans.
- Benchmarks by language/category.
- Organization dashboards.
- CI/check integrations.
- API access.

## 18. Review Checklist

Reviewers should evaluate:

- Is the product differentiated from existing AI-code detectors?
- Is the MVP small enough to build quickly?
- Are the metrics explainable and defensible?
- Which metrics can be reconstructed from GitHub history and which require snapshots?
- Are there hidden data quality traps?
- Does the UX lead with time-series insight rather than static vanity stats?
- Is "vibe" useful positioning or too unserious for the target users?
