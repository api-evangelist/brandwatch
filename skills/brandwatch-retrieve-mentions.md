---
name: Retrieve and page Brandwatch mentions
description: Pull mentions for a Brandwatch query over a date range and page through a historical set without exhausting the 30-calls-per-10-minutes account budget, accounting for the fact that the returned field set is not constant.
api: openapi/brandwatch-consumer-research-openapi.yml
operations:
  - total-mentions-1
  - retrieving-mentions-1
generated: '2026-08-13'
method: generated
source: >-
  openapi/brandwatch-consumer-research-openapi.yml,
  https://developers.brandwatch.com/docs/retrieving-mentions,
  https://developers.brandwatch.com/docs/tutorial-paging-through-historical-mentions,
  https://developers.brandwatch.com/docs/data-restrictions
---

# Retrieve and page Brandwatch mentions

Base URL: `https://api.brandwatch.com`. Requires a bearer token — see
`brandwatch-authenticate-and-discover.md`.

## 1. Count before you page — `total-mentions-1`

```
GET /projects/{projectId}/data/mentions/count
  ?queryId={queryId}
  &startDate=2016-05-01
  &endDate=2016-05-02
```

Always do this first. One call tells you how many pages the real fetch will
cost. At 30 requests per rolling 10 minutes, shared across the entire
Brandwatch Client, a 50,000-mention backfill at the default page size is not
something to discover halfway through.

## 2. Fetch a page — `retrieving-mentions-1`

```
GET /projects/{projectId}/data/mentions
  ?queryId={queryId}
  &startDate=2016-05-01
  &endDate=2016-05-02
  &pageSize=1
  &page=0
```

Response envelope:

```json
{
  "resultsTotal": 2,
  "resultsPage": 0,
  "resultsPageSize": 1,
  "results": [ { "resourceId": 0, "queryId": 0, "date": "...", "sentiment": "..." } ]
}
```

`page` is zero-based. Stop when `(page + 1) * pageSize >= resultsTotal`.

Useful narrowing parameters, all of which reduce your page count and are
therefore worth more here than on a normal API: `sentiment`, `pageType`, `tag`,
`xtag` (exclude tag), `category`, `xcategory` (exclude category),
`excludeTags`, `excludeCategories`, `orderBy`, `orderDirection`.

## 3. Page a historical set without burning the budget

- Serialize. Brandwatch explicitly says parallel requests "may be throttled and
  take longer to return than a sequence of linearly executed requests". Fan-out
  is actively counterproductive here.
- Use the largest page size the endpoint accepts so you spend fewer requests
  for the same rows.
- Slice by date, not by offset, for long backfills. Walk day by day with
  `startDate`/`endDate` and restart cleanly at a day boundary rather than
  resuming a 4,000-deep offset.
- Cache tags, categories and lists locally. Do not refetch reference data
  alongside every mentions call — the provider's own best-practices page says
  so, and each avoided call is 1/30th of your window.
- Track `x-rate-limit-used` on every response and stop yourself before the API
  does. There is no `Retry-After` to guide a resume.

## 4. Expect a variable field set — this is the important one

A Mention can carry ~40 documented fields (`resourceId`, `author`, `date`,
`sentiment`, `impressions`, `reachEstimate`, `country`, `gender`, `domain`,
`categories`, `tags`, `guid`, …), but **the field set is not constant and is not
expressed in the contract**:

- **X (Twitter)**: full text and most metadata are stripped for compliance. You
  get an allow-listed subset. To get the body, take the X post id out of `guid`
  and fetch it from the X API yourself.
- **Reddit**: restrictions took effect January 2026.
- **Data Packs**: what your account bought determines what comes back. Two
  customers calling this identical operation get different fields, with no
  difference anywhere in the OpenAPI.

So: never assume a field is present because the schema lists it. Treat every
field as optional, and expect the UI and the API to disagree — Brandwatch
documents that discrepancy as expected behavior.

Field reference:
<https://developers.brandwatch.com/docs/mention-metadata-field-definitions>.
Restrictions: <https://developers.brandwatch.com/docs/data-restrictions>.

## 5. Polling for new mentions

There are no webhooks and no AsyncAPI. "Real-time" on this API means you poll.
Brandwatch's guidance is roughly every 30 seconds for a live mention stream —
but note that a 30-second poll is 20 calls per 10 minutes, two thirds of the
entire account budget, before any other integration on the account does
anything.

Poll with `startDate` set to the last mention `date` you saw. Deduplicate on
`resourceId`; overlapping windows will re-deliver rows.

## Errors and stop conditions

| Status | Action |
|---|---|
| 401 | Mint a new token. |
| 403 | Entitlement, not transient. Stop. |
| 404 | Bad `projectId` or `queryId`. Stop; re-run discovery. |
| 429 | Stop for a full 10 minutes. No header tells you when you are clear. |

Envelope: `{"error": "...", "error_description": "..."}`. Not RFC 9457.
