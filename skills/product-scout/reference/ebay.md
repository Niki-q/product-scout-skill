# eBay pipeline

No mature community skill existed for personal (non-seller) eBay shopping
research at the time this was written — this pipeline was built from
scratch and verified live, not ported from a reference implementation.

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

## 4. Shipping and region — check per-listing, don't assume

eBay shipping cost and availability are set per-seller, per-listing — there
is no single "shipping to region X costs Y" rule the way some marketplaces
have. Read the actual shipping section on each verified listing's product
page rather than extrapolating from the search card's shipping text, which
reflects whatever ship-to default the current browser session has (see
`browser.md` §4) and can be silently wrong for the buyer's actual region.

**Do not try to force a specific ship-to location via URL query parameters**
(`_stpos`, `LH_PrefLoc`, and similar) on the default `ebay.com` domain —
confirmed live to trigger eBay's bot-check interstitial
(`ebay.com/splashui/challenge`, "Pardon Our Interruption"). Per
`browser.md` §2, that's a stop, not a signal to retry with different
parameters. If a country-specific price/availability view is genuinely
needed, use eBay's own region-specific storefront domain instead.

## 5. Ranking

No item-level star rating exists on most eBay listings (feedback is a
seller-level metric, not per-listing) — rank primarily on sold count and
price/condition fit against the buyer's criteria, and note plainly when a
listing has no rating signal at all rather than presenting it as
equivalent to a rated one from another marketplace.
