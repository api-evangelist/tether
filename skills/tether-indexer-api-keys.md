---
name: tether-indexer-api-keys
description: Issue, inventory, label, expire and revoke Tether WDK Indexer API keys over the API itself, including how to discover a key's own rate-limit budget.
api: openapi/tether-wdk-indexer-openapi-original.yml
base_url: https://wdk-api.tether.io
operations:
  - createApiKey
  - listApiKeys
  - deleteApiKey
generated: '2026-08-05'
method: generated
source: openapi/tether-wdk-indexer-openapi-original.yml
---

# Manage WDK Indexer API keys

The WDK Indexer exposes its own credential lifecycle as three operations. This is
unusual and useful: an operator can rotate credentials without a console. Deliberately
**not** exposed as an MCP tool — see `mcp/tether-tool-crosswalk.yml`.

Every operation here authenticates with an existing `X-API-KEY` header. Bootstrap the
first key through the registration form at <https://wdk-api.tether.io/register>.

## Create — `createApiKey`

`POST /api/v1/keys`

Request body:

| Field | Type | Required | Meaning |
|---|---|---|---|
| `label` | string | No | Free-text label for the key |
| `ttl` | number | No | Time-to-live in **milliseconds**; `0` means no expiry |

Response: `{ key, hashedKey, owner, ttl, label, createdAt, lastActive }`.

**`key` is the plaintext value and is returned once.** Persist it to your secret store
in the same code path that issues it. If it is lost, the only remedy is to delete the
key by its `hashedKey` and create a new one.

Set a real `ttl` for anything handed to an agent or a CI job. `ttl: 0` should be
reserved for a human-held break-glass credential.

## Inventory — `listApiKeys`

`GET /api/v1/keys`

Returns `{ keys: [ { hashedKey, owner, ttl, label, createdAt, lastActive, max, timeWindow } ] }`.

Note the two fields no other operation surfaces:

- `max` — the key's request budget
- `timeWindow` — the window that budget applies over

This is the **only** way to read your own rate limit at runtime: the API sends no
`RateLimit-*` or `X-RateLimit-*` response headers. Fetch this once at start-up and size
your client-side token bucket from it rather than hard-coding the numbers in
`rate-limits/tether-rate-limits.yml`, which are the documented defaults.

`lastActive` makes unused-credential cleanup straightforward — list, filter on a stale
`lastActive`, delete.

## Revoke — `deleteApiKey`

`DELETE /api/v1/keys/{hashedKey}`

Addresses the key by its **hash**, never by its value, so a revocation script never has
to hold plaintext credentials. Get `hashedKey` from `listApiKeys` or from the
`createApiKey` response.

Returns `404` when the key does not exist *or* when it is not yours — the API does not
distinguish the two, so do not treat a `404` as proof a key was never issued.

## Rotation sequence

1. `listApiKeys` — record the `hashedKey` of the credential being replaced.
2. `createApiKey` with the same `label` and an appropriate `ttl` — store the new `key`.
3. Deploy the new key and confirm a live call succeeds (`listChains` is unauthenticated,
   so use `getTokenBalances` to actually exercise the credential).
4. `deleteApiKey` on the old `hashedKey`.

Do not reverse steps 3 and 4. There is no grace period and no overlap window built into
the API.

## Error handling

| Status | Meaning |
|---|---|
| 400 | Invalid body on `createApiKey` |
| 401 | The key you authenticated with has **expired** |
| 403 | The key you authenticated with is **missing or invalid** |
| 404 | Target key not found, or not owned by you |
| 429 | Rate limited — no `Retry-After` is sent |
| 500 | Indexer service failure |

The 401/403 split is inverted relative to the usual convention. A key that has aged past
its `ttl` returns 401; a key that was never valid returns 403. Rotation logic should
trigger on **401**, not 403.
