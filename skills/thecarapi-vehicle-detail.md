---
name: thecarapi-vehicle-detail
description: >-
  Open one vehicle on TheCarApi and render it correctly — resolving detail, the live bid, the
  photo gallery and the three pending states that look like errors and are not. Use when an agent
  has a site_name + auction_id pair and needs the full picture of that lot.
api: TheCarApi REST API
base_url: https://api.thecarapi.com
operations:
  - get_api_auction_site_slug_auction_id
  - get_api_auction_images_site_slug_auction_id
  - get_api_car_details
  - post_api_car_details
  - get_api_contract
scopes:
  - auctions
  - details
generated: '2026-09-01'
method: generated
source: openapi/thecarapi-openapi.json + https://thecarapi.com/docs/recipes + https://thecarapi.com/docs/schema
---

# Open one vehicle on TheCarApi

Grounded in operations verified present in `openapi/thecarapi-openapi.json`.

## One request, not three

`get_api_auction_site_slug_auction_id` (`GET /api/auction/{site_slug}/{auction_id}`) returns
specification, condition and gallery together. Since contract `2026-08-19` it embeds
`vault_gallery` — the exact body of `/api/auction-images/{site}/{id}` (`images`, `count`,
`pending`). Both routes read one cache entry and can never disagree, so calling
`get_api_auction_images_site_slug_auction_id` as well is a wasted request. Use the separate route
only when you want the gallery alone.

A vehicle is addressed by the pair `site_name` + `auction_id_str` — for example
`openlane/11409652`. Address with `auction_id_str`, not `auction_id`.

## Branch on price, in this order

```
live_price present            -> that IS the current bid. Render it.
live_price_pending present    -> read once more after ~2 s, then stop.
neither                       -> the cycle price is the price. Do not retry.
```

A missing `live_price` block is **normal, not an error**. For most listings the cycle price is the
only price there is. Only running OpenLane and eCarsTrade auctions have their bid read from the
auction house at request time; check `live_prices.enabled` and `live_prices.sites` on
`get_api_contract` to know whether this deployment does it at all — that block's absence is the
only way to tell "live prices are unavailable" from "this car has none right now".

Search results are always cycle-priced. A detail page legitimately showing a higher
`current_price` than the search card it was opened from is a bid that moved, not an inconsistency.

## The three states that look like failures

- **`is_blind: true`** — the auction house publishes no bid at all, by design. This is the one
  case where `current_price: null` is a final answer rather than missing data. No live price will
  ever arrive. Most eCarsTrade auctions are blind. Show `estimated_value_eur` instead, **labelled
  as the auction house's own estimate — it is not a price you can pay**, and do not sort or filter
  on it.
- **`details_pending: true`** — the upstream fetch is still running and `vehicle_details` is
  omitted. Re-read on the short `max-age` the response carries and render the rest meanwhile.
  Poll; do not retry in a tight loop.
- **`vault_gallery.pending > 0`** — more photos are coming. Both routes are served `max-age=10`
  while pending is non-zero, so poll on that.

An empty `inspection_reports` or `images` array is also normal — most listings publish no reports.

## Photos

`served_url` is a **path**, not an absolute URL. Join it to the base URL. Fall back to
`remote_url` where `served_url` is absent.

`images[]` on a search or `listVehicles` row is capped at the first **8** display-ordered entries,
primary first — a card preview, never the full gallery. Read `vault_gallery` on the detail
response, or the dedicated images route.

`/image-vault/*` paths answer with a `302` to a pre-signed object URL rather than streaming bytes.
Mainstream clients follow it transparently and the object is immutable-cached, so the hop is paid
once per photo.

## The alternative detail route

`get_api_car_details` / `post_api_car_details` (`GET|POST /api/car-details`) returns the full
source payload. Two differences worth knowing:

- On `auto1` and `japanauction` it takes the offer id/UUID rather than the numeric auction id.
- Its response carries **no envelope metadata** — no `contract_version`, `request_id`,
  `server_time` or `data_updated_at`. Read the correlation id from the `X-Request-ID` header.

`POST` exists only because the parameter set is too large for a query string. It creates nothing
and changes no state; repeating it is safe. The body cap is 50 MB — over that is a `413`.

## Reading the normalized block

`vehicle_details` reconciles differently-named source payloads — a downloadable file is
`Documents[]` on Auto1 but `ECarsTradeDocuments[]` on eCarsTrade, while `Documents` on OpenLane is
a dict of paperwork booleans and not a file list at all. **Read `vehicle_details` instead of
branching on `site_name`.** Every key is omitted when the source has nothing for it.

`co2` (measured) and `co2_estimated` (derived) must never be merged; `co2_estimated_standard`
carries the NEDC/WLTP cycle the estimate is stated against.

Drive countdowns from `auction_end_at` (absolute UTC close instant), not from `batch_end_date`
(per-house timezone) or `auction_sec_left` (a snapshot taken at the last refresh).
