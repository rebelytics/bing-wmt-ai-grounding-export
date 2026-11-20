---
name: bing-wmt-ai-grounding-export
description: Export AI Performance data (cited pages and per-page grounding queries) from Bing Webmaster Tools at scale, through the same JSON requests the report page itself sends rather than clicking download buttons. Load this skill whenever the user wants to pull the AI Pages Report, AI Search/Grounding Queries Report, or "grounding query report" from Bing Webmaster Tools (bing.com/webmasters/aiperformance) — especially when they want every page matching a URL filter, a consolidated spreadsheet of grounding queries per page, or any bulk extraction that must not get the account rate-limited. Also trigger on mentions of "Bing AI performance", "grounding queries", "cited pages", "AI citations" in a Bing Webmaster Tools context, or requests to make such an extraction repeatable.
license: CC-BY-4.0
metadata:
  author: Eoghan Henn and contributors
  source: https://github.com/rebelytics/bing-wmt-ai-grounding-export
  version: "1.1.1"
---

# Bing Webmaster Tools — AI Performance / Grounding Query Export

This skill pulls Bing Webmaster Tools AI Performance data — the cited-pages
report and the per-page grounding-query breakdown — through the report's own
authenticated JSON API instead of driving the UI download buttons. The API
approach is faster, more reliable, and far gentler on rate limits when
extracting hundreds of pages at once.

## Reference files in this skill — load on demand

- `references/paginated-post-variant.md` — Variant: server-paginated reports behind an opaque POST body. **Load when the report is server-paginated behind an opaque POST body rather than the per-page API above.**
- `references/browser-data-out.md` — Getting data out of the browser (content-filter workarounds). **Load before moving extracted results out of the browser, and whenever a content filter blocks or truncates a returned payload.**
- `references/topic-portfolio.md` — The Topic Portfolio tab: topics and opportunity queries. **Load when the user asks for topics, opportunity queries or queries the site is not cited for, and when a completeness union needs the per-topic query lists.**

## When to use

The user wants the AI Pages Report and/or the per-page grounding-query
breakdown for a Bing-verified site — often filtered to a URL pattern and a date
range — delivered as a consolidated spreadsheet. A typical scope is "a set of
pages matched by a URL substring filter" on the target site. The naive path
("click each page, hit download") does not scale to hundreds of pages and risks
throttling. This skill replicates the report's own API calls instead.

## Prerequisites

- A browser the agent can drive (e.g. **Claude in Chrome**, or any equivalent
  browser-automation capability), with the user already **signed in** to Bing
  Webmaster Tools for the target site. Confirm a browser is connected
  (`list_connected_browsers` or equivalent), then navigate to the report URL.
- The report URL looks like:
  `https://www.bing.com/webmasters/aiperformance?siteUrl=https%3A%2F%2F<SITE>%2F&activeTab=AIPagesReport`

## Terms of use

Microsoft publishes no API for AI Performance data. The requests this skill
sends are the ones the report page itself makes, in the signed-in user's own
session, and they read only data that account can already see in the UI. Use
falls under the Microsoft Services Agreement, which covers Bing Webmaster:
pace every loop as the Pitfalls section describes, never share the session or
work around a limit the service sets, and check the agreement for your own use
— commercial use of Bing material in particular — before running at scale. If
Microsoft publishes an API for this data, prefer it.

## Always confirm up front

Before any bulk run, confirm the following with the user (a structured question
widget works well here):

1. **Filter** — what scopes the pages? Usually a URL substring typed into the
   "Search your page" box; it is sent to the API as `AIFilterKeyword`.
2. **Date range** — 7D / 30D / 3M / 6M / custom. The API wants HTTP date
   strings (GMT), e.g. `"Wed, 17 Dec 2025 00:00:00 GMT"`. The API reaches
   further back than the UI's range buttons; check the earliest date it
   returns. A range that starts before the data does is clamped silently,
   not refused.
3. **Page attribution — does the deliverable need to know *which* page each
   query cited?** Ask this before anything else about scale. If the answer is
   no, the whole job is the site-wide list (see the extraction procedure)
   and the filter, pacing and duration questions mostly fall away.
   Users often describe the output as "all the grounding queries" when they
   only want the distinct query strings.
4. **Output** — consolidated spreadsheet (default) vs per-page files.
5. **Scope check** — confirm the expected page count before any bulk run so the
   pacing/duration is known. Only relevant if attribution is needed.

## The API (the requests the report page itself sends; stable at last verification — re-check one request in the browser's network panel before a bulk run)

All calls are same-origin `POST` (except the token GET), JSON body, cookies via
`credentials:'include'`, and require an anti-forgery header.

**CSRF token** — `GET /webmasters/auth/token` returns a ~32-char string. Send it
as the `X-CSRF-Token` header on every POST. Without it: `400 "No anti-forgery
token found in X-CSRF-Token header"`. Capture it into a page variable; never
print it.

**Page list** — `POST /webmasters/api/aiperformance/pages/stats`
```json
{
  "SiteUrl": "https://<site>/",
  "DateRange": {"BeginTimeStamp": "<httpdate GMT>", "EndTimeStamp": "<httpdate GMT>"},
  "Pagination": {"PageNum": 1, "PageSize": 300},
  "SortParameters": {"SortField": "Citations", "SortOrder": "Desc"},
  "AIFilterKeyword": "<url substring filter, or ''>",
  "Query": "", "Page": ""
}
```
Returns `{ "Pages": [{"PageUrl","Citations"}], "TotalNumOfRecords", "SiteUrl" }`.
`PageSize` is **hard-capped at 500** — anything larger returns
`400 InvalidRequestParameters` ("The field PageSize must be between 1 and
500"). For larger page sets, paginate: increment `PageNum` and keep fetching
until the collected count reaches `TotalNumOfRecords`.

`pages/stats` also takes `Query: "<grounding query>"` and then returns the
pages cited for that query: attribution in the reverse direction (step 6).
It knows only queries in the site-wide list below. A query found only in a
per-page or per-topic list returns 404.

**Grounding queries** — `POST /webmasters/api/aiperformance/searchqueries/stats`
with the **same body shape**. The `Page` field selects the scope, and it answers
in two distinct modes:

| Mode | `Page` value | Requests | Returns |
|---|---|---|---|
| **Site-wide** | `""` (empty) | 1 per 500 rows | The site's largest single query list — **no page attribution, and not the full set** |
| **Per-page** | a specific page URL | 1 per page | That page's queries, with attribution |

The site-wide mode is the list the report's "List By: Grounding Queries"
view shows by default. It paginates like the page list: `PageSize` up to
500, `PageNum` until the collected count reaches `TotalNumOfRecords`. It is
the cheap path to a site's distinct query strings, where the per-page loop
needs hundreds of requests and the whole quota-pacing apparatus below.
**It is not the full set.** Per-page lists and the per-topic lists of the
Topic Portfolio tab each hold queries absent from it, some with more
citations than queries it does include. The lists are sampled separately,
and none is a superset of another (observed on one account's properties at
last verification). When completeness matters, union the site-wide list
with the per-page and per-topic lists, deduplicate on the query string, and
report each source's count.

Set `AIFilterKeyword: ""` for both modes.
Returns `{ "Queries": [{"GroundingQuery","Intent","Citations","TopicClusters","Topics","CitationRate"}], "TotalNumOfRecords", "SiteUrl" }`.
Re-check population per column on every run: a column that was empty last
time can be the most informative one now.
- `404` for a page = **no grounding-query breakdown available** for that page in
  the period. This is normal (the report is sampled), not an error or an
  encoding bug — verified with clean URLs. Record as "no data" and move on.
- `Intent` is an open list. Values seen include Informational, Research,
  Commercial, Navigational, Local, Learn and Solve, Comparison, Planning,
  Utility, Live Event, Media, Creation, Others, and an empty string. Count
  the empty value as its own bucket, and never hard-code the list.
- **`CitationRate` is the UI's "Citation Share"** (0.25 shows as 25%). Bing
  publishes no definition. Its denominator is per property: on one owner's
  properties, the same query's shares summed above 100%, and
  `Citations ÷ CitationRate` gave a different implied total on each. Use it
  to rank queries within one property only. Never compare or sum it across
  properties.
- **Either `Topics` or `TopicClusters` may
  be the populated label column** — which one carries the label varies by
  dataset and report version (extractions have seen both variants: `Topics`
  populated with `TopicClusters` empty, and the exact inverse). Treat both as
  candidate label columns and let a per-dataset population check decide which
  to use; never assume the same column is populated as last time.
  `TopicClusters` arrives as an array. Join it with an explicit separator
  before any flat export, because default stringification changes the value.
- **Always-empty columns are a trap for delimited exfiltration.** In practice
  one of `Topics`/`TopicClusters` is empty for *every* row. Such a column is
  harmless in JSON but breaks whitespace-collapsing exfiltration (see
  below), so handle it deliberately.

The same model powers the UI: the "List By: Grounding Queries" toggle hits
`searchqueries/stats`; the "Pages" toggle hits `pages/stats`. The "Search your
page" box just sets `AIFilterKeyword`.

The endpoints below were observed on one account's properties at last
verification. Re-check one request before relying on any of them.

**CSV export** — `POST .../searchqueries/stats/export` and
`.../pages/stats/export` take the same body without `Pagination` and answer
`text/csv` with every row in one request. Called through `fetch` in the
page and read as text, they trigger no download. Columns:
`"Grounding Query","Intent","Topic","Citations","Citation Share"` and
`"Page","Citations"`. Prefer the export for any unattributed list.
**The export does not escape double quotes inside a value**, so a standard
CSV parser silently merges rows. Parse line by line instead: strip the
outer quotes, split on `","`, assert five fields per query row, and check
the row count against `TotalNumOfRecords` from one paginated call. An
invented example: `"is "proof" the same as rise","Informational","Baking","12","0.041"`
splits into five fields this way and into a broken row any other way.

**Daily series** — `citationstats` returns the site's daily `Citations` and
`UniqueCitedPages`, plus `TotalCitations`, `TotalUniqueCitedPages` and
`AverageUniqueCitedPages`. `citationstats/filtered` returns the daily
series for one `Page` or one `Query`. It **ignores `AIFilterKeyword`** and
silently returns the site total instead, so filter by `Page` or `Query`
only, and check that a filtered series differs from the site series.

**Period compare** — `pages/compare/stats` and `searchqueries/compare/stats`
take `LastDateRange` and `PrevDateRange` in place of `DateRange`. The page
view returns `ComparedPages` rows of `{PageUrl, LastCitations, PrevCitations}`.
Sort only on `Page`/`Query`, `LastCitations`, `PrevCitations` or
`DiffCitations`; `Citations` returns 400. Page sums fall short of the
`citationstats` totals for the same months, so never report them as site
totals. The compare views have `/export` endpoints too.

**Topic Portfolio tab** — a second report tab with its own endpoints and the
only view of queries the site is *not* cited for. Load
`references/topic-portfolio.md` before using it.

## Extraction procedure

1. Navigate to the report; confirm signed in (`document.title` contains "AI
   Performance", body has no "Sign in").
2. `GET /webmasters/auth/token` → store token in `window.__tok`.
3. **Decide whether the task needs page attribution — before building any
   loop.** This is the fork that determines whether the rest of this procedure
   applies at all.
   - **No attribution needed** (the deliverable is "what queries is this site
     cited for" — demand signals, prompt-set seeding, query-theme analysis):
     pull the **site-wide list** — the `searchqueries/stats/export` CSV, or
     `searchqueries/stats` with `Page: ""` paginated to `TotalNumOfRecords` —
     and stop. A few requests, seconds, no pacing, no quota exposure, no
     sweep. The per-page loop then becomes an optional deepening step you
     run only if the site-wide list turns out to be too thin, or the
     deliverable needs the full set (the union rule above), not the entry
     point.
   - **Attribution needed** (the deliverable is a page→query mapping, or
     per-page citation counts): continue with the paced loop from step 4.
   - **Unsure, or the user's ask could go either way:** pull the site-wide list
     anyway first. It costs a few requests, and its output tells you how much the
     loop would add. Probing the unfiltered call before committing to the loop
     is the general rule here — where a parameter identifies a scope, the empty
     value is worth trying, because an aggregate the documentation never
     mentions can eliminate the entire N-request job and the rate-limit risk
     this procedure exists to manage.
4. `pages/stats` with the URL filter, paginating at `PageSize: 500` (the hard
   cap) until the collected count reaches `TotalNumOfRecords` → store the page
   array in `window.__pages` (or take the `pages/stats/export` CSV in one
   request). Confirm the count matches the user's expectation.
5. **Budget the quota before starting.** The API enforces an **hourly quota
   (~1,000 requests/hour observed)** on top of any per-request pacing. At
   ~1 req/s a bulk run hits hard `429`s after roughly 1,000 requests, followed
   by a **~40-minute lockout during which every attempt fails** — including
   single probe requests, so short-backoff retries just get burned. Plan
   accordingly:
   - Compute expected runtime as **pages ÷ quota rate** before starting and
     tell the user (e.g. ~5,000 pages at ~900 req/h is a multi-hour job).
   - For jobs over ~1,000 pages, default to **~3 s pacing** (~15 requests/min
     ≈ 900/h), which has run thousands of consecutive requests without
     throttling. Faster pacing (~1 req/s) only survives jobs that finish
     within the hourly budget — it does not scale beyond that.
   - If a hard-429 lockout hits anyway, **pause completely** with long sleeps
     (10+ minutes) until a single probe request succeeds again; do not rely on
     short exponential backoff, and do not let per-page retry counters burn
     down during the lockout.
   - **Always finish with a sweep** that compares the results store against
     the page list and re-fetches any pages missing from it (lockout
     casualties, exhausted retries).
   - **The authenticated session itself can expire on long runs** — observed
     at roughly 25 minutes on a continuous paced loop, and not under
     intermittent load; check it on the run before relying on the figure —
     independently of the quota. Once it goes, calls
     return `401` and **re-fetching the CSRF token does not recover it**; the
     run must be re-authenticated in the browser or truncated. Factor this into
     the plan: a job whose computed runtime exceeds ~25 minutes needs either a
     re-auth checkpoint or a decision to split it into sessions, and it is a
     further reason to check step 3 before committing to a multi-hour loop.
6. **Paced background loop over the smaller set.** Compare the site-wide
   query count with the page count, then loop over whichever is fewer:
   pages with `searchqueries/stats` and `Page=<url>`, or queries with
   `pages/stats` and `Query=<query>`. The query direction covers only
   site-wide queries. Launch it as a detached async IIFE that updates a
   `window.__progress` object; **do not** try to run hundreds of fetches inside
   one tool call (it will time out).
   - **Make the loop externally controllable from the start:** read the delay
     from `window.__delayMs` on *each* iteration and check a `window.__paused`
     flag (busy-wait in short sleeps while it is true). Later tool calls can
     then retune pacing or pause/resume the running loop without restarting
     it. Do not hardcode the delay in the closure.
   - On `429`/`503`, back off and raise `window.__delayMs`; on sustained 429s
     apply the lockout handling from step 5. Retry each page up to 3× (outside
     lockout windows).
   - Recovery note: if a loop with a *hardcoded* delay is already in flight,
     it can be remote-controlled by monkey-patching `window.setTimeout` to
     remap its known sleep durations (e.g. 900 ms → 60 s to pause, → 3 s to
     slow down). This works but is fragile and duration-range-based — it is a
     recovery technique for running loops, not a design pattern.
7. **Poll** progress with a promise-based wait so wall-clock actually passes in
   one call:
   `new Promise(r=>setTimeout(()=>r(JSON.stringify(window.__progress)),35000))`
   (top-level `await` is NOT supported by the JS tool; use Promises / detached
   async + polling).
   - **35 seconds is a ceiling, not an illustration.** The browser JS tool has
     its own execution timeout at roughly **45 seconds**, and the wait is
     bounded by the tool, not by the job being waited on. Get longer waits by
     polling repeatedly — **never** by lengthening a single poll. A 75-second
     poll does not fail cleanly: it returns an **empty object**, and the *next*
     poll then reports a progress figure far below reality. The corruption
     outlives the call that caused it.
   - **Diagnostic:** an empty or malformed poll result means the poll exceeded
     the timeout. **Discard the reading immediately after it** rather than
     interpreting it — take a fresh, correctly-sized poll instead. Two readings
     that span a failed call can show four items in ninety seconds on a job
     that was running at full speed throughout, which looks exactly like the
     stall you are watching for and invites the wrong intervention.
   - **Corollary for stall detection:** a stall must be confirmed by **two
     consecutive well-formed polls**. A single low delta following a failed
     poll is an artefact, not a stall.
   - **Long runs:** for extractions expected to exceed ~30 minutes, delegate
     the polling to a subagent instead of burning the main session's context
     on dozens of round-trips. Give the subagent (a) the exact poll call to
     run verbatim — **with the wait value fixed and a standing instruction not
     to raise it** — (b) quantified stop conditions: the finished flag, a
     retry-spike threshold, a stall timeout (e.g. no progress for 20 min)
     confirmed by two consecutive well-formed polls, and persistent browser
     errors — and (c) an explicit rule to do nothing else,
     returning the final progress object when a stop condition fires. A
     tightly-briefed monitor can babysit a multi-hour loop in ~100 tool calls
     while the parent context stays lean.

## Variant: server-paginated reports behind an opaque POST body

This section lives in `references/paginated-post-variant.md`. **Load it when the report is server-paginated behind an opaque POST body rather than the per-page API above.**

## Getting data out of the browser (content-filter workarounds)

This section lives in `references/browser-data-out.md`. **Load it before moving extracted results out of the browser, and whenever a content filter blocks or truncates a returned payload.**

## Building the spreadsheet

Use a spreadsheet-generation capability (e.g. the `xlsx` skill). Recommended
workbook:
- **Summary** — report, site, filter, date range, extracted date, and
  COUNTIF/SUM formulas over the other sheets (pages total, pages with/without
  grounding data, total query rows, total citations). Recalculate the formulas
  and require zero formula errors.
- **Grounding Queries** — one row per query for pages that have data:
  Page URL, Page Citations, Grounding Query, Intent, Query Citations, Topic.
- **All Pages** — every filtered page: URL, Citations, # grounding queries,
  Has Grounding Data (Yes/No) so the empty-but-cited pages are visible.

For a **site-wide** extraction there is no page dimension, so drop the All Pages
sheet and the Page URL / Page Citations columns, and label the query sheet
explicitly as site-wide, not the full set, and without page attribution —
otherwise a reader will assume it is exhaustive. Put the site's
`TotalCitations` from `citationstats` for the same range on the Summary
sheet, so an analysis can state what share of citations the list covers.

Save to the appropriate workspace folder, then present it to the user (e.g. via
a `computer://` link).

## Pitfalls

- No CSRF header → 400. Token is per session; re-fetch if calls start 401-ing
  early on. But `401`s appearing well into a long run mean the **session** has
  expired (observed at ~25 min on a continuous run), and re-fetching the token
  will *not* fix it — re-authenticate in the browser or truncate the run.
- Running the per-page loop for a task that never needed page attribution →
  hundreds of requests, hours of pacing, and a quota-lockout risk for a result
  the site-wide list (`Page: ""`, or its CSV export) would have delivered in
  seconds. Ask the attribution question first.
- Parsing the CSV export with a standard CSV parser → rows merged at
  unescaped inner quotes. Split each line on `","` and check the row count.
- Comparing or summing Citation Share across properties → a number with no
  common denominator. Rank within one property only.
- Lengthening a single poll past the browser tool's ~45 s ceiling → an empty
  result *and* a stale next reading that impersonates a stall. Poll repeatedly
  at ~35 s instead, and discard any reading that directly follows a failed call.
- `PageSize` above 500 → `400 InvalidRequestParameters`. Paginate with
  `PageNum` instead.
- Extrapolating pacing from a small run → hourly-quota lockout mid-job. A rate
  that survives a few hundred requests can still exhaust the ~1,000 req/h
  budget on a large job; budget requests per hour, not per second.
- Running the whole loop in one tool call → timeout. Detach + poll instead.
- Trusting `javascript_tool` to return URL data → blocked. Use the page-text
  DOM channel.
- Treating per-page 404s as failures → they mean "no sampled grounding data",
  which is expected for lower-citation pages.
- Echoing the CSRF token, cookies, or full URLs in a tool result → blocked
  and/or leaks a security token. Keep them in `window` vars.
- Letting always-empty columns (whichever of `Topics`/`TopicClusters` is empty
  in this dataset) collapse during `get_page_text`
  exfiltration → silent column shift and an all-empty read of a real column.
  Placeholder empties, then validate field counts and per-column population
  before analysis.

## Pre-flight verification

Before declaring the export done, re-read the rules in this skill and confirm:

1. The CSRF token, cookies, and full URLs were never echoed in a tool result —
   they stayed in `window` vars only.
2. Every empty/null field got an explicit placeholder before joining, so no
   interior column shifted during the whitespace-collapsing read-back.
3. The post-parse field-count assertion passed for every row (each row's field
   count equals the expected header count). For the CSV export, the parsed
   row count also equals `TotalNumOfRecords`.
4. Each named column was spot-checked for population — no column that should
   carry data (e.g. the populated one of `Topics`/`TopicClusters`) is silently
   all-empty due to a shift.
5. If the per-page loop was run: the final sweep ran, and every page in the
   page list is present in the results store either with data or with an
   explicit "no data" (404) record — no pages silently missing from throttling
   or exhausted retries. If the loop was run at all, confirm the attribution
   question was actually asked and answered "yes" — the loop is not the default.
   For a site-wide extraction, record the returned query count against
   `TotalNumOfRecords` instead, and state in the deliverable that the list
   is not the full set and carries no page attribution.
6. No client-identifying data leaked into any deliverable or shared artefact
   beyond what the user explicitly asked to export.

If any check fails, fix it before delivering.

## Feedback on this skill

If the user has methodology feedback, offer to draft a report for
https://github.com/rebelytics/bing-wmt-ai-grounding-export; before drafting, search that
repository's open and closed issues and pull requests for the same problem,
and read its CONTRIBUTING file for how the maintainer takes reports. If the
problem is the agent not following the skill's rules, acknowledge and
correct it instead.

Bing Webmaster Tools — AI Performance / Grounding Query Export: Eoghan Henn and contributors | CC BY 4.0
