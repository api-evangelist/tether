---
name: tether-indexer-balances-and-history
description: Read USD₮/XAU₮/USA₮/BTC balances and token-transfer history for one or many addresses from the Tether WDK Indexer API, across Ethereum, Arbitrum, Avalanche, Polygon, Sepolia, Tron, TON, Bitcoin and Spark.
api: openapi/tether-wdk-indexer-openapi-original.yml
base_url: https://wdk-api.tether.io
operations:
  - listChains
  - getTokenBalances
  - getTokenTransfers
  - batchTokenBalances
  - batchTokenTransfers
  - getHealth
generated: '2026-08-05'
method: generated
source: openapi/tether-wdk-indexer-openapi-original.yml + https://docs.wdk.tether.io/tools/indexer-api
---

# Read balances and transfer history from the Tether WDK Indexer

Operating instructions for the only hosted HTTP API Tether publishes. Every
`operationId` below is assigned in `overlays/tether-wdk-indexer-overlay.yaml`; the
method + path is given alongside so the mapping holds against the raw spec, which
declares no operationIds of its own.

## 1. Get a key

Register free at <https://wdk-api.tether.io/register>. Send it on every authenticated
call as the `X-API-KEY` **request header**. `getHealth` and `listChains` are the only
operations that do not require it.

Two credential quirks to code against:

- **401 means expired, 403 means missing or invalid.** This is the inverse of the more
  common convention. Do not treat 401 as "no credential sent".
- Keys are self-service over the API itself: `listApiKeys` (`GET /api/v1/keys`),
  `createApiKey` (`POST /api/v1/keys`), `deleteApiKey` (`DELETE /api/v1/keys/{hashedKey}`).
  `createApiKey` returns the plaintext `key` **once** — store it immediately. Deletion
  addresses a key by its `hashedKey`, never by its value.

## 2. Resolve the chain and token first — do not guess

Call `listChains` (`GET /api/v1/chains`) before your first read. It returns each
supported `name` and the `tokens` valid **on that chain**. A token identifier that is
valid on one chain is a `400` on another. As of the last capture: `usdt` is broad,
`xaut` is on Ethereum/TON/Plasma, `usat` is Ethereum-only, `btc` is Bitcoin and Spark.

`listChains` also returns a per-chain `caseSensitive` object. **If the object is
present, addresses are preserved as-is; if it is absent, addresses are lowercased for
you.** Normalise your cache key the same way the API does, or you will store two
entries for one address.

## 3. Read one address

- Balance — `getTokenBalances`:
  `GET /api/v1/{blockchain}/{token}/{address}/token-balances`
  → `{ tokenBalance: { blockchain, token, amount } }`.
  `amount` is a **decimal string**. Keep it a string or use a big-decimal type; parsing
  it as a float will silently lose precision on 18-decimal tokens.

- History — `getTokenTransfers`:
  `GET /api/v1/{blockchain}/{token}/{address}/token-transfers`
  → `{ transfers: [ { blockchain, blockNumber, transactionHash, transferIndex, … } ] }`,
  sorted by block number.

## 4. Page history by moving the time window, not by a cursor

There is **no next-page token**. The only controls are:

| Param | Default | Bounds |
|---|---|---|
| `limit` | 10 | 1–1000 |
| `fromTs` | 0 | ms epoch, inclusive |
| `toTs` | latest | ms epoch, inclusive |

To walk a long history: request with a high `limit`, take the oldest returned transfer,
and re-request with `toTs` set just below its timestamp. Repeat until a page comes back
short. Do not assume a stable page boundary across calls — new blocks land at the head.

## 5. Prefer batch for more than a couple of addresses

`batchTokenBalances` (`POST /api/v1/batch/token-balances`) and `batchTokenTransfers`
(`POST /api/v1/batch/token-transfers`) each take an array of
`{ blockchain, token, address }` objects, **maximum 10 items**.

**The most important rule in this API:** the batch response is an array positionally
aligned with the request, where each element is *either* a result object *or* an error
envelope `{ error, message, status }` — and the HTTP status stays **200** even when
individual items failed. Check every element's shape. Code that only checks the
response status will record failures as empty balances.

## 6. Respect the budgets — nothing will warn you

Published per-key limits:

| Operation | Budget |
|---|---|
| `getHealth` | 10 / hour |
| `listChains` | 60 / minute |
| `getTokenBalances` | 4 / 10s |
| `getTokenTransfers` | 8 / 10s |
| `batchTokenBalances` | 4 / 10s |
| `batchTokenTransfers` | 8 / 10s |

There are **no rate-limit response headers and no `Retry-After`**. Implement your own
token bucket and exponential backoff on `429`; you cannot read your remaining budget
from a response. You *can* read your key's configured `max` and `timeWindow` from
`listApiKeys`.

Batching is the intended escape valve: ten addresses in one call costs one unit of a
4-per-10-seconds budget instead of ten.

## 7. Errors

Envelope is `{ error, message, status }` as `application/json` — **not** RFC 9457
problem+json, and there are no dereferenceable problem type URIs. Branch on HTTP status.

| Status | Meaning | Do |
|---|---|---|
| 400 | Bad address format, or wrong chain/token pair | Re-check against `listChains` |
| 401 | Key expired | Rotate via `createApiKey` |
| 403 | Key missing or invalid | Send `X-API-KEY` |
| 404 | Key not found (delete only) | Re-list keys |
| 429 | Rate limited | Back off; no `Retry-After` is sent |
| 500 | Indexer unavailable | Retry with backoff, then check `getHealth` |

`getHealth` (`GET /api/v1/health`) returns a **different body** from the error
envelope — `{ status, timestamp, summary: { healthy, unhealthy, total }, checks }` —
and returns HTTP **503** when any per-chain indexer is degraded. Parse it separately.
A 503 there means "some chains are behind", not "your request was wrong": inspect
`checks` before failing a user-facing operation.

## 8. No writes, no events, no idempotency

This API is read-only apart from key lifecycle. There is no `Idempotency-Key`
contract, no webhooks and no AsyncAPI — polling is the only mechanism for detecting a
new transfer. See `conventions/tether-conventions.yml`.
