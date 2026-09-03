---
name: thecarapi-feed-sync
description: >-
  Mirror TheCarApi's auction feed into a local store cheaply and keep it in sync — walking search
  at maximum depth, revalidating with ETags, and hydrating detail only for rows that actually
  moved. Use when an agent maintains a copy of the inventory rather than answering one query.
api: TheCarApi REST API
base_url: https://api.thecarapi.com
operations:
  - get_api_search
  - get_api_listVehicles
  - post_api_listVehicles
  - get_api_auction_site_slug_auction_id
  - get_api_auction_site_slug_auction_id_price_history
  - get_api_vin_vin_history
  - get_api_contract
  - get_api_health
scopes:
  - search
  - details
  - auctions
  - ops
generated: '2026-09-01'
method: generated
source: openapi/thecarapi-openapi.json + https://thecarapi.com/docs/recipes + https://thecarapi.com/docs/conventions
---

# Mirror the TheCarApi feed

Grounded in operations verified present in `openapi/thecarapi-openapi.json`.

## The four levers

An unfiltered `get_api_search` walk has no depth limit. Four things make it an order of magnitude
cheaper, and all four are published:

1. **`Accept-Encoding: gzip`** — roughly an eighth of the bytes on a full 100-row page. Anything
   over 2 KB is compressed. PHP Guzzle needs `decode_content` set explicitly; most clients
   negotiate it transparently.
2. **`include_total=false`** — counting is the expensive half of a search, and a mirror does not
   need a page count.
3. **`ETag` + `If-None-Match`** — store each page's weak ETag and send it on the re-walk.
   Unchanged pages answer `304 Not Modified` with no body. Compare ETags as opaque strings and
   echo them back exactly as received: do **not** strip the `W/` prefix or the quotes.
4. **Do not expand every card.** Hydrate detail only for rows whose `last_changed_at` moved, or
   that you actually display.

```
GET /api/search?limit=100&offset=0&include_total=false
Accept-Encoding: gzip
X-API-Key: <key>
If-None-Match: W/"..."
```

`limit` is capped at 100. Pages past offset 5000 are not cached, so they cost more — but they are
reachable, because `/api/search` and `/api/listVehicles` have no offset cap.

## What search rows do and do not carry

Search rows are **always cycle-priced, never live-priced**. That is what makes them fast enough to
page. Do not build a mirror that claims to hold current bids.

`images[]` on a row is capped at 8 display-ordered entries — a card preview. If your mirror needs
galleries, read `vault_gallery` on the detail response per row you actually keep.

Persist `site_name` + `auction_id_str`. `auction_id` rounds for `japanauction` ids above 2^53.

## The bulk route

`get_api_listVehicles` / `post_api_listVehicles` (`GET|POST /api/listVehicles`) is the other
uncapped route. Note two things: it carries **no envelope metadata** (read `X-Request-ID` from the
header), and it is **not enabled on a new key by default** — a `403` means the scope was never
granted. `POST` exists for large parameter sets; the body cap is 50 MB (`413` over it). It is a
query, not a mutation.

## Change tracking

- `get_api_auction_site_slug_auction_id_price_history`
  (`GET /api/auction/{site_slug}/{auction_id}/price-history`) returns price events oldest first,
  capped at 5,000 events. `event_type` is `initial`, `baseline` or `change` — **there is no
  `price_change` value**, despite what older documentation printed. A client matching on
  `price_change` discards every event.
- `get_api_vin_vin_history` (`GET /api/vin/{vin}/history`) shows a car that has been through
  auction more than once. Not enabled on a new key by default.
- Row timestamps for a delta walk: `created_at`, `updated_at`, `first_seen_at`, `last_seen_at`,
  `last_available_at`, `last_changed_at`, and `archived_at` on archived rows.

## Operational discipline

- Call `get_api_contract` at startup, not per request, and gate on the declared schema keys rather
  than comparing version strings. `pagination.max_limit` and `pagination.max_offset` are published
  there — read them rather than discovering them by being clamped.
- `get_api_health` (`ops` scope) reports `status: healthy | degraded`. **A degraded service still
  answers 200**, with the reason under `schema_findings` — alert on the status field, not the HTTP
  code. `typesense.healthy: false` is what turns free-text `?search=` into a plain database match
  and is the usual explanation for a text search returning less than expected.
- `/api/health/live` and `/api/health/ready` need no key and consume no quota. Point an uptime
  monitor at those.

## Rate limits

Fixed-window quotas per key on any of minute / hour / day / month. Windows are fixed, not sliding,
so a boundary burst of two full windows back to back is expected. A key with no quota sends **none
of the rate-limit headers** — their absence means unlimited, not exhausted.

On `429`: honour `Retry-After`. On `503`: back off — unless it came from a deep *filtered* search,
which is asking you to narrow the filter rather than wait. Never retry 400, 401, 403 or 404.

## Data you will not receive

Account-scoped commercial data is stripped at the response boundary — auction fees, bid history,
transport and every `delivery*`/`selfpickup*` key, the provider's own buyer-account identity, and
internal processing metadata. Email addresses are redacted from free-text values. Do not build a
mirror schema expecting them.
