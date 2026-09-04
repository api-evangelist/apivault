---
name: Submit an API and manage likes on ApiVault
description: >-
  The authenticated half of ApiVault — submit a new API to the directory, like
  and unlike a record, send feedback, and read your own submissions. Requires a
  SimpleJWT that can only be obtained through an interactive Google sign-in.
api: openapi/apivault-api-openapi.yml
base_url: https://api.apivault.dev
operations: [auth_google_create, auth_token_verify_create, auth_token_refresh_create, auth_user_retrieve, categories_list, create_create, my_api_retrieve, pending_my_api_retrieve, interaction_like_create, interaction_like_destroy, interaction_feedback_create]
auth: bearer-jwt
---

# Submit an API and manage likes on ApiVault

## Read this before you plan anything

**An unattended agent cannot get a credential here.** ApiVault issues no API
keys and supports no client-credentials grant. The only token path is:
a human signs in to apivault.dev with Google in a browser, and the resulting
Google token is exchanged for a SimpleJWT at `auth_google_create`. If you do
not already hold a valid JWT handed to you by a human, stop — every operation
below returns `401 {"detail": "Authentication credentials were not provided."}`.

## Authenticate

1. **Exchange.** `auth_google_create` — `POST /api/auth/google/` with
   `{"auth_token": "<google token>"}`. Returns the SimpleJWT access/refresh
   pair.
2. **Send it.** `Authorization: Bearer <access>` on every call below, plus
   `Accept: application/json`. A malformed `Authorization` value produces
   `406 {"detail": "Could not satisfy the request Accept header."}`, not a 401 —
   do not misread that as a rate limit or an outage.
3. **Check / refresh.** `auth_token_verify_create` —
   `POST /api/auth/token/verify/` with `{"token": …}`;
   `auth_token_refresh_create` — `POST /api/auth/token/refresh/` with
   `{"refresh": …}`.
4. **Confirm who you are.** `auth_user_retrieve` — `GET /api/auth/user/`.
   Returns `username`, `email`, `picture`.

There is **no revocation endpoint**. A leaked token cannot be withdrawn
through this API.

## Submit an API — this is one-way

1. **Get the category ids.** `categories_list` — `GET /api/categories`.
   This step is not optional. On submission, `category` is the **integer id**,
   even though the same field is the category **name** on every read. Reading a
   record and posting it back without translating this field will fail.
2. **Submit.** `create_create` — `POST /api/create` with the `APICreate` body:
   `name` (≤100), `auth` (`"apiKey"`, `"OAuth"` or `""`), `category` (integer
   id), `cors` (bool), `description`, `https` (bool), `url` (≤200). Returns
   `201`.

> **There is no undo.** No withdraw, cancel or delete operation exists. The
> submission enters a moderation queue and removing it means contacting the
> ApiVault team. **And there is no idempotency mechanism** — no
> `Idempotency-Key` header, no client request id, no dedupe. A retried POST
> creates a *second* submission. Never retry `create_create` on a timeout;
> check `pending_my_api_retrieve` first.

3. **See what is pending.** `pending_my_api_retrieve` —
   `GET /api/pending/my_api`. Awaiting moderation.
4. **See what landed.** `my_api_retrieve` — `GET /api/my_api`.

## Like and unlike — the one reversible action

- **Like:** `interaction_like_create` —
  `POST /api/interaction/like/{api_id}`. Returns `200`, empty body.
- **Unlike:** `interaction_like_destroy` —
  `DELETE /api/interaction/like/{api_id}`. Returns `204`, empty body.

This is the only write on ApiVault with a first-class reversal. The provider
states **no time window** on it, so do not promise one to a user.

Neither operation returns the resulting state. Read it back from the
`likes_count` and `liked_by_user` fields on the record via `detail_retrieve`.

## Send feedback

`interaction_feedback_create` — `POST /api/interaction/feedback` with
`{"name": …, "email": …, "message": …}` (`message` required, ≤150 chars;
`name` ≤30; `email` ≤254). Returns `201`. **No retract operation** — treat it
as final, and confirm the text with your user before sending.

## Rules

- Retries are unsafe on every POST here. See
  `conventions/apivault-conventions.yml` (`idempotency.coverage: none`).
- Errors are `{"detail": "<english string>"}`, not RFC 9457. Match on status.
- No rate-limit headers are returned. Self-throttle.
