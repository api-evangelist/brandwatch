---
name: Authenticate to Brandwatch and discover the account
description: Mint a Brandwatch Consumer Research access token, then find out which client, projects and queries the token can actually see, before spending any of a very small rate-limit budget on data calls.
api: openapi/brandwatch-consumer-research-openapi.yml
operations:
  - retrieving-the-current-client
  - retrieving-the-current-user
  - the-me-call-1
  - retrieving-project-summary
  - retrieving-queries-1
generated: '2026-08-13'
method: generated
source: >-
  openapi/brandwatch-consumer-research-openapi.yml,
  openapi/brandwatch-consumer-research-authentication-openapi.yml,
  https://developers.brandwatch.com/docs/authenticate
---

# Authenticate to Brandwatch and discover the account

Base URL: `https://api.brandwatch.com`

## Before you start: the budget is 30 calls per 10 minutes

Brandwatch rate-limits at the **Client** level, not per key and not per user.
Every integration and every human on the account shares **30 requests per
rolling 10 minutes**. This whole discovery flow costs 3 calls. Budget
accordingly, and cache everything you learn here — Brandwatch's own guidance is
to cache projects, queries, tags, categories and lists locally rather than
refetch them.

Check the headers on every response:

```
x-rate-limit: 30/10m
x-rate-limit-used: 5
```

There is no `Retry-After` and no reset timestamp. If you get a `429`, you know
you are over but not when you are clear; wait out a full 10 minutes.

## 1. Get a token

```
POST https://api.brandwatch.com/oauth/token
  ?username=<user@example.com>
  &grant_type=api-password
  &client_id=brandwatch-api-client
Body (form): password=<password>
```

Response:

```json
{
  "access_token": "REDACTED_TOKEN",
  "token_type": "bearer",
  "expires_in": 31535999,
  "scope": "read trust write"
}
```

Notes that matter:

- `grant_type=api-password` is a Brandwatch-specific grant. This is not
  client-credentials, despite what the published OpenAPI declares — the spec's
  `tokenUrl` is an unfilled `https://example.com/oauth2/token` placeholder.
- The token is good for about **one year**. Store it as a long-lived secret.
  There is no refresh endpoint and no revocation endpoint.
- You cannot request scopes. The token comes back with whatever the underlying
  user already has.
- If the account uses Consumer Research organization switching, add
  `&platform_client_id=<id>`. You have to ask Brandwatch support for that id;
  there is no endpoint that lists it.
- Only **Regular** and **Admin** Consumer Research users can call this API at
  all. Anything less gets a `403`.

## 2. Send the token

Preferred:

```
Authorization: bearer <ACCESS_TOKEN>
```

Brandwatch also documents `?access_token=<ACCESS_TOKEN>` in the query string.
**Do not use it.** These tokens live for a year, and query strings end up in
proxy logs, browser history and `Referer` headers.

## 3. Find out who you are — `the-me-call-1`

```
GET /me
```

One call returns combined User and Client information. Prefer it over calling
`/user` (`retrieving-the-current-user`) and `/client`
(`retrieving-the-current-client`) separately — that is 1 request instead of 2,
and at 30 per 10 minutes that matters.

## 4. List projects — `retrieving-project-summary`

```
GET /projects/summary
```

```json
{
  "resultsTotal": 2,
  "resultsPage": -1,
  "resultsPageSize": -1,
  "results": [
    { "id": 398748937, "name": "Telecoms research", "billableClientId": 127732, "timezone": "Africa/Abidjan" }
  ]
}
```

`resultsPage: -1` and `resultsPageSize: -1` are the sentinel for "this endpoint
is not paginated, you have everything". Do not treat `-1` as a page number.

Hold on to `id` and `timezone` — every downstream path is
`/projects/{projectId}/...`, and the project timezone is what date filters are
interpreted against.

## 5. List queries in a project — `retrieving-queries-1`

```
GET /projects/{projectId}/queries/summary
```

A Query is a saved boolean search; it is the thing that defines a stream of
Mentions. You need its `id` for every data call.

Use `retrieving-detailed-summary-all-queries`
(`GET /projects/{projectId}/queries`) only when you actually need the boolean,
languages, content sources and sampling settings — it returns roughly 27 fields
per query instead of 2.

## Errors

Every error is the same two-field envelope, not RFC 9457 problem details:

```json
{ "error": "unauthorized", "error_description": "Invalid authentication credentials found on request" }
```

| Status | Means |
|---|---|
| 401 | Token missing, malformed or revoked. Mint a new one — expiry is a year out, so this is rarely staleness. |
| 403 | The user is not Regular/Admin, or the product (e.g. Data Upload) is not enabled on the account. A CSM has to fix this; retrying will not. |
| 404 | The project or query id does not exist, or is not visible to this Client. |
| 429 | Over 30 calls in 10 minutes. Not declared in the spec, but real. |

## Stop conditions

- `403` — do not retry. It is an entitlement problem, not a transient one.
- `429` — stop entirely for 10 minutes. There is no header telling you when you
  are clear.
