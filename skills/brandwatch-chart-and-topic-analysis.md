---
name: Get aggregated Brandwatch analytics instead of raw mentions
description: Use Brandwatch's server-side aggregate, topic and chart endpoints to answer volume, sentiment and demographic questions in one call rather than paging thousands of mentions and counting them yourself.
api: openapi/brandwatch-consumer-research-openapi.yml
operations:
  - total-mentions-1
  - basic-charts-1
  - topics-1
  - twitter-insights-1
generated: '2026-08-13'
method: generated
source: >-
  openapi/brandwatch-consumer-research-openapi.yml,
  https://developers.brandwatch.com/docs/basic-charts,
  https://developers.brandwatch.com/docs/chart-dimensions-and-aggregates,
  https://developers.brandwatch.com/docs/data-topics
---

# Get aggregated Brandwatch analytics instead of raw mentions

Base URL: `https://api.brandwatch.com`. Requires a bearer token — see
`brandwatch-authenticate-and-discover.md`.

**The rule for this API:** if the question is "how many" or "how did it break
down", aggregate server-side. Do not page mentions and count them in your own
code. At 30 requests per rolling 10 minutes per Client, a chart call that
replaces 200 pages of mentions is the difference between a working integration
and a throttled one.

## Volume in one call — `total-mentions-1`

```
GET /projects/{projectId}/data/mentions/count
  ?queryId={queryId}&startDate=...&endDate=...
```

Cheapest question in the API. Use it for any "how many mentions" answer, and
as the pre-flight before any paging job.

## Breakdowns — `basic-charts-1`

```
GET /data/{aggregate}/{dimension1}/{dimension2}
  ?queryId={queryId}&startDate=...&endDate=...
```

The aggregate and the two dimensions are **path** segments, not query
parameters, which is unusual and easy to get wrong. The full matrix of valid
aggregate/dimension combinations is published at
<https://developers.brandwatch.com/docs/chart-dimensions-and-aggregates> —
read it rather than guessing, because an invalid combination is not
discoverable from the OpenAPI (the parameters are typed as free strings).

Typical shapes: volume over days, sentiment by day, volume by country, volume
by page type, sentiment by gender.

Optional narrowing on the same call: `sentiment`, `pageType`, `tag`, `xtag`,
`category`, `xcategory`, `limit`.

Newer multi-aggregate chart calls (added 2025-07-25) let one request return
several aggregates at once — worth using when your dashboard needs three
numbers, because it turns three requests into one.

## Topics — `topics-1`

```
GET /projects/{projectId}/data/volume/topics/queries
  ?queryId={queryId}&startDate=...&endDate=...
```

Server-side clustering of what the conversation is *about*. There is no local
substitute — do not try to reconstruct this by pulling mentions and clustering
them yourself. The newer Topics endpoint arrived 2025-07-25; the older
`/docs/topics` page describes the previous one.

## X (Twitter) insights — `twitter-insights-1`

```
GET /data/hashtags?queryId={queryId}&startDate=...&endDate=...
```

Hashtag insights over the X portion of the query.

Related published surfaces documented in the guides but absent from the
OpenAPI: Top Sites, Top Authors, Top X Authors, Top Shared Sites, Top Shared
URLs. If you need those, work from the docs pages, not the spec — and note
Brandwatch's support boundary: "we only officially support the API endpoints
featured here, in our documentation. Any other endpoints are subject to change
without notice."

## Date handling

`startDate` and `endDate` are ISO 8601 and are interpreted against the
**project's** timezone, which you get from `retrieving-project-summary`
(`timezone`, e.g. `Africa/Abidjan`). Do not assume UTC. A day-boundary chart
computed in the wrong timezone is wrong in a way that looks plausible.

## What aggregation will not fix

Aggregates are computed over the same restricted mention pool as raw retrieval.
If your account's Data Packs limit X or Reddit metadata, a demographic or
source breakdown inherits that restriction silently — the numbers come back
without any indication that a source was thinned. See
<https://developers.brandwatch.com/docs/data-restrictions>.

## Errors

Same envelope as everywhere else: `{"error": "...", "error_description": "..."}`.
`400` on an invalid aggregate/dimension combination, `403` on entitlement,
`429` on the rate limit — with no `Retry-After`.
