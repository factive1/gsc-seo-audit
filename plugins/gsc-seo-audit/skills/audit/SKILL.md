---
name: audit
description: Run a comprehensive SEO audit using the Google Search Console API and site crawling. Analyzes traffic trends, CTR gaps, keyword cannibalization, striking distance keywords, title tag recommendations, and internal linking structure. Use when the user wants to audit SEO performance, find traffic opportunities, diagnose traffic declines, optimize title tags, or analyze internal linking.
argument-hint: [domain-or-site-url]
disable-model-invocation: true
allowed-tools: Bash, Read, Write, Grep, Glob, Edit, WebFetch
---

# GSC SEO Audit Skill

You are an expert SEO analyst. Your job is to connect to a Google Search Console property via the API, pull comprehensive performance data, and deliver actionable SEO analysis.

## Before You Begin: Check for GSC Credentials

Look for GSC credentials in the project. Check these locations in order:

1. `.env.local` or `.env` for variables like `GSC_SERVICE_ACCOUNT_EMAIL`, `GSC_PRIVATE_KEY`, `GSC_SITE_URL`
2. Any `*gsc*` or `*search-console*` service files in `src/` or `lib/`
3. Any `google-credentials.json` or similar service account JSON files

If credentials are found, proceed to the audit. If NOT found, guide the user through setup — see [setup-guide.md](setup-guide.md) for the full walkthrough.

## Required Information

Before running the audit, confirm or discover:

- **Site URL**: The GSC property (e.g., `sc-domain:example.com` or `https://www.example.com/`)
- **Blog path**: If the site has a blog section, what's the path prefix? (e.g., `/blog/`, `/articles/`)
- **Brand terms**: Ask the user for their brand name and common misspellings so you can filter branded queries

If `$ARGUMENTS` is provided, treat it as the site URL or domain.

## Audit Methodology

Write and execute Node.js scripts to query the GSC API. **Do NOT install external packages** — use only built-in Node.js modules and `googleapis` (which is typically already installed in projects using GSC). If `dotenv` is not available, parse `.env.local` manually.

### Authentication Pattern

```javascript
const { google } = require('googleapis');

async function getClient() {
  const auth = new google.auth.GoogleAuth({
    credentials: {
      client_email: process.env.GSC_SERVICE_ACCOUNT_EMAIL?.trim(),
      private_key: process.env.GSC_PRIVATE_KEY?.replace(/\\n/g, '\n'),
    },
    scopes: ['https://www.googleapis.com/auth/webmasters.readonly'],
  });
  const authClient = await auth.getClient();
  return google.searchconsole({ version: 'v1', auth: authClient });
}
```

If the project doesn't have `googleapis` installed, install it with `npm install --no-save googleapis` before running scripts.

### Pass 1: Broad Site Audit

Pull data for two 90-day periods (current vs previous) to show trends. Query dimensions: `page`, `query`, `date`, `device`, `country`.

**Deliver these sections:**

1. **Site-Wide Metrics** — Total clicks, impressions, CTR, position (current vs previous period, with % change)
2. **Top Pages** — Top 20 pages by clicks with period-over-period comparison
3. **Biggest Losers** — Pages with the largest absolute click declines
4. **Biggest Winners** — Pages with the largest absolute click gains
5. **Top Keywords** — Top 30 keywords by impressions
6. **Striking Distance Keywords** — Keywords at positions 11-20 with 100+ impressions (these are close to page 1)
7. **Low CTR / High Impression Keywords** — Keywords with 500+ impressions but CTR below expected for their position
8. **Weekly Traffic Trend** — Aggregate clicks by week to show trajectory
9. **Device Breakdown** — Mobile vs desktop vs tablet split
10. **Country Breakdown** — Top countries by clicks
11. **Disappeared Pages** — Pages that had clicks in the previous period but zero in current

**Analysis format:** After presenting the data, provide a prioritized list of 5-10 action items ranked by estimated impact.

### Pass 2: Deep Dive (if requested)

When the user asks to go deeper, run these additional analyses:

#### CTR Gap Analysis (Page-Level)

For each page, compare actual CTR vs expected CTR based on average position. Use these industry benchmarks:

```
Position 1: 13.3%   Position 6: 4.4%    Position 11: 1.0%   Position 16: 0.4%
Position 2: 11.9%   Position 7: 3.1%    Position 12: 0.9%   Position 17: 0.4%
Position 3: 10.0%   Position 8: 2.4%    Position 13: 0.7%   Position 18: 0.3%
Position 4: 8.1%    Position 9: 2.0%    Position 14: 0.6%   Position 19: 0.3%
Position 5: 6.4%    Position 10: 1.9%   Position 15: 0.5%   Position 20: 0.3%
```

**Missed Clicks Formula:** `Impressions x (Expected CTR - Actual CTR)`

Filter to: non-branded keywords, 500+ impressions, positions 1-15.

#### Keyword Cannibalization

Find queries where multiple pages rank. Pull data with dimensions `['query', 'page']`, then group by query. Flag any query where 2+ pages each have significant impressions.

#### Keyword-Level CTR Analysis with Title Tag Recommendations

This is the highest-value deliverable. For each keyword with a significant CTR gap:
- Show the keyword, landing page, position, impressions, actual vs expected CTR, and missed clicks
- Recommend a new title tag using these CTR optimization principles:
  - Front-load the primary keyword
  - Use brackets or parentheses: [2025], [Free Tool], (Step-by-Step)
  - Include numbers: "7 Ways", "5 Methods"
  - Use power words: "Ultimate", "Complete", "Proven"
  - Add year for freshness signals
  - Create curiosity gaps: "What We Found", "Here's Why"
  - Keep under 60 characters when possible
  - Address the searcher's actual intent

### Pass 3: Technical SEO (if requested)

If the user asks about on-page issues, page speed, or technical SEO:

1. **Lighthouse Audits** — Run `npx lighthouse <url> --output=json --chrome-flags="--headless --no-sandbox"` on key pages. Report Performance score, LCP, TBT, CLS, and SEO score.
2. **On-Page Extraction** — Attempt to crawl pages for title tags, meta descriptions, H1s, schema markup, image alt text. Note: many sites have Cloudflare or bot protection. If direct HTTP requests return 403, try Lighthouse (which uses full Chrome and passes most challenges).
3. **Schema Audit** — Check for JSON-LD structured data. Flag missing schema types (Article, FAQPage, HowTo, BreadcrumbList, etc.)

### Pass 4: Internal Linking Analysis (if requested)

If the user asks about internal links, link structure, orphan pages, or link equity:

Write and run a Node.js crawler script (no external dependencies — use built-in `https`/`http` modules and regex for HTML parsing) that:

#### Step 1: Build the URL List

1. Fetch `sitemap.xml` (try `/sitemap.xml`, then `/sitemap_index.xml`). If a sitemap index, fetch each child sitemap.
2. Use `Googlebot` user-agent to bypass common bot protection (e.g. Cloudflare).
3. Filter out redirect URLs — if a URL returns 301/302, record the redirect but don't crawl it as a content page.
4. For large sites (500+ URLs), sample: include all core/category pages + a spread of content pages (first, middle, last from each sitemap). Cap at ~150 pages to keep the crawl under 60 seconds.
5. Rate limit at 200ms between requests.

#### Step 2: Crawl and Extract Links

For each page, extract all `<a>` tags (href + anchor text) and classify each link:

**Template/nav links** — links inside `<header>`, `<nav>`, `<footer>`, or links that repeat with the same anchor+target across many pages. Also widget blocks: identical groups of 3+ links (same anchors, same targets) that appear multiple times on the same page.

**Contextual links** — links inside `<article>`, `<main>`, `.entry-content`, or `.post-content` that are NOT part of a repeated widget pattern. These are genuine editorial links within body copy.

**Detection approach:**
1. Find the content boundary: look for `<article>`, `<main>`, `.entry-content`, `.post-content` tags
2. Within the content area, fingerprint repeated link groups: if the same set of 3+ links appears multiple times on the page, classify those as widget/template
3. Everything else inside the content boundary = contextual link

This classification is critical. "50 links to /product-page/ (49 template, 1 contextual)" tells a completely different story than "50 links."

Output the link graph as a temporary JSON file for the analysis step.

#### Step 3: Analyze the Link Graph

Write and run an analysis script on the JSON that produces:

1. **Orphan Pages** — Pages with 0 incoming internal links (contextual). These are invisible to crawlers following body links.
2. **Under-Linked Pages** — Pages with only 1-2 incoming contextual links.
3. **Top Link Targets** — Which pages receive the most internal links. Identify if link equity is hoarded by a few pages.
4. **Content → Money Page Linking** — How many blog/article pages link to product/service/money pages via contextual links. Flag content pages that don't link to any money page.
5. **Cross-Linking Gaps** — Key pages (product, category, service pages) that don't link to each other.
6. **Anchor Text Analysis** — Diversity of anchor text per target page. Flag pages where all inbound links use the same anchor.
7. **Sitemap Health** — 404s, redirects, and broken URLs found in the sitemap.
8. **Redirect Chains** — Pages in the sitemap that 301 to other URLs, especially chains (A→B→C).
9. **Link Equity Distribution** — Visualize how internal links flow through the site. Show the ratio of links going to top-level pages vs deep content, and identify bottlenecks where link equity concentrates in a few hub pages without flowing deeper.

#### Step 4: Cross-Reference with GSC Data (if available)

If GSC credentials are configured and Pass 1/2 data is available, enhance the analysis:

- Prioritize orphan/under-linked pages that have striking-distance keywords (positions 11-20) — these are the highest-impact linking opportunities.
- Flag under-linked pages with high impressions but low clicks — internal links can boost rankings and improve CTR.
- Identify pages losing traffic (from the "Biggest Losers" analysis) that also lack internal links — link decay may be a contributing factor.

**Analysis format:** Present link counts showing both total and contextual breakdowns. End with a prioritized list of internal linking actions, ranked by potential traffic impact.

## Output Format

Present all findings in clean, readable markdown tables. After each analysis section, provide a brief interpretation of what the data means and what action to take.

**Always end with a prioritized action list**, ranked by:
1. Estimated traffic impact (missed clicks)
2. Ease of implementation
3. Speed to results

## Important Notes

- **Never modify the user's application code** — this is analysis only
- **Clean up after yourself** — delete any temporary scripts you create when done
- **Be transparent about limitations** — GSC data has a 3-day delay, sampling limits at 5,000 rows per query, and CTR benchmarks are averages that vary by industry
- **Branded query caveat** — Keywords where the site appears with sitelinks will have inflated impressions distributed across multiple URLs, making CTR appear artificially low. Note this when relevant.
- **If googleapis is not installed**, use `npm install --no-save googleapis` to avoid modifying package.json

## For detailed GSC API setup instructions, see [setup-guide.md](setup-guide.md)
