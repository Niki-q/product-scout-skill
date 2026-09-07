# eBay pipeline

No mature community skill existed for personal (non-seller) eBay shopping
research at the time this was written — this pipeline was built from
scratch and verified live, not ported from a reference implementation.

## 0. Preferred path: the `ebay-browse-mcp` MCP server

If the `ebay_search_items` / `ebay_get_item` tools are available (the
[`ebay-browse-mcp`](https://github.com/Niki-q/ebay-browse-mcp) MCP server
is configured), **use them instead of everything below** — Playwright/DOM
scraping is the fallback for when that server isn't set up, not the
default. It wraps eBay's official Browse API (app-level OAuth, no user
login, no bot-check risk) and was built specifically because the
Playwright path here kept hitting eBay's bot-check (§2/§4 below).

Pipeline via the MCP tools:

1. `ebay_search_items({ keyword, country })` — `country` is the buyer's
   2-letter region (`CY`/`UA`/`MD`). Returns candidates with title, price,
   condition, canonical `itemWebUrl`, and a *coarse* shipping estimate —
   candidates, not verified cards, same as the scraped path.
2. For each candidate worth keeping, `ebay_get_item({ itemId, country })` —
   this **is** the verification step (§3) and the shipping step (§4) in one
   call. There's no title/price scraping mismatch to guard against here:
   `getItem` is authoritative API data for that exact `itemId`, not a
   second DOM read that could drift from the search result. Its
   `shippingOptions` carries the real, region-accurate cost.

Field mapping, `ebay_get_item` response → `card-schema.md`:

| Browse API field | Card field | Notes |
|---|---|---|
| `itemId` | `id` | |
| `title` | `title` | Already authoritative — see above, no separate verification read needed. |
| `itemWebUrl` | `url` | Canonical, straight from the API — no tracking-param stripping needed. |
| `price.amount` / `price.currency` | `price.amount` / `price.currency` | |
| `condition` | `condition` | eBay's condition strings are more granular than the schema's 3 buckets — map `NEW*` → `"new"`; `CERTIFIED_REFURBISHED` / `SELLER_REFURBISHED` → `"refurbished"`; everything else (`USED_*`, `FOR_PARTS_OR_NOT_WORKING`, etc.) → `"used"` (note "for parts" separately in presentation prose, don't silently call it ordinary used). |
| `shippingOptions[0].cost` | `shipping.cost` | Already normalized to `"free"` when eBay charges nothing — pass through as-is. |
| `shippingOptions[0].currency` | `shipping.currency` | |
| `shippingOptions[0].minDays`–`maxDays` | `shipping.eta_days` | Format as `"<min>-<max>"`, same range convention as the other marketplaces. |
| `shipToLocationUnavailable` | — | If `true`, treat `shipping.cost` as unavailable for that region (`null`), not as free/zero — don't let an empty `shippingOptions` array silently read as "no shipping cost". |
| `seller.feedbackScore` | — | No schema field for this (it's seller-level, not item-level, same reason `rating` stays `null` for eBay per §5) — mention it in presentation prose as an informal trust signal if notably low, don't invent a schema slot for one marketplace's quirk. |

`rating` and `sold_count` stay `null` via the API the same as via scraping
— the Browse API doesn't expose either at the item level; §5's ranking
guidance (sold count / price fit, don't fabricate a rating) still applies.

---

**Everything below (§1-§5) is the Playwright fallback** for when
`ebay-browse-mcp` isn't configured. Criteria-gathering (§1) and ranking
(§5) apply either way; §2-§4 are scrape-specific and moot once the MCP
tools are in play.

## 1. Gather criteria, same as Amazon

Budget, use case, deal-breakers — see `amazon.md` §1, identical reasoning.
eBay adds one more axis worth asking about explicitly: **condition**
(new / used / refurbished) and **buying format** (fixed-price "Buy It Now"
vs. auction) — a buyer who didn't mean to bid on an auction ending in six
days needs that ruled out up front, not discovered on the product page.
These get applied by filtering extracted candidates, not via search-URL
parameters — see §2's bot-check finding.

## 2. Search and extract candidates

Navigate to `ebay.com/sch/i.html?_nkw=<query>` — **bare keyword only.**
Filter server-side via extra query parameters (`_udlo`/`_udhi` for price,
`LH_ItemCondition`, `LH_BIN`, `_stpos`, `LH_PrefLoc`, etc.) at your own
risk: confirmed live, twice, that adding query parameters to this URL —
once with only location params, once with a completely ordinary
combination of price range + condition + buying-format params — triggered
eBay's bot-check interstitial both times. This may be about automation
fingerprinting in general rather than these specific parameters, but it
was not possible to confirm any filter parameter as safe in this session.
**Until someone verifies otherwise, treat every eBay search query parameter
beyond `_nkw` as unverified and risky** — search bare keywords, pull a
larger candidate list than you need, and apply budget/condition/format
filters by inspecting each extracted candidate's own fields instead of
asking eBay's server to pre-filter.

Map the buyer's country to the closest eBay domain (`ebay.co.uk`,
`ebay.de`, etc.) if a country-specific storefront is needed — that's a
different mechanism (subdomain, not query param) and wasn't implicated in
either bot-check.

Extract with the title-anchored pattern in `selectors.md` — anchor on the
title element and walk up to its `<li>`, don't trust `.s-item`/`.s-card` as
the card boundary on their own (a small image-carousel widget reuses the
same item id in a narrower, wrong-shaped `<li>`).

Filter out placeholder cards (a "Shop on eBay" filler card with a fake id
was observed in a live run — not every element matching the item-link
pattern is a real listing).

## 3. Verify on the real product page — same principle as Amazon

Load the item's own `ebay.com/itm/<id>` page and confirm title match.
eBay adds a wrinkle worth checking every time, not just once: a listing's
**search-result price is often a currency-converted estimate** — a live
verification found a listing priced at GBP 4.29 (UK seller) showing as
"US $5.80 (approx.)" on the US-domain search card, with the product page
stating it "may not ship to United States" at all. Don't treat the
search-card price/availability as final — the product page's native price
and its own shipping/location section are the source of truth.

## 4. Shipping and region — check per-listing, don't assume, and it's mandatory

eBay shipping cost and availability are set per-seller, per-listing — there
is no single "shipping to region X costs Y" rule the way some marketplaces
have. Read the actual shipping section on each verified listing's product
page rather than extrapolating from the search card's shipping text, which
reflects whatever ship-to default the current browser session has (see
`browser.md` §4) and can be silently wrong for the buyer's actual region.
This is required for every card, not optional — compute `total_price`
(item + shipping) per `card-schema.md` the same as every other
marketplace; don't let "it's per-listing and annoying to check" become an
excuse to skip it.

**Never force ship-to location via URL query parameters** (`_stpos`,
`LH_PrefLoc`, and similar) — confirmed live to trigger eBay's bot-check
interstitial (`ebay.com/splashui/challenge`, "Pardon Our Interruption").
The legitimate mechanism is the **"Ship to" button in the site header** →
"Set your shipping location" → its own "Ship to: <country>" button opens a
country picker — see `regions.md` for the exact confirmed path and its
current CY/UA/MD status. Per `browser.md` §2, hitting the bot-check is a
stop, not a signal to retry with different parameters — and note that this
UI path itself was seen to trigger the same bot-check on a subsequent
search in testing, so treat eBay overall as bot-check-prone and space out
requests rather than assuming the header UI is a safe workaround in every
run. Confirmed again in a later, separate session on the same browser
profile: even a completely bare `_nkw=`-only keyword search (no region
change, no other parameters, first request of that run) hit the same
interstitial — this looks like the automation profile itself accumulating
a bot-risk score over a day of testing rather than a per-request trigger.
If eBay keeps bot-checking a fresh, minimal request, that's a sign the
current browser profile is flagged and a plain retry won't help — say so
plainly rather than repeatedly retrying.

## 5. Ranking

No item-level star rating exists on most eBay listings (feedback is a
seller-level metric, not per-listing) — rank primarily on sold count and
price/condition fit against the buyer's criteria, and note plainly when a
listing has no rating signal at all rather than presenting it as
equivalent to a rated one from another marketplace.
