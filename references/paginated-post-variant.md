# Variant: server-paginated reports behind an opaque POST body

Reference file for the `bing-wmt-ai-grounding-export` skill (CC BY 4.0 — Eoghan Henn and contributors). Load when the report is server-paginated behind an opaque POST body rather than the per-page API above.

## Variant: server-paginated reports behind an opaque POST body

Other Bing Webmaster Tools reports — the IndexNow submitted-URL list, for
instance — are **server-paginated at ~25 rows** via a POST with a JSON body and
no query string (e.g. `POST /webmasters/api/indexnow/list` with
`{SiteUrl, Pagination:{PageNum, PageSize}, SortParameters, SourceFilter}`).
The AI Performance recipe above does not transfer directly, because there is no
URL to guess and the browser content filter **blocks any tool result containing
the request body** — it reads as query-string data. So the body cannot be
inspected and replayed the usual way.

Do not keep trying to reconstruct the request. **Let the application issue it**
and intercept the result:

1. **Probe the shape without printing it.** Parse the intercepted body and
   return only `Object.keys` — or key-name + type pairs. Key names pass the
   filter where the body itself does not.
2. **Accumulate responses; do not replay requests.** Patch `window.fetch` (or
   `XMLHttpRequest`, depending on what the report uses) to push each matching
   response into a `window` array. This needs no auth headers and no
   anti-forgery token, because the app supplies both.
3. **Drive the app's own paging control** in a loop rather than manipulating
   `Pagination`. Mutating `PageSize` upward is the obvious move and it does not
   work — the server ignores it and still returns a page of 25. Clicking
   through at ~1 s intervals does work.
4. **Chunk the click loop to ~20 iterations per tool call.** A 20-click loop at
   1 s intervals fits inside the browser tool's ~45 s execution ceiling; two
   chained loops in one call do not (see the polling ceiling above — the same
   limit, the same failure mode).
5. **If a call does time out, re-read the accumulator with a trivial separate
   call.** The loop keeps running in the page and the collected data survives —
   a call that reports failure has usually still banked nearly all its rows.
   Treat the first reading after a failed call as suspect and re-take it before
   acting on it.
6. **Classify in-page and return counts.** A result containing URLs is blocked,
   so run the aggregation inside the page and let only the aggregate numbers
   cross the tool boundary.

The general principle: **a channel that restricts what may come out is a reason
to move the computation inward, not a reason to abandon the extraction.**
