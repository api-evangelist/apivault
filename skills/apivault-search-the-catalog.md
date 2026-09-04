---
name: Search ApiVault for an API
description: >-
  Find a public API in the ApiVault directory by free-text query. Uses the
  undocumented `query` parameter that the provider's own client sends but the
  published OpenAPI omits.
api: openapi/apivault-api-openapi.yml
base_url: https://api.apivault.dev
operations: [search_list, categories_list, detail_retrieve]
auth: none
---

# Search ApiVault for an API

## The parameter the spec does not mention

`search_list` — `GET /api/search` — is published with **zero parameters**. If
you call it as documented you get the entire catalog back, unfiltered, which
looks like a working search that ignores your input.

The real call is:

```
GET https://api.apivault.dev/api/search?query=<term>
Accept: application/json
```

Evidence: the provider's own Nuxt client sends it —
`frontend/services/ApivaultServices.ts`, `search(query, authToken)` builds
`${baseUrl}/search?query=${query}` — in
`https://github.com/exa-studio/ApiVault`. Confirmed live on 2026-09-04.
Treat this as observed behaviour that could change without notice, because it
is outside the published contract.

## Steps

1. **Search.** `search_list` — `GET /api/search?query=<term>`, URL-encoded.
   Returns an array of catalog records, unpaginated.
2. **If the result set is too broad, narrow by category instead.** Call
   `categories_list` (`GET /api/categories`) for the 51 valid names, then
   `category_list` (`GET /api/category/{category_name}`). Never pass a category
   name you did not read from `categories_list` — an unknown name returns a
   `500` HTML page, not a `404`.
3. **Read the winner.** `detail_retrieve` — `GET /api/detail/{id}`.

## Reading the result

Each record's `auth` field is the *catalogued* API's own auth style —
`apiKey`, `OAuth`, or empty meaning none. `cors` and `https` are booleans about
that API, not about ApiVault. `url` points at the third-party API's own page.

ApiVault records **no machine-readable contract** for the APIs it lists: there
is no OpenAPI link, no spec URL, no `apis.json`. Following `url` and reading the
target's own documentation is the only way onward.

## Rules

- No credential is needed and none should be sent.
- No pagination exists on any of these operations.
- No rate-limit headers are returned. Self-throttle.
