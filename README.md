# Weekly Sector Briefing Agent

Copilot Studio agent that produces a weekly business and geopolitical news briefing for Paradigm Consulting Asia.

## Pipeline
1. Fetch 30 Google Alerts RSS feeds (10 sectors x 3 searches)
2. Parse entries (headline, domain, date in HKT, link, snippet)
3. Remove duplicate links
4. AI: select ~15 stories and assign sectors (to do)
5. AI: short factual summaries from snippets (to do)
6. Build HTML email and send via Outlook (to do)

## Files
- `agent.mcs.yml`, `settings.mcs.yml`: agent configuration (Copilot Studio VS Code extension)
- `topics/`: topic YAML
- `alerts_tracker.xlsx`: the 30 Google Alerts search strings and feed URLs

## Status
Platform checks passed 20 Nov 2026 (prompt, HTTP fetch, Outlook HTML email). Fetch + parse working.
