---
name: thecarapi-vehicle-search
description: >-
  Search TheCarApi's multi-source auction inventory and page through it correctly — building a
  cross-filtered facet sidebar, choosing one pagination spelling, and persisting the right
  identifier. Use when an agent must find vehicles matching criteria across Auto1, OpenLane,
  eCarsTrade, Schadeautos, Copart DE, Encar or the Japanese auction houses.
api: TheCarApi REST API
base_url: https://api.thecarapi.com
operations:
  - get_api_facets
  - get_api_search
  - get_api_models
  - get_api_sites
  - get_api_contract
scopes:
  - search
generated: '2026-09-01'
method: generated
source: openapi/thecarapi-openapi.json + https://thecarapi.com/docs/recipes + https://thecarapi.com/docs/conventions
---

# Search TheCarApi auction inventory

Grounded in operations verified present in `openapi/thecarapi-openapi.json`. Every operationId
below appears in that document.

## Before you start

1. Call `get_api_contract` (`GET /api/contract`) once. Read `pagination.max_limit`,
   `pagination.max_offset` and the `schemas` object rather than hardcoding field lists. If the
   route answers `403`, the `ops` scope was not granted on this key — that is a provisioning
   problem, not a runtime one. Fail loudly.
2. Call `get_api_sites` (`GET /api/sites`) for the live source slug set. Never hardcode it — an
   unknown slug is a `400`, so a stale list turns a new source into a broken deploy.
3. Authenticate with `X-API-Key: <key>`. Send `Accept-Encoding: gzip`.

## Step 1 — build the filter sidebar in one call

`get_api_facets` (`GET /api/facets?fields=brands,years,fuels,gearboxes,countries,sites`) returns
six dimensions in one response and costs **one** unit of quota, versus six for the per-dimension
endpoints. Use it.

Facets cross-filter: pass the filters the user has already chosen and every remaining dimension
recounts against them, while each dimension ignores its own filter so the user can change their
mind without the option disappearing. `is_active` and `include_ended` are not accepted here —
facets always describe lots whose auction is still open.

Models are brand-scoped and stay on their own route: `get_api_models`
(`GET /api/models?brand=bmw&ordering=-count`).

Facet responses are served `max-age=600` with an `ETag`. Store it and send `If-None-Match`; an
unchanged sidebar refresh costs an empty `304`.

## Step 2 — search

`get_api_search` (`GET /api/search`) is the primary operation, with 32 documented parameters.

- **Pick one pagination spelling and stay with it.** `page` + `page_size`, or `offset` + `limit`.
  Mixing them — `page` with `offset`, or `page_size` with `limit` — is a `400`.
- `limit` is capped at **100** across the API; larger values are clamped down, not rejected.
  `limit=0` or a negative value is a `400`, and `page=0` is a `400`.
- Default page size is 100 on `/api/search` and 50 everywhere else.
- `/api/search` has **no offset cap**. A very deep *filtered* search may return `503` asking you
  to narrow it — narrow the filter, do not retry the same URL. Pages past offset 5000 are not cached.
- Ended lots are hidden by default. `is_active=false` or `include_ended=true` shows them **in
  addition to** live ones; neither returns only ended lots. If both are sent, `include_ended` wins.

### Counting is the expensive half

- Streaming rather than showing a page count? Send `include_total=false`.
- Want only the number? Send `count_only=true` and get no rows. Note `page` and `page_size` come
  back `null` on a `count_only` response — it has no pages to describe.

## Step 3 — persist the right identifier

Persist **`site_name` + `auction_id_str`**, never `auction_id` alone.

`auction_id` and `auction_id_str` carry the same id, but `japanauction` ids exceed 2^53 for almost
every lot. Any client parsing JSON numbers as IEEE-754 doubles silently rounds `auction_id`, and
the rounded value no longer addresses the row it came from. `auction_id_str` is the exact value.

The same caveat applies to `vehicle_id`, which duplicates `auction_id`.

## Related operation

`get_api_search_auction_ids` (`GET /api/search/auction-ids`) returns bare numeric `auction_id`
values with no source attached — they are not unique across sources. It is a filter, not an
addressing scheme; use `get_api_search` when you need cards you can address. It is not enabled on
a new key by default. An empty list on a `503` means "unknown", not "zero".

## Errors and retries

| Status | Do |
| --- | --- |
| 400 | Fix the request. The body names the offending value and lists what would have been accepted. |
| 401 | Stop. Five failed authentications from one IP lock that address out for 15 minutes, even for a valid key. |
| 403 | The scope was not granted on this key. Ask the operator; do not retry. |
| 429 | Wait `Retry-After` seconds. Never back off on your own schedule. |
| 503 | Retry with backoff — unless it came from a deep filtered search, which is asking you to narrow the filter. |

Never retry 400, 401, 403 or 404. Read `X-Request-ID` from the response header on any failure —
it is the only handle support has on a specific call.

Unknown query parameters are currently ignored. Do not rely on it: the provider says they are on
their way to becoming a `400`. Treat a filter that seems to have no effect as a misspelling.
