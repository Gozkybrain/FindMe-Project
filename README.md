# FindMe - LinkedIn Profile Discovery Tool

FindMe lets you search LinkedIn for targeted profiles using **your own LinkedIn session cookies**. No APIs, no third-party data brokers, no cloud dependency, no paid services. Everything runs locally.

## Two Versions

### 1. Desktop App (.dmg / .exe)
Download and run - no terminal, no Node.js, no technical knowledge needed. Built with **Electron**, packaged as a native macOS or Windows app. The Chromium browser engine is bundled inside.

- **macOS**: [FindMe-mac.dmg](https://github.com/Gozkybrain/FindMe-Desktop/releases/latest/download/FindMe-mac.dmg)
- **Windows**: [FindMe-Windows.exe](https://github.com/Gozkybrain/FindMe-Desktop/releases/latest/download/FindMe-Windows.exe)

### 2. Local Webpage (Developer)
Clone the repo, run `npm run dev`, open `http://localhost:3000`. The same UI, running in your browser instead of an Electron window.

```bash
git clone https://github.com/Gozkybrain/FindMe-Desktop.git
cd FindMe-Desktop
npm install
npm run dev
```

## How It Works

### Step 1 - Export Your LinkedIn Cookies
Install the **Cookie-Editor** browser extension (Chrome/Firefox). Log into linkedin.com, open the extension, click **Export** → saves a `cookies.json` file.

### Step 2 - Paste Cookies Into FindMe
Open the app, go to the **Import** tab, paste the JSON. Your session is now loaded - the app will browse LinkedIn as you.

### Step 3 - Search
Fill in target criteria:
- **Location** (e.g. "San Francisco Bay Area")
- **Industry** (e.g. "Technology")
- **Skills** (e.g. "Python, React")
- **Age range** (optional)
- **Profile count** (1–50 per scan)

### Step 4 - Get Results
The app launches a real Chromium browser (headless), logs into LinkedIn with your cookies, runs the search, and extracts:
- Name
- Headline / title
- Location
- LinkedIn profile URL

Results appear in a table, can be exported as JSON or copied individually.

## What Makes It Different

| Feature | FindMe | Alternatives |
|---|---|---|
| **Data source** | Real-time LinkedIn search | Stale databases, data brokers |
| **Session control** | Your own cookies | They use their own (risky) |
| **Privacy** | 100% local, nothing leaves your machine | Data passes through their servers |
| **Cost** | Free | $50–500/mo per tool |
| **Accuracy** | Current, real LinkedIn data | Often outdated or wrong |

## Important

- **Cannot deploy to Vercel / Netlify** - requires Playwright + Chromium (~300MB binary). Must run locally or on a VPS.
- **Rate limits** depend on your LinkedIn account activity. Start with 10–20 profiles per scan.
- **Cookies expire** after ~12 hours. Re-export from LinkedIn to refresh.
- **No user data is stored or transmitted** - cookies stay in browser memory, temporary files are deleted after each scan.

## Tech Stack

- **Frontend**: Next.js (plain JSX, no TypeScript/Tailwind)
- **Backend**: Next.js API routes (Node.js)
- **Browser automation**: Playwright with stealth plugin
- **Desktop wrapper**: Electron + electron-builder
- **CI/CD**: GitHub Actions → builds macOS (.dmg) + Windows (.exe) on push

