# Google Search Console API Setup Guide

This guide walks you through connecting your Google Search Console account so Claude Code can pull data directly via the API.

## Overview

You need three things:
1. A Google Cloud project with the Search Console API enabled
2. A service account with credentials
3. That service account added as a user in your GSC property

Total time: ~10 minutes. No credit card required.

---

## Step 1: Create a Google Cloud Project

1. Go to [Google Cloud Console](https://console.cloud.google.com/)
2. Click the project dropdown at the top of the page
3. Click **New Project**
4. Name it something like `seo-automation` and click **Create**
5. Make sure the new project is selected in the dropdown

If you already have a GCP project, you can skip this step and use the existing one.

---

## Step 2: Enable the Search Console API

1. In your GCP project, go to **APIs & Services > Library** (or search "API Library" in the top search bar)
2. Search for **Google Search Console API**
3. Click on it and click **Enable**

This is the API that lets you programmatically query your search performance data — the same data you see in the Search Console web interface.

---

## Step 3: Create a Service Account

1. Go to **APIs & Services > Credentials**
2. Click **+ Create Credentials** at the top
3. Select **Service account**
4. Give it a name like `seo-reader` and click **Create and Continue**
5. For the role, you can skip this (click **Continue**) — the service account doesn't need GCP project-level roles
6. Click **Done**

Now create a key for the service account:

1. Click on the service account you just created
2. Go to the **Keys** tab
3. Click **Add Key > Create New Key**
4. Select **JSON** and click **Create**
5. A JSON file will download — this contains your credentials

The JSON file will look something like this:

```json
{
  "type": "service_account",
  "project_id": "your-project-id",
  "private_key_id": "abc123...",
  "private_key": "-----BEGIN PRIVATE KEY-----\nMIIEv...\n-----END PRIVATE KEY-----\n",
  "client_email": "seo-reader@your-project-id.iam.gserviceaccount.com",
  "client_id": "123456789",
  ...
}
```

You need two values from this file:
- `client_email` — the service account email
- `private_key` — the private key (including the BEGIN/END markers)

---

## Step 4: Add the Service Account to Google Search Console

1. Go to [Google Search Console](https://search.google.com/search-console)
2. Select your property
3. Click **Settings** (gear icon) in the left sidebar
4. Click **Users and permissions**
5. Click **Add User**
6. Enter the service account email (the `client_email` from the JSON file, e.g., `seo-reader@your-project-id.iam.gserviceaccount.com`)
7. Set permission to **Full** (read access is sufficient, but "Full" ensures no permission issues)
8. Click **Add**

**Important:** The service account needs to be added to each property you want to analyze. If you have both `sc-domain:example.com` and `https://www.example.com/` properties, add it to the one you want to query.

---

## Step 5: Add Credentials to Your Project

Create or update `.env.local` (or `.env`) in your project root with these variables:

```bash
# Google Search Console Configuration
GSC_SERVICE_ACCOUNT_EMAIL=seo-reader@your-project-id.iam.gserviceaccount.com
GSC_PRIVATE_KEY="-----BEGIN PRIVATE KEY-----\nMIIEv...your-full-private-key...\n-----END PRIVATE KEY-----\n"
GSC_SITE_URL=sc-domain:example.com
GSC_BLOG_PATH=/blog/
```

### Variable Reference

| Variable | Required | Description | Example |
|----------|----------|-------------|---------|
| `GSC_SERVICE_ACCOUNT_EMAIL` | Yes | The service account email from Step 3 | `seo-reader@my-project.iam.gserviceaccount.com` |
| `GSC_PRIVATE_KEY` | Yes | The full private key string, wrapped in quotes | `"-----BEGIN PRIVATE KEY-----\n..."` |
| `GSC_SITE_URL` | Yes | Your GSC property URL | `sc-domain:example.com` or `https://www.example.com/` |
| `GSC_BLOG_PATH` | No | Path prefix for blog content (for blog-specific analysis) | `/blog/`, `/articles/`, `/posts/` |

### Site URL Format

GSC has two types of properties:

- **Domain property**: `sc-domain:example.com` — covers all subdomains and protocols. This is the most common and recommended type.
- **URL-prefix property**: `https://www.example.com/` — covers only that exact prefix. Include the trailing slash.

Check your Search Console to see which type your property is. Use the exact format shown in the SC interface.

### Private Key Formatting

The private key is a multi-line string. In `.env.local`, you have two options:

**Option A — Escaped newlines (recommended):**
```bash
GSC_PRIVATE_KEY="-----BEGIN PRIVATE KEY-----\nMIIEvQIBADANBg...\n-----END PRIVATE KEY-----\n"
```
Replace actual line breaks with `\n`. This is one long line.

**Option B — Paste the JSON file path instead:**
If you prefer, save the entire JSON key file and reference it:
```bash
GOOGLE_APPLICATION_CREDENTIALS=./google-credentials.json
GSC_SITE_URL=sc-domain:example.com
```
Note: This approach uses Application Default Credentials and requires a slightly different auth pattern in the code.

---

## Step 6: Install googleapis (if needed)

The audit scripts use the `googleapis` npm package. If your project doesn't already have it:

```bash
npm install --no-save googleapis
```

The `--no-save` flag installs it without modifying your `package.json`.

---

## Step 7: Verify the Connection

You can verify the setup works by asking Claude Code to run a quick test:

```
/gsc-seo-audit example.com
```

Or simply ask: "Connect to my Search Console and show me the top 10 pages by clicks for the last 30 days."

---

## Troubleshooting

### "The caller does not have permission"
- The service account email hasn't been added to the GSC property (Step 4)
- Double-check the email address matches exactly

### "Invalid grant" or authentication errors
- The private key may be malformed — make sure all `\n` sequences are preserved
- The service account may have been deleted or disabled in GCP

### "Not Found" errors
- The `GSC_SITE_URL` format is wrong
- For domain properties, use `sc-domain:example.com` (no `https://`)
- For URL-prefix properties, include the full URL with trailing slash

### "API has not been enabled"
- Go back to Step 2 and enable the Search Console API in your GCP project

### Rate Limits
- The GSC API has generous rate limits (1,200 queries per minute)
- If you hit limits, the scripts will show a rate limit error — just wait a moment and retry

---

## Security Notes

- **Never commit `.env.local` to version control** — add it to `.gitignore`
- **Never commit the JSON key file** — add `*.json` key files to `.gitignore`
- The service account has read-only access to your Search Console data
- You can revoke access at any time by removing the service account from GSC settings or deleting the service account in GCP
