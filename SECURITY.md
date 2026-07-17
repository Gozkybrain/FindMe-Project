# FindMe - Developer Guide

## Overview

FindMe is a LinkedIn profile scraper wrapped in a Next.js UI. It launches a real Chromium browser (via Playwright), injects the user's own LinkedIn session cookies, and runs people searches on LinkedIn directly. No API keys, no third-party data brokers.

### Flow
1. User exports LinkedIn cookies (li_at + JSESSIONID) using the Cookie-Editor browser extension
2. Pastes them into the sidebar accordion in the UI
3. Fills in search criteria (location, industry, job title, etc.)
4. Clicks Start Scan
5. Next.js API route spawns `node index.js` as a child process with the cookies
6. The scraper runs multiple LinkedIn people-search queries, deduplicates, writes results to `output/profiles.json`
7. The API polls the child process, reads `output/profiles.json` when done, returns profiles to the UI
8. Results displayed in a table; JSON tab shows raw data; Import tab merges previous exports

---

## Directory Structure

```
FindMe/
├── api/
│   ├── enrich/          (empty - deleted, unused)
│   └── scan/route.js    Backend API - accepts POST to start scan, GET to poll status
├── sources/
│   └── linkedin.js      Playwright-based LinkedIn scraper (the core engine)
├── src/
│   ├── app/
│   │   ├── globals.css   All styles (green/gold theme, dark/light, responsive, no border-radius)
│   │   ├── layout.jsx    Root layout (imports globals.css, sets <html>/<body>)
│   │   └── page.jsx      Main UI - sidebar accordion, 5 tabs, scan flow, import/merge
│   ├── components/
│   │   ├── CookieAccordion.jsx   Sidebar: paste cookies / show saved status with countdown
│   │   ├── CookieUpload.jsx      (unused - kept for reference, delete if clean)
│   │   ├── ImportJson.jsx        Import tab: paste JSON array of profiles, deduplicates on import
│   │   ├── ResultsTable.jsx      Profile results table (name, location, headline, LinkedIn link)
│   │   ├── TargetForm.jsx        Search form (firstName, lastName, location, industry, jobType, count)
│   │   └── TerminalLog.jsx       Live log output during scan (fullscreen during scan, embedded in Logs tab)
│   └── lib/
│       └── api.js        Fetch wrappers: startScan(POST), getScanStatus(GET), clearCookies(DELETE)
├── index.js              CLI entry - loads config, runs LinkedIn scraper, writes output/profiles.json
├── config.js             Default config (all fields empty, count 20, source: linkedin only)
├── merger.js             Profile merge/dedup helper - merges by email or name similarity (≥60%)
├── package.json          Dependencies: Next.js, React, Playwright, playwright-extra, stealth plugin
├── SECURITY.md           This file
├── cookies-template.json Template for cookie format reference
├── next.config.js        Next.js configuration
├── jsconfig.json         JS config with path aliases (@/ → src/)
└── output/               (gitignored) Written by index.js - profiles.json with last scan results
```

---

## Key Files Explained

### `api/scan/route.js` - Scan API

The Next.js API route that orchestrates a scan.

**POST /api/scan**
- Accepts `{ targets, cookies }` in the body
- Writes cookies to a temp file (`cookies.tmp.*.json`)
- Writes a config file (`config.tmp.*.js`) with the search targets
- Spawns `node index.js` as a child process with `COOKIE_FILE` and `NODE_CONFIG` env vars
- Returns `{ ok: true }` immediately - scan runs async

**GET /api/scan**
- Returns the current scan state: `{ running, log, profiles, error }`
- UI polls this every 400ms until `running: false`

**Cleanup**
- Temp files (cookie + config) deleted on process exit
- Cookies never stored permanently - temp files are removed

### `sources/linkedin.js` - LinkedIn Scraper

Node.js script using Playwright with stealth plugin to avoid detection.

**How it works:**
1. Reads LinkedIn cookies from the temp cookie file
2. Launches a headless Chromium browser (or visible if `HEADLESS=false`)
3. Creates a new browser context with the real LinkedIn cookies injected
4. Builds search queries from the user's criteria - tries variations (senior, retired, former director) to maximize coverage
5. For each query: navigates to `linkedin.com/search/results/people/?keywords=...`
6. Scrapes the DOM for profile cards: extracts name, headline, location, and LinkedIn URL
7. Deduplicates by LinkedIn URL or full name
8. Returns up to `targets.count` profiles

**Query strategy** (`buildSearchQueries`):
- If industry is provided: `"Location Industry"`, `"Location retired Industry"`, `"Location former director Industry"`, `"Location senior Industry"`
- If only location: same pattern with generic titles
- Job title/keywords appended as an additional query
- All queries are deduplicated with a Set

**Scraping logic** (inside `page.evaluate`):
- Finds all `<a href*="/in/">` links
- Walks up the DOM tree to find the profile card container (looks for text containing "Connect", "Follow", or "Message")
- Parses the first non-empty line as name, next relevant lines as headline and location
- Filters out duplicate `/in/` links

**Headless mode**: Controlled by `HEADLESS` env var. Defaults to `true`. Set to `false` for debugging.

### `index.js` - CLI Entry

Compatible with both API-triggered and manual CLI usage.

**CLI usage**: `npm run scan` or `HEADLESS=false node index.js`

**What it does:**
1. Loads config (from `NODE_CONFIG` env var or default `config.js`)
2. Calls the LinkedIn scraper
3. Merges profiles (via `merger.js`)
4. Writes formatted output to `output/profiles.json`

**Output format** (`output/profiles.json`):
```json
{
  "generatedAt": "2026-01-01T...",
  "targetDescription": "Texas, USA",
  "totalProfiles": 20,
  "profiles": [
    {
      "fullName": "John Doe",
      "firstName": "John",
      "lastName": "Doe",
      "location": "Austin, Texas",
      "headline": "Software Engineer at ...",
      "linkedin": "https://www.linkedin.com/in/...",
      "snippet": "..."
    }
  ]
}
```

### `config.js` - Default Configuration

```js
module.exports = {
  targets: {
    location: "",          // e.g. "California, USA"
    firstName: "",         // optional filter
    lastName: "",          // optional filter
    industry: "",          // e.g. "Aerospace"
    jobType: "",           // e.g. "Hardware Engineer"
    count: 20,             // max profiles to return (capped at 50 in API)
  },
  sources: {
    linkedin: { enabled: true },
  },
  output: {
    file: "output/profiles.json",
  },
};
```

When triggered via the API, a temp config is generated that overrides `config.js` with the user's form inputs.

### `merger.js` - Profile Merging Helper

Originally designed for multi-source merging. Currently only LinkedIn is active.

**Dedup logic:**
1. First tries to match by exact email
2. Falls back to name similarity (≥60% shared tokens)
3. Unmatched profiles are appended as new entries

**Merge rules** (when a new profile matches an existing one):
- Preserves existing values for email, location, names (fills in blanks)
- Adds the new source to the `sources` array
- Appends search date to `matchedOn`
- Never overwrites existing data

### `src/app/page.jsx` - Main UI

React client component managing the entire UI.

**States:**
- **Default** (no scan yet): 2 tabs - "New Scan" (form + existing results table) | "Import"
- **During scan**: Full-height terminal showing live logs
- **After scan**: 5 tabs - Results, JSON, Logs, New Scan, Import

**Key state variables:**
- `profiles[]` - current profile data (from scan or import)
- `userCookies` - LinkedIn cookies from localStorage (12h TTL)
- `loading`/`scanDone` - scan phase tracking
- `lastTargets` - search criteria shown in sidebar (cleared on import)
- `mainTab` - active tab
- `mergeInfo` - merge notification shown above results

**Cookie persistence:**
- Cookies stored in localStorage with 12-hour expiry
- `useEffect` checks expiry every second and auto-clears
- No cookies sent to any server - only used to pass to the API route (which writes them to a temp file)

**Theme:**
- Dark/light toggle, persisted to localStorage
- Default respects `prefers-color-scheme`

### `src/components/` - UI Components

**CookieAccordion.jsx**: Collapsible sidebar section. When empty shows a textarea to paste cookies + export instructions. When saved shows session info (created time, time remaining) with a clear button.

**TargetForm.jsx**: Search form with fields for first name, last name, location, industry, job title/keywords, and max profiles. Start Scan button disabled until cookies are pasted and at least one of location/industry/job title is filled.

**ResultsTable.jsx**: Displays profiles in a table with Name, Location, Headline, and LinkedIn link columns. Empty state shows a placeholder message.

**TerminalLog.jsx**: Renders an array of log lines in a dark terminal-like `<pre>` block. Used fullscreen during scan and embedded in the Logs tab after scan.

**ImportJson.jsx**: Textarea to paste a JSON array of profiles. Validates JSON, calls `handleImport` which deduplicates and merges with any existing results.

---

## Dependencies

| Package | Purpose |
|---|---|
| next | React framework - serves frontend + API routes |
| react / react-dom | UI framework |
| playwright | Headless Chromium for LinkedIn scraping |
| playwright-extra | Plugin wrapper for Playwright |
| puppeteer-extra-plugin-stealth | Hides automation traces from LinkedIn |
| axios + cheerio | (unused - kept from earlier Bing scraper) |

---

## Running Locally

```bash
# Install dependencies
npm install
npx playwright install chromium

# Start dev server
npm run dev

# Or run CLI directly (visible browser - good for debugging)
npm run scan

# Run CLI headless
HEADLESS=true node index.js
```

The UI will be at `http://localhost:3000`. Open the sidebar accordion, paste your LinkedIn cookies (exported via Cookie-Editor extension → "This site only"), fill in search criteria, and scan.

---

## Security Notes

- **Cookies stay in your browser**: Stored in localStorage (12h TTL), sent to the API once to launch the scan, written to a temp file that's deleted immediately after
- **No cloud storage**: All processing is local. No telemetry, no external API calls, no credential storage
- **Output/profiles.json**: Written locally, gitignored, overwritten each scan
- **Playwright caution**: The scraper runs under your own LinkedIn session. LinkedIn rate-limits aggressively - keep counts under 50 per scan, wait between scans
- **No email/age enrichment**: LinkedIn doesn't expose these in search results. Google enrichment was attempted but blocked by CAPTCHA even with real cookies
