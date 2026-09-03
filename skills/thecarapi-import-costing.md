---
name: thecarapi-import-costing
description: >-
  Find under-priced auction lots on TheCarApi, cost one landed, and check the result against the
  retail market before treating it as a deal. Use when an agent must answer "is this car worth
  buying at this price, delivered to this country?"
api: TheCarApi REST API
base_url: https://api.thecarapi.com
operations:
  - get_api_top_offers
  - get_api_auction_site_slug_auction_id
  - get_api_calculator_countries
  - post_api_calculator_calculate
  - get_api_cars_bg_market
  - get_api_auction_market
scopes:
  - top-offers
  - auctions
  - calculator
  - market
generated: '2026-09-01'
method: generated
source: openapi/thecarapi-openapi.json + https://thecarapi.com/docs/recipes + https://thecarapi.com/docs/calculator
---

# Price an import with TheCarApi

Grounded in operations verified present in `openapi/thecarapi-openapi.json`.

> These are estimates, not a binding quote. The provider says so on the calculator itself. Never
> present the output as a price a user will actually pay.

## Step 1 — find candidates

`get_api_top_offers` (`GET /api/top-offers`) returns live auctions the pipeline judged to be
priced below their market reference. The same lots come back from
`GET /api/search?sort=top_offers`, but the top-offers cards additionally carry the
`market_reference` the verdict was made against — use this route so you can show your working.

`/api/top-offers` is capped at offset 5000 and returns only `total`, `limit`, `offset` and
`total_pages` — page by `offset` here, not by `page`.

## Step 2 — read the lot's own fee inputs

Open the lot with `get_api_auction_site_slug_auction_id` and read `is_margin` off the detail
response before costing it.

For **eCarsTrade**, pass the lot's own `is_margin` value. Omitted, it prices as VAT-deductible,
which is the dearer quote — so a caller who leaves it out is never under-quoted, but is often
over-quoted.

For **Copart**, price a running lot from `final_price`, which tracks the live high bid.
`current_price` trails it.

## Step 3 — cost it landed

`get_api_calculator_countries` (`GET /api/calculator/countries`) first: an origin or destination
code outside that set is a `400` naming it. Note the spelling trap — the calculator spells the
United Kingdom **UK** where the vehicle facets spell it **GB**.

`post_api_calculator_calculate` (`POST /api/calculator/calculate`) with `price`, `site_name`,
`origin` and `destination`. Sending `site_name` matters: for a source with a fee profile, `price`
plus `site_name` reproduces the same model that computed that lot's `buynow_final` /
`current_final`, so the two agree. Omit it and you get the generic model, which will not.

The breakdown, in the order the arithmetic runs:

```
subtotal_customs_value = lot_price + auction_fee + trucking + shipping
duty_amount            = duty_rate % of subtotal_customs_value
vat_amount             = vat_rate % of (subtotal_customs_value + duty_amount)
custom_clearance_total = duty_amount + vat_amount + customs_agency
estimated_total        = subtotal_customs_value + custom_clearance_total + our_fee
```

Two things clients get wrong: **VAT is charged on `subtotal_customs_value + duty_amount`, not on
the lot price alone**, and `our_fee` plus `customs_agency` sit *outside* the duty/VAT base.
`trucking` is 0 under a source profile, where it is folded into `shipping`.

Rejections are all `400`: a missing price; a price that is not a number, is negative, or exceeds
10,000,000; a `car_type` other than `standard` or `classic`. `classic` relief (no duty, reduced
VAT) applies only where duty applies at all — an import into the EU from outside it.

This response carries **no envelope metadata**. Read `X-Request-ID` from the header.

## Step 4 — compare against retail, not against the lot price

`get_api_cars_bg_market` (Bulgarian retail) and `get_api_auction_market` (our own auction
inventory) return precomputed snapshots for a brand/model/year window.

- **Check `listing_count` before you trust the median.** A median over three listings is noise.
- A `404` means no snapshot exists for that window — the normal answer for a thin brand/model/year
  combination, not an error to retry.
- Neither route carries envelope metadata; read `X-Request-ID` from the header.
- **Neither is enabled on a new key by default.** A `403` here means the `market` scope was not
  granted — ask the operator.

Compare the **landed total** with the retail market. Comparing the lot price with retail is the
mistake this whole flow exists to prevent.
