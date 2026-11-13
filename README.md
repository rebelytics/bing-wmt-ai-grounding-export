# bing-wmt-ai-grounding-export

Open-source skill that **exports AI Performance data (cited pages and per-page grounding queries) from Bing Webmaster Tools at scale**, via the report's own authenticated JSON API rather than clicking download buttons. The API approach is faster, more reliable, and far gentler on rate limits when extracting hundreds of pages at once. Load it into any MCP-capable AI agent (Claude, Cursor, Codex, n8n, etc.) that can drive a browser the user is signed in with.

Part of the [Open Skill Library](https://github.com/rebelytics/one-skill-to-rule-them-all/blob/main/OPEN-SKILL-LIBRARY.md), the open-source agent skills maintained with Task Observer.

## When it triggers

Whenever the user wants to pull the AI Pages Report, AI Search/Grounding Queries Report, or "grounding query report" from Bing Webmaster Tools (bing.com/webmasters/aiperformance) — especially when they want every page matching a URL filter, a consolidated spreadsheet of grounding queries per page, or any bulk extraction that must not get the account rate-limited. Also on mentions of "Bing AI performance", "grounding queries", "cited pages", "AI citations" in a Bing Webmaster Tools context, or requests to make such an extraction repeatable.

## What this repo contains

- `SKILL.md` — when to use, the prerequisites, the terms of use, what to confirm up front, the API (endpoints, request shapes, the query modes, the CSV export, daily series and period compare), the extraction procedure, building the spreadsheet, the pitfalls, the pre-flight, and the pointer list for the reference files.
- `references/paginated-post-variant.md` — the variant for server-paginated reports behind an opaque POST body.
- `references/browser-data-out.md` — getting data out of the browser: when rows need to leave it at all, and the content-filter workarounds.
- `references/topic-portfolio.md` — the Topic Portfolio view: topic sizes and opportunity queries, where other sites are cited and yours is not.
- `LICENSE` — CC BY 4.0.

The analysis companion [`bing-grounding-query-analysis`](https://github.com/rebelytics/bing-grounding-query-analysis) turns the spreadsheet this skill produces into decisions.

## Install

Install the whole skill directory — `SKILL.md` and `references/` must travel together — into your MCP-capable agent's skills directory, following the client-specific path, and restart the client.

## Contributing

Open an issue or a PR at [github.com/rebelytics/bing-wmt-ai-grounding-export](https://github.com/rebelytics/bing-wmt-ai-grounding-export).

## Credits

- Original author: [Eoghan Henn](https://www.rebelytics.com) / [LinkedIn](https://www.linkedin.com/in/eoghanhenn)

## License

CC BY 4.0 — the licence text is in [LICENSE](./LICENSE): the full legal code, unmodified, as Creative Commons publishes it; the [human-readable summary](https://creativecommons.org/licenses/by/4.0/) says what it allows. Use it, fork it, adapt it, monetise it. Keep the attribution.

Copyright (c) 2026 Eoghan Henn / rebelytics.com
