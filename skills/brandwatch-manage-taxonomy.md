---
name: Manage Brandwatch project taxonomy
description: Create and remove the tags, categories and author/location/site lists that classify Brandwatch mentions, and backfill a category across historical data — with the caveat that none of these writes are idempotent.
api: openapi/brandwatch-consumer-research-openapi.yml
operations:
  - retrieving-tags-1
  - creating-tags-1
  - deleting-tags-1
  - retrieving-categories-1
  - creating-categories-1
  - backfill-categories
  - check-category-backfill-progress
  - creating-author-lists-1
  - retrieving-author-lists-1
  - deleting-author-lists-1
  - creating-site-lists-1
  - retrieving-site-lists-1
  - creating-location-lists-1
  - retrieving-location-lists
  - retrieving-rules-1
  - retrieving-workflow-1
generated: '2026-08-13'
method: generated
source: >-
  openapi/brandwatch-consumer-research-openapi.yml,
  https://developers.brandwatch.com/docs/tags,
  https://developers.brandwatch.com/docs/categories,
  https://developers.brandwatch.com/docs/lists,
  https://developers.brandwatch.com/docs/rules
---

# Manage Brandwatch project taxonomy

Base URL: `https://api.brandwatch.com`. Requires a bearer token — see
`brandwatch-authenticate-and-discover.md`.

## Read this first: there is no idempotency

Brandwatch supports **no idempotency key** — no header, no parameter, nothing
in the docs. If a `POST` times out and you retry it, you create a second tag,
a second category, a second list. There is also no update operation for tags or
categories: the surface is create and delete only.

So, for every write in this skill:

1. **List first.** Fetch the existing set and match on `name`.
2. Only create if it is genuinely absent.
3. If a create times out with no response, **list again** to find out whether it
   landed. Do not blind-retry.

Ids are unprefixed integers, so nothing about an id tells you what it refers
to — track which entity type you got each id from.

## Tags — flat labels

```
GET    /projects/{projectId}/tags            # retrieving-tags-1
POST   /projects/{projectId}/tags            # creating-tags-1
DELETE /projects/{projectId}/tags/{tagId}    # deleting-tags-1
```

Read returns `{ id, name }` per tag. Filter server-side with `nameContains`
rather than pulling everything and filtering locally — you have 30 calls per 10
minutes.

Create returns `201`. Delete returns `204`.

## Categories — hierarchical, with a `children` array

```
GET  /projects/{projectId}/categories        # retrieving-categories-1
POST /projects/{projectId}/categories        # creating-categories-1
```

Fields: `{ id, name, multiple, children }`. Categories nest via `children`, so
a read is a tree, not a list — walk it recursively when matching by name.
`multiple` controls whether a mention may hold more than one child of that
category.

There is no category delete operation in the published spec.

### Backfilling a category over history

```
POST /projects/{projectId}/rulebackfill/rulecategories/{parentCategoryId}   # backfill-categories
GET  /projects/{projectId}/rulebackfill/rulecategories                      # check-category-backfill-progress
```

This is asynchronous and it is the most consequential write in the whole API —
it reclassifies historical mentions in bulk. Submit, then poll the progress
endpoint. Poll on a long interval: a tight polling loop here is the fastest way
to spend an account's entire rate-limit budget on a job that changes nothing by
being watched.

Because there is no idempotency key, **do not resubmit a backfill you are
unsure about.** Check progress first.

## Lists — reusable filter sets

Three parallel resources with the same shape:

```
POST   /projects/{projectId}/group/author            # creating-author-lists-1
GET    /projects/{projectId}/group/author/summary    # retrieving-author-lists-1
DELETE /projects/{projectId}/group/author/{authorListId}

POST   /project/{projectId}/group/location           # creating-location-lists-1  ← note: /project/, singular
GET    /projects/{projectId}/group/location/summary  # retrieving-location-lists
DELETE /projects/{projectId}/group/location/{locationListId}

POST   /projects/{projectId}/group/site              # creating-site-lists-1
GET    /projects/{projectId}/group/site/summary      # retrieving-site-lists-1
DELETE /projects/{projectId}/group/site/{siteListId}
```

**Watch the location create path.** It is `/project/{projectId}/group/location`
— singular `project` — while every other path in the API uses plural
`projects`. This is in the published contract, not a typo here. Hard-code it;
do not template the path from the others.

Author list reads return `{ id, name, shared, sharedProjectIds, authors,
userId, userName }`. `shared` + `sharedProjectIds` mean a list you delete may
be in use by another project.

## Rules and workflow — read-only from the API

```
GET    /projects/{projectId}/rules              # retrieving-rules-1
DELETE /projects/{projectId}/rules/{ruleId}     # deleting-rules-1
GET    /projects/{projectId}/workflow           # retrieving-workflow-1
```

Rules carry `{ id, projectId, name, filter, scope, enabled, ruleAction }` and
are what automatically apply sentiment, categories, tags and workflow states to
incoming mentions. You can list and delete them but **not create or update
them** through the API — rule authoring is UI-only. A rule's `filter.queryId`
tells you which query it acts on.

`retrieving-rules-1` declares only a `200` in the spec — no error responses at
all. Do not read that as "this cannot fail".

## Cache what you read

Brandwatch's own best-practices guidance: cache categories, tags and lists
locally rather than refetching them with every mentions call. On a 30-per-10-
minutes budget this is not an optimization, it is a requirement.

## Errors

| Status | Meaning |
|---|---|
| 201 | Created (tags, categories) |
| 204 | Deleted (tags, queries, query groups) |
| 400 | Invalid body or path — returned by the delete-rule, delete-site-list and both backfill operations |
| 403 | Not a Regular/Admin user |
| 404 | Project or entity id not visible to this Client |
| 429 | Rate limit — stop for 10 minutes |

Envelope: `{"error": "...", "error_description": "..."}`. Not RFC 9457.
