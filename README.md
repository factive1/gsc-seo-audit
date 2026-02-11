# GSC SEO Audit — Claude Code Plugin

Run comprehensive SEO audits using the Google Search Console API directly from Claude Code. No paid SEO tools needed.

## What It Does

This plugin turns Claude Code into a senior SEO analyst. Point it at your Search Console property and it will:

- **Traffic Analysis** — Site-wide metrics with period-over-period comparison, biggest winners/losers, weekly trends
- **CTR Gap Analysis** — Find keywords where you're ranking but losing clicks, with exact missed click counts
- **Keyword Cannibalization** — Detect pages competing against each other for the same queries
- **Striking Distance Keywords** — Surface keywords at positions 11-20 that are close to page 1
- **Title Tag Recommendations** — AI-generated title tags optimized for CTR using proven click triggers
- **Technical SEO** — Lighthouse audits, schema markup review, Core Web Vitals assessment
- **Disappeared Pages** — Find pages that dropped out of search results entirely

## Install

```bash
claude /plugin install --dir /path/to/gsc-seo-audit
```

Or if published to a GitHub repo:

```bash
claude /plugin install github:your-username/gsc-seo-audit
```

## Setup (One Time)

You need a Google Cloud service account connected to your Search Console property. Takes ~10 minutes.

### Quick Version

1. Create a [Google Cloud project](https://console.cloud.google.com/) and enable the **Search Console API**
2. Create a **Service Account** under APIs & Services > Credentials
3. Download the JSON key file
4. Add the service account email as a user in [Search Console Settings](https://search.google.com/search-console) > Users and permissions
5. Add credentials to `.env.local` in your project:

```bash
GSC_SERVICE_ACCOUNT_EMAIL=seo-reader@your-project.iam.gserviceaccount.com
GSC_PRIVATE_KEY="-----BEGIN PRIVATE KEY-----\nYOUR_KEY_HERE\n-----END PRIVATE KEY-----\n"
GSC_SITE_URL=sc-domain:example.com
GSC_BLOG_PATH=/blog/
```

For the detailed walkthrough with screenshots and troubleshooting, see [setup-guide.md](skills/audit/setup-guide.md).

## Usage

```
/gsc-seo-audit:audit example.com
```

Or just run `/gsc-seo-audit:audit` and Claude will find your credentials automatically.

### What You Can Ask

**Pass 1 — Broad audit:**
```
/gsc-seo-audit:audit
```
Returns: traffic overview, top pages, biggest losers/winners, top keywords, striking distance, weekly trends, device/country breakdown, disappeared pages.

**Pass 2 — Go deeper:**
After the initial audit, just ask Claude to go deeper on any area:
- "Go deeper on CTR gaps"
- "Show me keyword cannibalization"
- "Run a keyword-level CTR analysis with title tag recommendations"
- "Find pages losing the most traffic"

**Pass 3 — Technical SEO:**
- "Run Lighthouse audits on the top pages"
- "Check schema markup"
- "What are the page speed issues?"

## How CTR Gap Analysis Works

The plugin compares your actual click-through rate against industry benchmarks for each SERP position:

| Position | Expected CTR |
|----------|-------------|
| 1 | 13.3% |
| 2 | 11.9% |
| 3 | 10.0% |
| 5 | 6.4% |
| 10 | 1.9% |

**Missed Clicks** = Impressions × (Expected CTR - Actual CTR)

This tells you exactly how many clicks you're leaving on the table for each keyword, and which title tags need rewriting.

## Requirements

- Claude Code 1.0.33+
- Node.js 18+
- `googleapis` npm package (the plugin will install it automatically with `--no-save` if missing)
- A Google Cloud service account with Search Console API access

## No Paid Tools Required

This plugin queries the Google Search Console API directly. No Ahrefs, Semrush, Moz, or any other paid SEO tool needed. The data comes straight from Google.

## License

MIT
