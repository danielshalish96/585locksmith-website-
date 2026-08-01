# 585 Locksmith SEO Agent — Build, Host & Publish Plan

**Goal:** An agent that runs daily and weekly on its own, monitors SEO performance, and produces content improvements — at near-zero infrastructure cost.

**Total expected cost: $0 infrastructure + $1–3/month in API usage.**

---

## Read This First — One Important Correction

You asked for an agent that improves SEO "by himself." I need to be direct with you about one thing before you build it:

**Do not let the agent auto-publish content to your live site.**

Google's spam policies explicitly target *scaled content abuse* — mass-produced pages published without human review. An agent that automatically pushes 15 town pages live is a fast route to a manual penalty, and recovering from one takes months. For a business whose entire lead flow depends on ranking, that risk is not worth the two minutes a week it saves you.

**The right architecture is:** the agent runs fully autonomously, does all the monitoring, research, and writing, and then **opens a pull request** with its proposed changes. You get a phone notification, glance at it, and tap Merge. Total human time: roughly **2 minutes per week**.

Everything below is built around that model. It gives you 98% of the autonomy with none of the penalty risk.

---

## The Stack (all free tiers)

| Layer | Tool | Cost |
|---|---|---|
| Code storage | GitHub (private repo) | Free |
| Scheduler + compute | GitHub Actions | Free (2,000 min/mo — you'll use ~30) |
| AI model | Claude API (Haiku 4.5) | ~$1–3/month |
| Ranking data | Google Search Console API | Free |
| Website hosting | Cloudflare Pages or Netlify | Free |
| Analytics | Google Search Console + GA4 | Free |

**Why GitHub Actions:** It is a scheduler and a server in one, it's free, and it requires no credit card, no VPS, and no server maintenance. This is the cheapest legitimate way to run a recurring agent.

---

# PHASE 1 — Foundation (Do This First)

*Estimated time: 2–3 hours. Do not skip to Phase 2 — the agent needs this data to function.*

### Step 1.1 — Publish the website

Your site must be live before anything else. Take the `585locksmith.html` file already built:

1. Rename it to `index.html`
2. Go to [Cloudflare Pages](https://pages.cloudflare.com) → Create project → Upload assets
3. Drag the file in, deploy
4. Buy a domain (~$10–12/year at Cloudflare Registrar — cheapest, sold at cost) and connect it

**This is the only unavoidable recurring cost in the entire plan.**

### Step 1.2 — Set up Google Search Console

This is the agent's eyes. Without it, the agent is guessing.

1. Go to [Google Search Console](https://search.google.com/search-console)
2. Add property → Domain → `585locksmith.com`
3. Verify via DNS (Cloudflare makes this one click)
4. Submit your sitemap: `585locksmith.com/sitemap.xml`

Search Console gives you real impressions, clicks, and average position data — for free, straight from Google. This is more reliable than any paid rank tracker.

### Step 1.3 — Set up Google Business Profile

Do this manually and do it properly. It is the single highest-impact thing on this entire list for a local locksmith.

1. Go to [Google Business Profile](https://business.google.com)
2. Create the listing using the checklist in your `CLAUDE-SEO.md`
3. Complete verification (Google mails a postcard — takes 5–14 days, start now)

**Note on automation:** The Google Business Profile API is gated behind an application process and is not practical for a single small business. The agent will *draft* your GBP posts and review replies; you'll paste them in. That takes about 60 seconds a week.

### Step 1.4 — Get a Claude API key

1. Go to [platform.claude.com](https://platform.claude.com)
2. Create an account → API Keys → Create Key
3. Add $5 in credits — at your usage rate, this will last several months
4. Copy the key somewhere safe; you'll need it in Step 2.4

---

# PHASE 2 — Build the Agent

*Estimated time: 2–3 hours.*

### Step 2.1 — Create the repository

In VS Code:

1. Create a new folder: `585locksmith-seo`
2. Open it in VS Code (File → Open Folder)
3. Open the terminal (Ctrl + `) and run:

```bash
git init
npm init -y
npm install @anthropic-ai/sdk googleapis
```

4. Create a private repo on GitHub, then connect it:

```bash
git remote add origin https://github.com/YOUR-USERNAME/585locksmith-seo.git
```

### Step 2.2 — Set up the folder structure

```
585locksmith-seo/
├── .github/
│   └── workflows/
│       ├── daily-monitor.yml
│       └── weekly-content.yml
├── agent/
│   ├── CLAUDE-SEO.md          ← the agent brief you already have
│   ├── daily-monitor.js
│   ├── weekly-content.js
│   └── lib/
│       ├── claude.js
│       └── search-console.js
├── site/
│   └── index.html             ← your live website
├── output/
│   ├── reports/               ← agent's weekly reports land here
│   └── drafts/                ← generated content awaiting review
├── .env                       ← local secrets (never commit this)
├── .gitignore
└── package.json
```

Copy your existing `CLAUDE-SEO.md` into `agent/`. That file is the agent's brain — it's what gets fed into every API call as the system prompt.

### Step 2.3 — Create `.gitignore`

```
node_modules/
.env
*.log
```

**Critical:** if you commit your API key to GitHub, it will be scraped and abused within hours. This file prevents that.

### Step 2.4 — Store secrets in GitHub

Never put keys in code. In your GitHub repo:

**Settings → Secrets and variables → Actions → New repository secret**

Add these three:

| Secret name | Value |
|---|---|
| `ANTHROPIC_API_KEY` | Your Claude API key from Step 1.4 |
| `GSC_CLIENT_EMAIL` | From your Google service account JSON |
| `GSC_PRIVATE_KEY` | From your Google service account JSON |

*(To get the Google credentials: Google Cloud Console → Create Project → Enable "Search Console API" → Create Service Account → Download JSON key → add that service account's email as a user in Search Console.)*

---

# PHASE 3 — Write the Agent Scripts

### Step 3.1 — The Claude wrapper (`agent/lib/claude.js`)

This is the reusable function every script calls.

```js
import Anthropic from '@anthropic-ai/sdk';
import fs from 'fs';
import path from 'path';

const client = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

const SYSTEM_PROMPT = fs.readFileSync(
  path.join(process.cwd(), 'agent/CLAUDE-SEO.md'),
  'utf-8'
);

export async function askAgent(task, model = 'claude-haiku-4-5-20251001') {
  const response = await client.messages.create({
    model,
    max_tokens: 4000,
    system: SYSTEM_PROMPT,
    messages: [{ role: 'user', content: task }],
  });
  return response.content[0].text;
}
```

**Model choice:** Start with Haiku 4.5 ($1 in / $5 out per million tokens) — it is more than capable for SEO content and monitoring. If you find the writing quality lacking on long-form blog posts, switch that one script to Sonnet. Don't use the expensive model for everything.

### Step 3.2 — Search Console reader (`agent/lib/search-console.js`)

```js
import { google } from 'googleapis';

export async function getSearchData(days = 28) {
  const auth = new google.auth.JWT({
    email: process.env.GSC_CLIENT_EMAIL,
    key: process.env.GSC_PRIVATE_KEY.replace(/\\n/g, '\n'),
    scopes: ['https://www.googleapis.com/auth/webmasters.readonly'],
  });

  const searchconsole = google.searchconsole({ version: 'v1', auth });

  const endDate = new Date().toISOString().split('T')[0];
  const startDate = new Date(Date.now() - days * 86400000)
    .toISOString().split('T')[0];

  const res = await searchconsole.searchanalytics.query({
    siteUrl: 'sc-domain:585locksmith.com',
    requestBody: {
      startDate,
      endDate,
      dimensions: ['query', 'page'],
      rowLimit: 100,
    },
  });

  return res.data.rows || [];
}
```

### Step 3.3 — Daily monitor (`agent/daily-monitor.js`)

Runs every morning. Cheap, fast, catches problems early.

```js
import { askAgent } from './lib/claude.js';
import { getSearchData } from './lib/search-console.js';
import fs from 'fs';

const data = await getSearchData(7);

const task = `
Here is the last 7 days of Google Search Console data for 585locksmith.com:

${JSON.stringify(data.slice(0, 50), null, 2)}

Analyze it and report:
1. Any keyword that dropped 3+ positions vs. typical performance
2. Any page with high impressions but low click-through rate (an easy title/meta fix)
3. Any new keyword we're ranking for that we should build a dedicated page around
4. ONE specific action to take today

Keep it under 200 words. Be specific. No filler.
`;

const report = await askAgent(task);
const date = new Date().toISOString().split('T')[0];

fs.mkdirSync('output/reports', { recursive: true });
fs.writeFileSync(`output/reports/daily-${date}.md`, report);
console.log(report);
```

### Step 3.4 — Weekly content generator (`agent/weekly-content.js`)

The workhorse. Produces the actual SEO assets.

```js
import { askAgent } from './lib/claude.js';
import { getSearchData } from './lib/search-console.js';
import fs from 'fs';

const data = await getSearchData(28);
const date = new Date().toISOString().split('T')[0];

const tasks = {
  'blog-post': `Based on this Search Console data:\n${JSON.stringify(data.slice(0, 40))}\n\nWrite ONE complete, publish-ready blog post targeting the highest-opportunity long-tail keyword you identify. Follow the blog post structure in your brief. Output clean HTML with proper heading tags, a meta description, and a title tag.`,

  'gbp-post': `Write one Google Business Profile post for this week. Follow the GBP post templates in your brief. Under 750 characters. Make it seasonally relevant to Rochester NY for the current month.`,

  'action-items': `Based on this month's Search Console data:\n${JSON.stringify(data.slice(0, 40))}\n\nGive me the 3 highest-impact SEO actions for 585 Locksmith this week, ranked. For each: what to do, why it matters, and expected impact. Be specific and brief.`,
};

fs.mkdirSync(`output/drafts/${date}`, { recursive: true });

for (const [name, prompt] of Object.entries(tasks)) {
  const result = await askAgent(prompt);
  fs.writeFileSync(`output/drafts/${date}/${name}.md`, result);
  console.log(`✓ Generated ${name}`);
}
```

---

# PHASE 4 — Schedule It (The Autonomous Part)

### Step 4.1 — Daily monitor workflow

Create `.github/workflows/daily-monitor.yml`:

```yaml
name: Daily SEO Monitor

on:
  schedule:
    - cron: '0 11 * * *'   # 7am Eastern, every day
  workflow_dispatch:        # lets you trigger it manually too

jobs:
  monitor:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'

      - run: npm ci

      - name: Run daily monitor
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          GSC_CLIENT_EMAIL: ${{ secrets.GSC_CLIENT_EMAIL }}
          GSC_PRIVATE_KEY: ${{ secrets.GSC_PRIVATE_KEY }}
        run: node agent/daily-monitor.js

      - name: Commit report
        run: |
          git config user.name "SEO Agent"
          git config user.email "agent@585locksmith.com"
          git add output/
          git diff --staged --quiet || git commit -m "Daily SEO report $(date +%F)"
          git push
```

### Step 4.2 — Weekly content workflow

Create `.github/workflows/weekly-content.yml`:

```yaml
name: Weekly SEO Content

on:
  schedule:
    - cron: '0 12 * * 1'   # 8am Eastern, every Monday
  workflow_dispatch:

jobs:
  generate:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'

      - run: npm ci

      - name: Generate weekly content
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          GSC_CLIENT_EMAIL: ${{ secrets.GSC_CLIENT_EMAIL }}
          GSC_PRIVATE_KEY: ${{ secrets.GSC_PRIVATE_KEY }}
        run: node agent/weekly-content.js

      - name: Open pull request
        uses: peter-evans/create-pull-request@v6
        with:
          commit-message: "Weekly SEO content — $(date +%F)"
          branch: seo/weekly-$(date +%F)
          title: "🔍 Weekly SEO Package — Review & Merge"
          body: |
            The SEO agent generated this week's content.

            **Before merging, check:**
            - Blog post reads naturally and is factually correct
            - No invented claims about pricing or response times
            - GBP post is ready to copy-paste

            Merge to publish the blog post.
```

**This is the whole autonomy loop.** Every Monday at 8am the agent pulls your real Google data, writes content, and opens a PR. You get a GitHub notification on your phone, skim it, tap Merge.

### Step 4.3 — Test before trusting it

Do not wait a week to find out it's broken. In your repo:

**Actions → select the workflow → Run workflow**

Run both manually. Fix any errors now. Then let the schedule take over.

---

# PHASE 5 — Your Weekly Routine (2 Minutes)

Once running, here's your entire ongoing involvement:

**Monday morning:**
1. Open the PR notification → skim the blog post → Merge if it reads well *(60 sec)*
2. Copy the GBP post → paste into Google Business Profile *(30 sec)*
3. Skim the action items → note anything worth doing *(30 sec)*

**After every job you complete:**
- Text the customer the review request from your `CLAUDE-SEO.md` *(15 sec)*

That last one is not optional. **Reviews will move your rankings faster than everything else in this plan combined.** The agent cannot do it for you — it requires you to have actually done the job.

---

# Cost Breakdown (Honest Numbers)

| Item | Monthly cost |
|---|---|
| GitHub Actions | $0 (you'll use ~30 of 2,000 free minutes) |
| Cloudflare Pages hosting | $0 |
| Google Search Console | $0 |
| Google Business Profile | $0 |
| Claude API — daily runs (~30/mo, small) | ~$0.30 |
| Claude API — weekly runs (~4/mo, larger) | ~$0.50 |
| **Total recurring** | **Under $1/month** |
| Domain name | ~$10/year |

Even if you switch to Sonnet for content quality and triple your usage, you're looking at $3–5/month. Your $5 in API credits from Step 1.4 will genuinely last months.

---

# Build Order — Do It In This Sequence

**Week 1**
1. Publish site to Cloudflare Pages, connect domain
2. Set up Google Search Console
3. Create and verify Google Business Profile *(start the postcard now — it's the long pole)*

**Week 2**
4. Create GitHub repo and folder structure
5. Get Claude API key, add all three GitHub secrets
6. Write `claude.js` and `search-console.js`, test locally

**Week 3**
7. Write `daily-monitor.js`, run it locally until it works
8. Write `weekly-content.js`, run it locally until it works

**Week 4**
9. Add both workflow files
10. Trigger both manually via `workflow_dispatch` and fix anything broken
11. Let the schedule take over

**Ongoing** — 2 minutes every Monday.

---

# What This Agent Cannot Do

Be clear-eyed about the boundaries:

- **It cannot post to Google Business Profile automatically.** API access is gated. It drafts; you paste.
- **It cannot generate reviews.** Only real customers can, and only after real jobs.
- **It cannot build backlinks.** That requires outreach and relationships.
- **It cannot fix a bad Google Business Profile.** Get that right manually in Phase 1.
- **It should not publish without your glance.** See the note at the top.

---

# The One Thing That Matters Most

If you only do one thing from this entire document: **set up Google Business Profile properly and ask every single customer for a review.**

For a local locksmith, the Google Map Pack drives the overwhelming majority of emergency calls. Someone locked out at midnight taps the first result with good reviews — they do not scroll to your blog post. The agent is a real multiplier on top of that foundation, but it is not a substitute for it.

Build the foundation first. Then let the agent compound it.
