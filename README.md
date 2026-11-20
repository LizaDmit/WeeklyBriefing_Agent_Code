# Weekly Sector Briefing Agent

Automated weekly business and geopolitical news briefing for the Paradigm Consulting Asia team, built with Google Alerts, Power Automate, SharePoint and Microsoft Copilot Studio.

**Status (20 Nov 2026):** news collection is working (hourly flow into SharePoint). The agent reads the last 7 days from the list, selects stories (`Briefing_Select`) and builds event text for summaries. The summary prompt (`Briefing_Summarise`) is written and tested in the prompt editor but not yet connected. HTML email generation is not built yet.

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
Copilot Studio agent "WeeklyBriefing_Agent"   (run manually on Monday)
        │  Get items (last 7 days) → Briefing_Select (choose ~15, group, sector)
        │  → build event text → Briefing_Summarise (outlet, headline, summary)  ← not connected yet
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

### 4. Copilot Studio (`copilot_studio/`, `prompts/`)

**WeeklyBriefing_Agent, topic `Weekly briefing` (trigger `run briefing`):**

1. **Get items** (SharePoint) from `Briefing Articles`: Filter Query set as a formula in the code editor (`$filter: ="Published ge '" & Text(DateAdd(Now(), -7, TimeUnit.Days), "yyyy-mm-dd") & "T00:00:00Z'"`), Order By `Published desc`, Top Count `5000`. A filter typed as plain text is sent literally and fails with HTTP 400.
2. **ArticlesText**: one line per article: `ID | search sector | date | domain | headline | snippet (300 chars)`.
3. **Briefing_Select** prompt (`prompts/briefing_select.txt`): returns `SECTOR | PRIMARY_ID | OTHER_IDS | SHORT_LABEL`. The AI returns only SharePoint IDs; headlines and links are looked up afterwards, so links cannot be invented.
4. **EventsText**: for each selected event, the PRIMARY article and all RELATED (grouped) articles with their snippets, built with `LookUp` on the IDs.
5. **Briefing_Summarise** prompt (`prompts/briefing_summarise.txt`): returns `PRIMARY_ID | OUTLET | HEADLINE | SUMMARY`; the summary is two sentences, one sentence, or `-` depending on the facts available.

Tested on 70 articles (20 Nov): 10 events selected, noise removed, PDD coverage from 5 outlets grouped into one event.

**Prototype (reference only), agent `TestTest`:**

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

1. Check weekly volume (row count) and the flow's run history; decide whether a pre-filter is needed before Briefing_Select (about 285 characters per article).
2. Connect Briefing_Summarise in the topic and validate it on the real events.
3. Parse the summary output and look up link and date for each PRIMARY_ID.
4. Build the HTML email (sector headings, headline, outlet, date, summary, "Read article" link) and test it in Outlook.
5. Change the date filter to the exact Monday–Sunday window (HKT).
6. Send the first agent-generated sample to Mark.
7. Write a short operating guide.

## Security

- This repository must stay **private**. The feed URLs give read access to the alert results.
- Do not commit credentials or connection secrets. Topic YAML and flow export packages contain connection references only.
