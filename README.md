# Weekly Sector Briefing Agent

Automated weekly business and geopolitical news briefing for the Paradigm Consulting Asia team, built with Google Alerts, Power Automate, SharePoint and Microsoft Copilot Studio.

**Status (20 Nov 2026):** news collection is working. Story selection, summaries and email generation are not built yet.

## Goal

Every Monday, send an Outlook email covering the previous week:

- About 10–15 significant stories across 10 sectors, grouped under sector headings
- For each story: headline, publication, date, link, and a short factual summary (about 2 sentences) where the source text supports one
- No duplicate coverage of the same event
- No forced equal coverage across sectors

**Sectors:** Banking, Private Equity, Property, Technology, Digital Assets & Tokenization, E-Commerce, Insurance, Professional Services, Luxury Retail, Geopolitics (China / US).

## Architecture

```
Google Alerts (30 searches, RSS delivery)
        │  each feed keeps only its latest 20 entries
        ▼
Power Automate: "Briefing - Collect alerts"   (every hour)
        │  RSS connector → extract real URL → clean text
        ▼
SharePoint list: "Briefing Articles"   (unique Link column = deduplication)
        │
        ▼
Copilot Studio agent   (run manually on Monday)        ← not built yet
        │  filter to the week → AI selects ~15 stories → AI summaries
        ▼
HTML email via Outlook (Office 365 Outlook connector)
```

### Why collect hourly instead of reading the feeds on Monday

Each Google Alerts RSS feed holds only its **20 most recent entries**. For busy searches, 20 entries can cover only a few hours, so a single weekly read would miss most of the week. The Power Automate flow saves new entries into SharePoint every hour.

## Components

### 1. Google Alerts (`feeds/`)

- `alerts_tracker.xlsx`: the 30 searches (sector, label, exact string used, RSS URL, notes)
- `feeds_30.json`: the same list in the format used by the flow's `Feeds` step
- Settings for every alert: Sources = News, Language = English, Region = Any, How many = All results, How often = As-it-happens, Deliver to = RSS feed
- Search strings follow Mark's official keyword list. Two strings had brackets added where the original was missing an operator:
  - Banking / Traditional: `("central bank" "interest rates")`
  - Property / Industrial & Logistics: `("data center" "real estate")`

### 2. Power Automate flow: `Briefing - Collect alerts`

| Step | Detail |
|---|---|
| Recurrence | Every 1 hour |
| `Feeds` (Compose) | JSON array from `feeds/feeds_30.json` |
| `For each feed` | Input: `outputs('Feeds')` |
| List all RSS feed items | Feed URL: `items('For_each_feed')?['url']`, Since: `addHours(utcNow(), -6)` |
| `For each item` | Input: `body('List_all_RSS_feed_items')` |
| Create item (SharePoint) | Writes one row per article (expressions below) |
| `Done` (Compose) | Run after: is successful **and** has failed, so that rejected duplicates don't mark the run as failed |

Field expressions in **Create item**:

| Column | Expression |
|---|---|
| Title (Headline) | `replace(replace(replace(replace(replace(replace(items('For_each_item')?['title'],'<b>',''),'</b>',''),'&quot;','"'),'&#39;',''''),'&nbsp;',' '),'&amp;','&')` |
| Link | `take(decodeUriComponent(if(contains(items('For_each_item')?['primaryLink'],'&url='),first(split(last(split(items('For_each_item')?['primaryLink'],'&url=')),'&ct=')),items('For_each_item')?['primaryLink'])),255)` |
| Published | `items('For_each_item')?['publishDate']` |
| Snippet | same cleaning as Title, applied to `['summary']` |
| Sector | `items('For_each_feed')?['sector']` |
| Search | `items('For_each_feed')?['search']` |

Uses only standard connectors (RSS, SharePoint). No premium licence or Copilot Credits needed.

### 3. SharePoint list: `Briefing Articles`

| Column | Type | Note |
|---|---|---|
| Title (shown as Headline) | Single line of text | |
| Link | Single line of text | **Enforce unique values = Yes** (deduplication) |
| Published | Date and time | Stored in UTC |
| Snippet | Multiple lines of text | Google's excerpt (often one sentence, sometimes menu text) |
| Sector | Single line of text | Sector of the search that found it, not a final classification |
| Search | Single line of text | Search label |

Currently stored on the intern's personal OneDrive (temporary). **To do:** move it to a team SharePoint site.

### 4. Copilot Studio (`copilot_studio/`)

- Agent: `TestTest` (scratch agent, Paradigm Consulting environment)
- `fetch_all_feeds_v3.yaml`: prototype topic (trigger `run feeds`) that fetches all 30 feeds directly, parses the Atom entries into `link | headline | domain | date (HKT) | snippet` lines, and removes exact duplicates. It is kept for reference; the final agent will read from the SharePoint list instead.

Lessons learned:

- Copilot Studio formulas time out after **1 second**. Anything that compares every article with every other article is too slow. Single-pass operations (`Split`, `ForAll`, `Distinct`) are fine at about 500 entries.
- Google Alerts feeds are **Atom** (`<entry>`, `<published>`, `<link href="...">`), not RSS `<item>`.
- Links are Google redirects; the real URL is between `&url=` and `&ct=` and may be percent-encoded.
- The Office 365 Outlook connector (not Outlook.com) sends HTML email from the work account.

## Known issues and noise

- Keyword alerts return off-topic results (e.g. "RWA" matching *risk-weighted assets*; non-Asia deals; consumer and lifestyle items). The AI selection step must filter for relevance.
- The same event is often covered by several outlets (e.g. PDD results appeared 5 times in one feed). The AI step must group these into one story.
- Google truncates long headlines and snippets with `...`.
- Many snippets support at most one short factual sentence. Two-sentence summaries at the quality Mark approved are not always possible from free sources; Farseer may be assessed later for gaps.

## Next steps

1. Let the flow collect for several days; measure weekly volume per sector.
2. Copilot Studio agent: read the list for the Monday–Sunday window (HKT).
3. AI prompt 1: relevance filtering, grouping by event, ranking, choosing ~15 stories, assigning final sectors.
4. AI prompt 2: grounded summaries (about 2 sentences; headline only if the snippet is too thin).
5. Build the HTML email and test it in Outlook; send the first sample to Mark.
6. Write a short operating guide.

## Security

- This repository must stay **private**. The feed URLs give read access to the alert results.
- Do not commit credentials or connection secrets. Topic YAML and flow export packages contain connection references only.
