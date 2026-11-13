# The Topic Portfolio tab: topics and opportunity queries

Reference file for the `bing-wmt-ai-grounding-export` skill (CC BY 4.0 — Eoghan Henn and contributors). Load when the user asks for topics, opportunity queries or queries the site is not cited for, and when a completeness union needs the per-topic query lists.

## The Topic Portfolio tab

The AI Performance report has a second tab, "Topic Portfolio", which Bing
marks as beta and experimental. It is the only view of queries the site is
**not** cited for. The endpoints, body shapes and caps below were observed
on one account's properties at last verification. A beta surface changes
without notice, so re-check one request in the network panel before a bulk
run.

All calls are same-origin `POST` with the `X-CSRF-Token` header, under the
same API base as the core's endpoints; confirm the full path in the network
panel.

| Endpoint | Body | Returns |
|---|---|---|
| `topics/status` | `{SiteUrl}` | `HasData` — check it first |
| `topics/availablemonth` | `{SiteUrl}` | `TopicsDataEndDate`, the last complete month |
| `topics/overallportfolio` | `{SiteUrl, SessionId, DateRange, TopicsMonths, Pagination, Query, Page}` | up to 50 rows of `Topic, VolumeRange, QueriesRange, SitesRange` |
| `topics/topopportunityqueries` | the same, plus `SortParameters` | up to 50 rows of `Query, Topic, CitationRange` |
| `topics/topiclevel` | `{SiteUrl, SessionId, Source: "DirectNavigation", DateRange, TopicsMonths, Topic, GroundingQueriesSortParameters, OpportunityQueriesSortParameters, Query, Page}` | `TotalCitations, TotalTopicSizeRange, TotalQueriesRange`, `GroundingQueries` (10 per topic), `OpportunityQueries` (up to 50 per topic) |

Every view also has an `/export` endpoint.

- **The data is monthly.** `DateRange` is one calendar month ending no later
  than `TopicsDataEndDate`, and `TopicsMonths` (1, 2 or 3) sets the window.
  `SessionId` accepted any string.
- **Sizes are bucketed ranges, not counts** (e.g. `"1K-5K"`). The UI defines
  Topic Size as the approximate number of distinct grounding queries in the
  topic, and Sites Cited as the approximate number of sites cited on them.
  Report the range as given; never average or sum ranges.
- **An opportunity query is one where other sites are cited and this one is
  not**, by the UI's own definition. `Topic` can hold two comma-separated
  topics; split it before grouping.
- **Neither view names the other sites.** To learn who is cited instead, run
  the queries through a citation tracker.

## Rules for using the lists

- **The 50-row cap is per list, not per site.** The top-level opportunity
  list stops at 50. Looping `topics/topiclevel` over every portfolio topic
  returns far more, and the per-topic union contained every top-level entry
  where tested. For a full opportunity set, loop the topics (one request
  each, paced as in the core) and deduplicate on the query string.
- **The per-topic `GroundingQueries` belong in a completeness union.** They
  hold queries missing from the site-wide list (core, "Grounding queries").
- **The lists are not a neutral market view.** Bing states that topics and
  opportunity queries are personalised to the site's context. Site owners
  can also add suppression rules that hide topics or opportunity queries.
  Ask whether suppression rules exist, and state both caveats in the
  deliverable.
- **The lists are topic-scoped, not relevance-filtered.** A topic can sweep
  in lookups far outside the business. Invented example: a regional
  orchestra's ticketing topic carrying airline seat-map queries. Any use of
  the lists (outreach, content, prompt gaps) needs a relevance gate against
  what the brand actually offers before anything is counted.
- **Spreadsheet:** add an "Opportunity Queries" sheet with Query, Topic,
  CitationRange, the source list (top-level or per-topic) and the month
  window, and record on the Summary sheet whether the per-topic loop ran.
