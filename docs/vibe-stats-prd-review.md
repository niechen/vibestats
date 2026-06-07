# Vibe Stats PRD Review

Date: 2026-06-07
Reviewed document: `docs/vibe-stats-prd.md`
Reviewers: Codex native subagent, Claude Code CLI

## Executive Summary

Both reviewers agree that the product direction is strong, but the PRD currently overreaches. The most important correction is to narrow MVP from a broad repo intelligence platform into one defensible wedge:

> Reconstruct and explain repository acceleration over time from public GitHub metadata, without overclaiming.

The first implementation should prove that the product can produce trustworthy historical acceleration timelines. Quality drift, deep file analysis, AI/vibe signals, public risk rankings, README badges, and full sustainability scoring should move to later versions.

## Shared High-Severity Findings

### 1. MVP Is Too Large For API-Only Under-60s Scans

The PRD promises an API-only scan that completes quickly while also requiring full historical commits, PRs, reviews, review comments, contributors, issues, releases, LOC churn, and governance scoring.

This is not realistic for large repositories. Cold-scanning repos like Kubernetes, TypeScript, Rust, Linux, Home Assistant, or other long-lived projects would require heavy pagination, expensive per-commit or per-PR requests, and likely exceed rate limits or latency targets.

Required PRD change:

- Define a scan budget.
- Define max lookback windows.
- Define partial report states.
- Define async deep scan behavior.
- Replace "under 60 seconds for any repo" with tiered scan modes.

### 2. Several Core Metrics Need Git Clone Or Deep Scan

The PRD includes metrics that cannot be reliably reconstructed from high-level GitHub API calls:

- LOC by day.
- Files changed across full history.
- Test-to-code ratio over time.
- Generated/scaffold detection.
- Dependency churn.
- Historical lint/typecheck/CI behavior.
- Per-commit additions/deletions/files_changed for large histories.

Required PRD change:

- Mark every metric as one of:
  - MVP GitHub metadata.
  - API enrichment.
  - Deep git scan.
  - Observed snapshot only.
- Move file-level quality drift and sustainability scoring out of MVP.

### 3. Public Scores Can Be Misinterpreted

Governance and velocity scores can accidentally shame or misclassify serious projects. Low GitHub review density does not always mean poor governance. Some projects review on mailing lists, Gerrit, Phabricator, Discord, Slack, private mirrors, or direct commit workflows.

Likewise, high review-comment count can mean healthy review, but it can also mean conflict or instability.

Required PRD change:

- Add workflow model detection.
- Suppress governance score when review data is not representative.
- Use "not enough GitHub review data" instead of low governance in those cases.
- Avoid public "high-velocity, low-review" rankings in MVP.

### 4. Confidence Must Be Per-Metric, Not One Score

A repository can have high confidence for PR timestamps, low confidence for review density, medium confidence for contributor identity, and no confidence for historical CI state.

Required PRD change:

- Replace single confidence score with per-metric confidence labels.
- Each chart should show source, lookback, completeness, and known blind spots.
- Store enough raw provenance to reproduce every derived metric.

### 5. Positioning Still Fights Itself

The PRD wants serious analytical credibility, but the "Vibe Stats" name and public index ideas can imply a playful or judgmental product. This is especially risky for maintainers, security teams, and platform teams.

Required PRD change:

- Decide whether "vibe" is the product brand or just launch/marketing language.
- If keeping the name, make methodology and UI especially sober.
- Avoid roast language, shame language, and unsupported AI authorship claims.

### 6. Public Index Should Not Be MVP

Public leaderboards amplify measurement errors. They require:

- Comparable scoring.
- Confidence thresholds.
- Bot handling.
- Repo category normalization.
- Maintainer dispute or annotation.
- Abuse and takedown process.
- Scheduled scanning infrastructure.

Required PRD change:

- Replace public ranked indexes in MVP with "recently scanned examples."
- Move serious ranking pages to V1 or later.

### 7. Bot And Identity Handling Is Missing

Many repos have high-volume bot activity from Dependabot, Renovate, github-actions, release bots, pre-commit bots, and increasingly AI-related GitHub Apps. Contributor counts and velocity can be badly distorted without identity rules.

Required PRD change:

- Add bot detection.
- Distinguish actor, author, committer, merger, and reviewer.
- Normalize user identity where possible.
- Store bot flags and raw identity provenance.

## Recommended MVP Rewrite

### MVP Wedge

Public GitHub repository acceleration timeline from reconstructable metadata.

### MVP In Scope

- Paste public GitHub repo URL.
- API-only quick scan.
- Default lookback: last 12 months plus prior 12-month baseline when available.
- Weekly timeline for:
  - Commits.
  - PRs opened.
  - PRs merged.
  - Issues opened/closed.
  - Releases.
  - Reviews where available.
  - Review comments where available.
  - Active GitHub actors.
- Acceleration windows.
- Contributor concentration.
- Review coverage as a metric, not a moral score.
- Per-chart confidence labels.
- Partial report states.
- Shareable report URL.
- Methodology page.

### MVP Out Of Scope

- Public rankings with negative labels.
- README badge.
- Deep file scan.
- LOC/test/dependency quality drift.
- Sustainability score.
- AI/vibe signal.
- Organization dashboards.
- Private repos.

### Suggested First Report Interpretation

Use decomposed statements instead of one opaque score:

- Activity increased 3.1x over the selected baseline.
- PR merge volume rose 2.4x.
- Active GitHub actors stayed flat at 2-3 people.
- Review coverage is unavailable or low-confidence for this repository.
- This report is based on GitHub metadata from the last 24 months.

## Required PRD Edits Before Implementation

1. Rename or reframe positioning around "repo acceleration intelligence."
2. Define MVP scan budget and partial scan states.
3. Add a metric source matrix.
4. Move deep scan metrics to V1.
5. Replace top-line score stack with decomposed insight cards.
6. Add per-metric confidence.
7. Add bot and identity handling.
8. Add known failure modes.
9. Add trust and safety policy for public reports.
10. Move public indexes and badges out of MVP.
11. Rework the data model around raw source events plus derived metrics.
12. Define acceleration methodology before UI implementation.

## Claude Review Notes

Claude's strongest concern was that the brand and analytical positioning conflict. It argued that "Vibe Stats" plus "AI coding era" makes users expect an AI-code detector or roast, while the PRD then repeatedly tries to undo that expectation. Claude also noted that the real competitive set should include OSS Insight, OpenSSF Scorecard, CHAOSS metrics, Repobeats, GitHub Pulse/Insights, and similar OSS analytics tools, not only AI-code scanners.

Claude also emphasized that "under 60 seconds for any public repo" conflicts with full historical review and commit analysis, and that several required fields such as per-commit additions, deletions, and files changed require N+1 GitHub calls or a git clone. It recommended cutting top-line scores to Acceleration plus Governance plus structured confidence, dropping AI/vibe signals from MVP, and adding trust/safety policies before public indexes.

## Codex Review Notes

Codex's strongest concern was that the MVP should prove one defensible thing: whether Vibe Stats can reconstruct and explain acceleration from GitHub metadata without overclaiming. It recommended narrowing the MVP to weekly acceleration timelines from commits, PRs, releases, issues, reviews, active GitHub actors, and caveats.

Codex also warned that governance scoring can be badly misread without workflow model detection, that confidence must be per metric rather than one score, and that public indexes should wait until confidence thresholds and maintainer annotation exist.

## Bottom Line

The product idea survives review, but the PRD should be tightened before implementation. The serious core should be:

> Historical repo acceleration, contributor leverage, and governance coverage from public GitHub metadata, with explicit confidence and methodology.

Everything that requires deep file analysis or public comparative judgment should come later.
