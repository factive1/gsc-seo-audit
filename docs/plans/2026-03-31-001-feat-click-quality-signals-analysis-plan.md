---
title: "feat: Add Click Quality Signals Analysis (Pass 5)"
type: feat
status: completed
date: 2026-03-31
---

# feat: Add Click Quality Signals Analysis (Pass 5)

## Overview

Add a new analysis pass to the GSC SEO Audit plugin that evaluates **page-level click quality** through the lens of Google's NavBoost algorithm framework. While Pass 2 operates at the **query level** (finding individual keyword CTR gaps and recommending title rewrites), Pass 5 operates at the **page level** — aggregating all queries per page into a holistic click health assessment with temporal trend analysis.

Based on findings from the Google antitrust trial, API leak, and patent US8661029B1, Google's NavBoost system evaluates five click signals: impressions, clicks, bad clicks (quick bounces), good clicks (extended engagement), and last longest clicks (complete satisfaction). GSC cannot directly measure the post-click signals (bad/good/last longest), so Phase 1 uses GSC-derived proxy indicators with honest confidence labeling. Phase 2 adds optional GA4 integration for true post-click measurement.

**Source:** [Zyppy — Google Click Signals](https://signal.zyppy.com/p/google-click-signals)

## Problem Statement

The current plugin has a gap between Pass 2 (query-level CTR optimization) and the broader question: **"Is this page generating the right kind of clicks?"** A page can have acceptable CTR on individual keywords but still exhibit poor click quality signals in aggregate — declining click velocity, over-dependence on a single query, volatile positions suggesting Google uncertainty, or inconsistent CTR across its query portfolio.

SEO practitioners need a page-level health view that synthesizes click behavior across all queries, tracks momentum over time, and interprets findings through the NavBoost framework to prioritize which pages need attention and why.

## Proposed Solution

### Phase 1: GSC-Only Click Quality Indicators (v1)

Add **Pass 5: Click Quality Signals** to SKILL.md with six analysis dimensions that are genuinely distinct from Pass 2:

| Dimension | What It Measures | GSC Fields Used | NavBoost Alignment |
|-----------|-----------------|-----------------|-------------------|
| **Click Health Score** | Composite page-level CTR performance across all queries | clicks, impressions, position (aggregated per page) | Clicks signal — relevance |
| **Click Trend Velocity** | Week-over-week click momentum (accelerating, stable, decaying) | clicks by date dimension, grouped by week | NavBoost's 13-month memory — sustained signals matter |
| **Query Concentration Risk** | Does the page depend on 1-2 queries or have diversified click sources? | clicks per query per page | Fragility indicator — concentrated = one algorithm change away from collapse |
| **Position Stability** | Is the page's average position volatile or stable across weeks? | position by date dimension | Google testing/uncertainty signal — volatile = NavBoost hasn't decided |
| **CTR Consistency** | Does CTR hold across the page's full query portfolio, or do some queries severely underperform? | CTR per query per page vs position-expected CTR | Bad clicks signal — queries with far-below-expected CTR suggest dissatisfaction |
| **Impression-to-Click Efficiency** | How efficiently does the page convert visibility into clicks across all queries? | impressions, clicks aggregated per page | Overall relevance signal |

**Key differentiation from Pass 2:**
- Pass 2 says: *"This keyword has a CTR gap — rewrite the title tag"*
- Pass 5 says: *"This page has an overall click quality problem — here's the diagnosis across all its queries, the trend direction, and the NavBoost-informed interpretation"*

### Phase 2: GA4 Integration (Future Enhancement)

Add optional GA4 engagement data to unlock true post-click measurement:
- Average engagement time per page (proxy for good clicks vs bad clicks)
- Bounce rate (proxy for bad clicks)
- Pages per session from organic entry (proxy for last longest clicks)
- Returning visitor rate by landing page

This phase requires new credentials, setup guide updates, and URL-matching logic between GA4 paths and GSC URLs. Spec separately.

## Technical Approach

### Files to Modify

1. **`plugins/gsc-seo-audit/skills/audit/SKILL.md`** — Add Pass 5 section (~80-100 lines)
2. **`README.md`** — Add Pass 5 to feature list and usage examples

### Pass 5 Specification (to add to SKILL.md)

#### Data Requirements

Pass 5 generates its own GSC queries (self-contained, does not depend on Pass 1/2 context):

**Query 1 — Page-level aggregates (current 90 days):**
- Dimensions: `page`
- Metrics: clicks, impressions, ctr, position
- Filter: pages with 200+ impressions AND 10+ clicks

**Query 2 — Page-level weekly trends (current 90 days):**
- Dimensions: `page`, `date`
- Group by: page, ISO week
- Used for: click velocity and position stability calculations

**Query 3 — Page × Query breakdown (current 90 days):**
- Dimensions: `page`, `query`
- Filter: pages from Query 1 results only
- Used for: concentration risk and CTR consistency

**Data thresholds** (matching Pass 2's pattern of explicit minimums):
- Minimum 200 impressions per page to receive a score
- Minimum 10 clicks per page to receive a score
- Pages below threshold get "Insufficient data" label
- Query-level breakdowns require 50+ impressions per query-page pair

#### Click Health Score Formula

A composite score (0-100) for each qualifying page:

```
Click Health Score = weighted average of:
  - CTR Performance (30%): actual aggregate CTR vs expected CTR at avg position
  - Click Velocity (25%): week-over-week click trend (positive = good)
  - Query Diversity (20%): 1 - HHI of click distribution across queries
  - Position Stability (15%): inverse of position standard deviation across weeks
  - CTR Consistency (10%): % of queries with CTR within 50% of position-expected CTR
```

**Important framing:** The score header should read:
> **Click Quality Indicators** (informed by NavBoost research)

And include a methodology note:
> These indicators are derived from GSC data and align with the dimensions Google's NavBoost system is known to evaluate, based on antitrust trial testimony and API documentation. They do not directly measure NavBoost signals. Engagement metrics from GA4 would improve confidence in post-click quality assessment.

#### Output Sections

**1. Click Quality Dashboard**
Table of all qualifying pages with:
- Page URL
- Click Health Score (0-100)
- Score tier: Strong (75+), Moderate (50-74), Weak (25-49), Critical (<25)
- Primary issue flag (e.g., "Declining velocity", "Single-query dependent", "Volatile position")
- Trend arrow (↑ improving, → stable, ↓ declining)

**2. Click Velocity Analysis**
- Pages with strongest positive momentum (accelerating clicks)
- Pages with strongest negative momentum (decaying clicks)
- Week-over-week chart for top 10 pages in each direction
- NavBoost interpretation: "Pages with sustained declining velocity may be losing NavBoost credit as Google's 13-month memory window updates"

**3. Query Concentration Risk Report**
- Pages dependent on a single query for 60%+ of clicks — "fragile"
- Pages with diversified query portfolios — "resilient"
- Recommendation: diversify content to capture related queries for fragile pages

**4. Position Stability Assessment**
- Pages with high position variance (Google is still testing)
- Pages with locked-in positions (NavBoost has likely settled)
- Interpretation: volatile positions = opportunity to influence NavBoost's decision through click quality improvements

**5. CTR Consistency Audit**
- Per page: which queries are performing well vs poorly relative to position
- This is distinct from Pass 2's approach — here we show the spread per page, not individual keyword gaps
- Pages where most queries underperform = systemic page-level issue (not a title tag problem)
- Pages where only 1-2 queries underperform = targeted fix needed

**6. Prioritized Action List**
Ranked by Click Health Score (worst first), with specific recommendations:
- Critical pages: diagnose why clicks are declining or concentrated
- Weak pages: specific issues identified (volatility, consistency, concentration)
- Moderate pages: targeted improvements to reach Strong tier
- Format matches existing plugin convention: ranked by estimated impact, ease, speed

#### Cross-Reference with Earlier Passes (When Available)

If Pass 1/2 data is in the conversation context, enhance the analysis:
- Pages that are "Biggest Losers" (Pass 1) AND have declining click velocity (Pass 5) = highest priority
- Striking distance keywords (Pass 1) on pages with Strong click health = best promotion candidates
- Pages with CTR gaps (Pass 2) AND low CTR consistency (Pass 5) = systemic page issue, not just a title problem
- Orphan pages (Pass 4) with Weak click health = internal linking could boost NavBoost signals

### NavBoost Framing Guidelines

Include in SKILL.md as instructions for how Claude should frame the analysis:

1. **Never claim to measure NavBoost directly.** Use "indicators that align with" or "informed by NavBoost research."
2. **Cite sources when interpreting.** Reference the antitrust trial, API leak, or patent when making NavBoost-related claims.
3. **Distinguish confidence levels.** CTR and click volume = high confidence (direct GSC data). Click quality inference = moderate confidence (proxy-based). Satisfaction signals = low confidence without GA4.
4. **Don't oversell the score.** The Click Health Score is a diagnostic tool, not a prediction of Google rankings. Frame it as: "Pages with low scores likely have characteristics that would generate weak click signals in NavBoost's framework."

## Acceptance Criteria

- [ ] Pass 5 added to SKILL.md following the existing pass pattern (section header, methodology, data queries, output format, analysis notes)
- [ ] Click Health Score formula implemented with all 5 components and documented weights
- [ ] Data thresholds defined (200+ impressions, 10+ clicks minimum)
- [ ] All 6 output sections specified with table formats matching plugin conventions
- [ ] NavBoost framing guidelines included — no overclaiming
- [ ] Cross-reference section covers Pass 1, 2, and 4 integration points
- [ ] README.md updated with Pass 5 in feature list and usage examples
- [ ] Pass 5 is self-contained (re-queries GSC, does not require prior passes)
- [ ] Methodology note included explaining proxy-based approach and GA4 caveat
- [ ] "Insufficient data" handling for pages below thresholds

## What This Does NOT Include (Phase 2 / Future)

- GA4 integration (requires separate credential flow, URL matching, setup guide)
- Direct bounce rate or engagement time measurement
- "Last longest click" scoring (impossible without post-click behavioral data)
- Changes to Passes 1-4
- Any new npm dependencies or authentication patterns

## Success Metrics

- Pass 5 produces actionable insights on a real site with GSC data
- Output clearly differentiates from Pass 2 (page-level vs query-level)
- No user confusion about what is being measured vs inferred
- NavBoost framing adds strategic context without overclaiming

## Dependencies & Risks

| Risk | Likelihood | Mitigation |
|------|-----------|------------|
| Pass 5 output overlaps with Pass 2 | Medium | Strict page-level aggregation; Pass 2 stays query-level. Cross-reference section explicitly shows how they complement each other |
| Users interpret Click Health Score as Google's actual score | Medium | Methodology note, confidence labels, "informed by" language |
| GSC 5,000-row limit truncates Query 3 data for large sites | Medium | Filter Query 3 to only pages from Query 1 results; document the sampling caveat |
| Click velocity misleading for seasonal sites | Low | Note seasonal patterns in the analysis; compare against site-wide trend, not absolute direction |

## Implementation Estimate

This is a prompt-only change — modifying SKILL.md and README.md. No code, no new dependencies, no new auth.

### MVP

#### SKILL.md — Pass 5 Addition

The new section follows the exact pattern of Passes 1-4: methodology description, data queries (as Node.js script patterns), output format specification, analysis guidelines, and cross-reference notes.

#### README.md — Updates

- Add "Click Quality Signals" to the feature bullet list
- Add Pass 5 usage examples to the "What You Can Ask" section
- Add brief explanation of Click Health Score to the "How CTR Gap Analysis Works" section (or a new parallel section)

## Sources & References

- **Zyppy article:** [Google Click Signals](https://signal.zyppy.com/p/google-click-signals) — primary source for NavBoost framework
- **Google Patent:** US8661029B1 — click-fraction re-ranking methodology
- **Google antitrust trial:** U.S. v. Google — NavBoost testimony and documents
- **Google API leak:** Internal attribute names (goodClicks, lastLongestClicks, badClicks)
- **Existing plugin:** `plugins/gsc-seo-audit/skills/audit/SKILL.md` — Pass 1-4 patterns to follow
- **CTR benchmarks:** Position-based expected CTR table already in SKILL.md (lines 85-91)
