# Weekly Sector Briefing Agent

Automated weekly business and geopolitical news briefing for the Paradigm Consulting Asia team, built with Google Alerts, Power Automate, SharePoint and Microsoft Copilot Studio.

**Status (26 Nov 2026):** end-to-end working. The agent reads the previous Monday–Sunday (HKT) from the SharePoint list, selects about 15 stories (`Briefing_Select`), writes summaries (`Briefing_Summarise`), builds an HTML email in Power Fx and sends it through Outlook. Tested at full volume (1,140 articles). First sample (19–26 Nov) prepared for Mark.

## Goal

Every Monday, send an Outlook email covering the previous week:

- About 10–15 significant stories, grouped under up to 10 sector headings
- For each story: bold headline, publication, date, a 1–2 sentence factual summary (or none if the source text is too thin), and a "Read article" link
- No duplicate coverage of the same event
- No forced equal coverage across sectors

**Sectors (fixed order):** Banking (Traditional & Digital), Private Equity, Property (Real Estate), Technology, Digital Assets & Tokenization, E-Commerce, Insurance, Professional Services, Luxury Retail, Geopolitical Developments (China / US).

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
Copilot Studio agent "WeeklyBriefing_Agent"   (run manually on Monday: "run briefing")
        │  Get items (previous Mon–Sun HKT)
        │  → Briefing_Select (choose ~15 events, group, assign sector; returns IDs only)
        │  → keep only selected rows → build event text
        │  → Briefing_Summarise (outlet, headline, summary)
        │  → story table (sector, outlet, headline, summary, link, date)
        │  → HTML email built in Power Fx
        ▼
Outlook email via Office 365 Outlook connector "Send an email (V2)"
```

### Why collect hourly instead of reading the feeds on Monday

Each Google Alerts RSS feed holds only its **20 most recent entries**. For busy searches, 20 entries can cover only a few hours, so a single weekly read would miss most of the week. The Power Automate flow saves new entries into SharePoint every hour.

### Volume

About 190 articles per day, about 1,300 per week. At ~285 characters per article line, a full week of `ArticlesText` is ~330,000–380,000 characters. `Briefing_Select` handles this in about one minute and selects from the whole week (checked by the spread of selected IDs), so **no pre-filter is used**. Fallback if a very busy week ever fails: drop youtube.com, facebook.com and linkedin.com rows before building `ArticlesText`.

## Running it

1. On Monday **after 08:00 HKT**, open WeeklyBriefing_Agent → Test pane → refresh → type `run briefing`.
2. Wait 2–3 minutes. Check the `WINDOW` line shows the previous Monday to Sunday and `stories in email=N of N` (both numbers equal).
3. Read the email that arrives in the intern mailbox before forwarding it. Check that each summary matches its headline; fix any wording by hand when forwarding.
4. If a prompt step fails with HTTP 403/500 (`DependencyFailure`), wait about 5 minutes and run again. This is a temporary AI service error, not a topic error.

Each run selects slightly different stories (AI selection is not deterministic). The run you check is the one you send.

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

### 4. Copilot Studio (`copilot_studio/weekly_briefing.yaml`, `prompts/`)

**WeeklyBriefing_Agent, topic `Weekly briefing` (trigger `run briefing`).** Nodes in order:

| # | Node | What it does |
|---|---|---|
| 1 | `set_week_start` → `Topic.WeekStart` | Previous Monday: `DateAdd(Today(), -(Weekday(Today(), StartOfWeek.Monday) - 1) - 7, TimeUnit.Days)` |
| 2 | `set_window_start` / `set_window_end` | UTC bounds of Mon 00:00–Sun 23:59 HKT: `Text(DateAdd(Topic.WeekStart, -1, TimeUnit.Days), "yyyy-mm-dd") & "T16:00:00Z"` and `Text(DateAdd(Topic.WeekStart, 6, TimeUnit.Days), "yyyy-mm-dd") & "T16:00:00Z"` |
| 3 | `dbg_window` | Shows the window (HKT and UTC) |
| 4 | **Get items** (SharePoint) → `Topic.GetItems` | `$filter: ="Published ge '" & Topic.WindowStartUtc & "' and Published lt '" & Topic.WindowEndUtc & "'"`, Order By `Published desc`, Top Count `5000` |
| 5 | `set_articles_text` → `Topic.ArticlesText` | One line per article: `ID \| search sector \| date \| domain \| headline \| snippet (300 chars)` |
| 6 | **Briefing_Select** → `Topic.Selection` | Returns `SECTOR \| PRIMARY_ID \| OTHER_IDS \| SHORT_LABEL`. IDs only, so links cannot be invented |
| 7 | `set_needed_ids` → `Topic.NeededIds` | All selected IDs as `",id1,id2,...,"` |
| 8 | `set_needed_items` → `Topic.NeededItems` | `Filter(Topic.GetItems.value, ("," & Text(ID) & ",") in Topic.NeededIds)`: one pass, ~80–110 rows kept |
| 9 | `set_events_text` → `Topic.EventsText` | Per event: `EVENT`, `PRIMARY` and `RELATED` lines (domain \| headline \| snippet), using `LookUp` on `Topic.NeededItems` |
| 10 | **Briefing_Summarise** → `Topic.Summaries` | Returns `PRIMARY_ID \| OUTLET \| HEADLINE \| SUMMARY` (summary or `-`) |
| 11 | `set_stories` → `Topic.Stories` | Table: `id, sector, outlet, headline, summary, link, published`. Sector from the Selection line, link/date from `Topic.NeededItems` |
| 12 | `set_range_text` → `Topic.RangeText` | e.g. "23 Nov – 29 Nov 2026" |
| 13 | `set_email_html` → `Topic.EmailHtml` | HTML built deterministically (see below) |
| 14 | **Send an email (V2)** (Office 365 Outlook) | To: intern mailbox, Subject: `="WEEKLY SECTOR BRIEFING \| " & Topic.RangeText`, Body: `=Topic.EmailHtml` |

Debug messages (`ITEMS`, `ARTICLES`, `SELECTION`, `NEEDED`, `EVENTS`, `SUMMARIES`, `STORIES`, `HTML ... stories in email=N of M`) are kept between the steps so a failure can be located from the Test pane.

**Outlet names** are set in `set_stories`, not left to the AI: a `Switch` on the link's domain gives fixed names for known sites (e.g. `info.gov.hk` → Hong Kong Government, `ustr.gov` → Office of the US Trade Representative, `mkbn.mk.co.kr` → Pulse by Maeil Business News). Otherwise the AI's outlet is used; if that is still a domain, the " - Publication" ending of the original headline is used, but only when it matches the domain. Add new sites to the `Switch` when they appear as raw domains.

**HTML email:** sectors in the fixed order, only sectors with stories, numbered 1..M; header "WEEKLY SECTOR BRIEFING", date range, "N stories | M sectors"; per story bold headline, `outlet | d mmm yyyy`, summary (omitted when `-`), "Read article" link; short footer. Table layout with inline CSS only (renders in Outlook). Headlines, summaries and outlets are HTML-escaped (`&`, `<`, `>`); zero-width characters (`UniChar(8203)`) are stripped.

**Prompts (`prompts/`):**

- `briefing_select.txt`: remove (no Asia link, off-topic, low value, old) → group same-event coverage → rank → select 10–15 → assign sector with tie-break rules → choose primary article. Primary must not be a press-release wire (GlobeNewswire, PR Newswire, Business Wire, Accesswire) when a news outlet covers the event, because wire headlines are often cut off.
- `briefing_summarise.txt`: outlet, cleaned headline, and a summary of two sentences, one sentence or `-`, using only facts in the event lines. Key rules: never complete text cut off with "..." (even when the missing word seems obvious); refer to a company by name or "it", not a description; never mention other coverage ("a related report").

Tested 20 Nov (70 articles) and 26 Nov (full volume, 1,140 articles; ~10 runs). Summaries checked line by line against the event text: no invented facts after the 26 Nov prompt edits.

**Prototype (reference only), agent `TestTest`:** `fetch_all_feeds_v3.yaml` fetches the 30 feeds directly and parses the Atom entries. Kept for reference; the final agent reads from the SharePoint list.

## Lessons learned

- Copilot Studio formulas time out after **1 second** per node. `LookUp` scans the whole table each time: ~85 lookups on 1,140 rows timed out, so the topic first filters down to the selected rows (one pass), then looks up in that small table. Single-pass `Split`, `ForAll`, `Filter`, `Concat` over ~1,300 rows are fine.
- Values in the topic YAML must start with `=` to be formulas; a formula typed into a text field is sent literally (HTTP 400 from SharePoint).
- Choose variables with the {x} picker and check actual output names (`Topic.GetItems`, `.text` on prompt outputs).
- Use the **Office 365 Outlook** connector, not Outlook.com. In the `SendEmailV2` YAML the inputs are `To`, `Subject`, `Body`.
- `Text()` on a SharePoint date shows Hong Kong time in the Test pane.
- AI prompts are inconsistent on mechanical tasks (e.g. outlet names). Do those in Power Fx.
- Prompt changes cause regressions: make small targeted edits and retest on more than one run.
- Google Alerts feeds are **Atom**; links are Google redirects with the real URL between `&url=` and `&ct=`.

## Known issues

- Keyword alerts return off-topic results; `Briefing_Select` filters for relevance.
- Many snippets support at most one short sentence, so some stories have no summary. Farseer may be assessed later for gaps.
- Summaries occasionally drop a qualifier (e.g. "asset managers" for "crypto asset managers"). Read before sending.
- New outlets may appear as raw domains until added to the `Switch` in `set_stories`.
- Prompt calls occasionally fail with a temporary HTTP 403 from the AI service; rerun after a few minutes.

## Next steps

1. Agree with Mark: team SharePoint site for the list, sending mailbox and recipient list, cover for the 7 Dec issue.
2. Move the list and repoint the Get items node (dataset, table).
3. Change the email `To` to the agreed recipients and the connection to the agreed mailbox.
4. Optional: remove or shorten the debug messages once runs are stable; set temperature to 0 on `Briefing_Select` if the prompt editor allows, to reduce run-to-run variation.

## Security

- This repository must stay **private**. The feed URLs give read access to the alert results.
- Do not commit credentials or connection secrets. Topic YAML and flow export packages contain connection references only.
