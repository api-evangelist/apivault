---
name: Browse the ApiVault catalog
description: >-
  Read the whole ApiVault public-API directory with no credentials — list every
  catalogued API, list the 51 categories, filter by category, and read one
  record. Every operation here is anonymous.
api: openapi/apivault-api-openapi.yml
base_url: https://api.apivault.dev
operations: [count_retrieve, categories_list, categories_trending_list, all_list, category_list, detail_retrieve]
auth: none
---

# Browse the ApiVault catalog

ApiVault is a directory of free and public APIs. The read half of its API needs
**no credential at all** — the spec declares `security: [{jwtAuth: []}, {}]` on
these operations, which means the token is optional, and every one of them was
confirmed answering `200` anonymously.

The published OpenAPI ships **no `servers[]` block**. Use
`https://api.apivault.dev` as the base — that is the host the schema is served
from and the host all of these operations answer on.

## Before you start

- Send `Accept: application/json`. A malformed `Accept` gets you a `406` with
  `{"detail": "Could not satisfy the request Accept header."}`.
- Send no `Authorization` header. You do not need one here, and a malformed one
  causes the `406` above.

## Steps

1. **Size the catalog.** `count_retrieve` — `GET /api/count`. Returns
   `{"api_count": <n>}`. It was 1,454 on 2026-09-04.

2. **Get the category vocabulary.** `categories_list` — `GET /api/categories`.
   Returns `[{"id": 39, "name": "Science & Math"}, …]`, 51 of them.
   **Keep the `id` values.** You will need them if you ever submit an API: the
   `category` field is the category *name* when you read a record and the
   category *id* when you write one.

3. **Pick a starting point.**
   - Everything at once: `all_list` — `GET /api/all`. This returns the
     **entire** catalog in one array. There is no pagination — no `limit`, no
     `offset`, no cursor, no `Link` header — so budget for the whole payload or
     do not call it.
   - One category: `category_list` — `GET /api/category/{category_name}`, where
     `category_name` is the **name** from step 2, URL-encoded.
     **Validate the name against step 2 first.** An unknown category name does
     not return `404`; it returns `500` with an HTML Django error page, which
     will break a client that assumes JSON.
   - What is active: `categories_trending_list` —
     `GET /api/categories/trending`. Returns `{name, api_count}` per category.

4. **Read one record.** `detail_retrieve` — `GET /api/detail/{id}` with the
   integer `id` from any list above. A missing id returns `404`
   `{"detail": "Not found."}`.

## What a record contains

`id`, `name`, `auth` (`apiKey`, `OAuth`, or empty for none), `category` (the
name), `cors`, `description`, `https`, `url`, `likes_count`, `liked_by_user`.

There are **no timestamps**. Nothing tells you when an entry was added or last
checked, so treat the `url` as a lead to verify, not a fact.

## Rules

- **No rate-limit signal exists.** No `X-RateLimit-*`, no `RateLimit-*`, no
  `Retry-After`, and no documented limit. Self-throttle; you have nothing to
  back off against. See `rate-limits/apivault-rate-limits.yml`.
- **Errors are not RFC 9457.** They are `{"detail": "<english string>"}` with
  `Content-Type: application/json`. Match on the HTTP status, not the prose.
  See `errors/apivault-problem-types.yml`.
- **`GET /api/search` has an undocumented parameter.** See the searching skill.
