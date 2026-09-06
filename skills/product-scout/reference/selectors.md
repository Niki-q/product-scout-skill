# Marketplace selectors and patterns

Status key: **V** = verified against a live page this session (date noted).
**U** = candidate, unverified — capture and confirm before trusting. A
selector's status is a claim about when it was last checked, not a
permanent property — marketplaces change their markup; if a pattern below
stops matching, downgrade it back to U in this file rather than silently
patching around it.

## Amazon — search results (`amazon.com/s?k=<query>`) — V (2026-09-06)

Each result card is `div[data-asin]` with a non-empty `data-asin` attribute
— this is the ASIN and the single most reliable identifier on the page,
present on organic *and* sponsored cards alike.

```js
document.querySelectorAll('div[data-asin]:not([data-asin=""])')
```

Per card:
- **Title**: `h2 span` (or `h2 a span`).
- **Sponsored flag**: check `h2`'s `aria-label` for `/Sponsored/i` — a
  sponsored card's `h2` has no wrapping `<a>`, only a `span`; its only link
  is a `/sspa/click?...` tracking redirect, never the product URL.
- **Price**: `.a-price .a-price-whole` + `.a-price .a-price-fraction`.
- **Rating**: an element matching `[aria-label*="out of 5 stars"]`; parse
  the leading float out of its `aria-label`.
- **Review count**: `a[aria-label*="ratings" i]` (falls back to
  `a.s-underline-text`); its `aria-label` is the count directly, e.g.
  `"19,235 ratings"`.
- **Canonical URL — do NOT scrape an in-card link.** Build it directly from
  the ASIN: `https://www.amazon.com/dp/<ASIN>`. Confirmed live: a
  sponsored card's only in-card anchor is a tracking redirect
  (`/sspa/click?...`), and even organic cards' in-card links carry
  session/ranking query params that don't belong in a canonical URL. This
  is the concrete mechanism behind the "ASIN gets mismatched with the wrong
  product" failure mode — always derive the URL from the ASIN, never trust
  a scraped anchor.

De-duplicate by ASIN — the same product commonly appears more than once per
page (sponsored + organic placements).

## Amazon — product page — V (2026-09-06)

No fixed selector table needed: verification is just "does the page title
(`document.title` or the H1) contain the search-result title, and does a
visible price element show the same figure." If either doesn't match,
treat the search-result card as unverified — see `browser.md` §5.

## eBay — search results (`ebay.com/sch/i.html?_nkw=<query>`) — V (2026-09-06)

Don't rely on `.s-item` alone as the card boundary — recent layouts mix in
`.s-card` styling and a small image-carousel widget that reuses the same
item id in a different, narrower `<li>`. Anchor on the **title** element
instead and walk up to its containing `<li>`:

```js
document.querySelectorAll('.s-card__title, .s-item__title').forEach(titleEl => {
  const li = titleEl.closest('li');
  const a = li.querySelector('a[href*="/itm/"]');
  // id = a.href.match(/\/itm\/(?:[^\/?]+\/)?(\d+)/)[1]
});
```

Per card (relative to that `<li>`):
- **Price**: `.s-card__price` / `.s-item__price`.
- **Shipping**: no single stable class — search descendant `span`/`div`
  leaves for text matching `/shipping|delivery|postage/i`. The shown figure
  is for whatever ship-to location the session currently defaults to (see
  the region caveat below) — never assume it's the buyer's actual region.
- **Sold count**: a `span` matching `/sold/i` (e.g. `"1,395 sold"`); absent
  on many listings — treat as `null`, not zero.
- **Rating**: often genuinely absent at the item level on eBay (feedback
  score lives at the seller level, not per-listing) — don't force a value.

Filter out placeholder/ad cards before use — one observed id (`123456`) was
a "Shop on eBay" filler card with no real listing behind it, not a parsing
bug.

**Search-URL filter parameters — U, blocked by bot-check.** Two separate
live attempts to add query parameters to the search URL (once
location-only, once a plain price/condition/format combination) both
triggered eBay's bot-check interstitial (`ebay.com/splashui/challenge`).
Neither is safe to treat as verified — see `ebay.md` §2 for the resulting
rule (bare-keyword search, filter extracted candidates client-side
instead). Condition and buying-format text per card were not extracted or
verified this session as a consequence — a card's condition (if needed)
currently has to be read off its own product page during the verification
step, not the search card.

**Region/currency — a real, confirmed gap, not a hypothetical:** an
unauthenticated `ebay.com` search returns USD prices that are themselves a
converted estimate — visiting one listing's own product page showed its
*actual* native price in GBP (seller located in the UK), with the US-site
search card showing only the converted USD figure and "May not ship to
United States." Do not trust a search card's price/shipping as final for a
non-US buyer — the product page is the source of truth for both.

**Do not try to force a ship-to location via query parameters
(`_stpos`, `LH_PrefLoc`, etc.) — confirmed to trigger eBay's bot-check
interstitial** (`ebay.com/splashui/challenge`, "Pardon Our Interruption").
Per `browser.md` §2, hitting that page is a stop, not a retry-with-different-
params. Where a country-specific storefront is needed, use eBay's own
domain mapping instead (`ebay.co.uk`, `ebay.de`, etc. — map the user's
region to the closest one) rather than forcing location via the default
domain's query string.

## AliExpress — search results (`aliexpress.com/wholesale?SearchText=<query>`) — V (2026-09-04/05, re-confirmed 2026-09-06)

See `browser.md` §3 for the full extraction snippet (price is read from the
product link's `pdp_npi` query param, not from visible DOM text — visible
price/rating/sold text nodes sit close enough together that naive
`textContent` regexes concatenate them into garbage numbers).

- Item links match `a[href*="/item/"]`, with the numeric id in
  `/item/(\d+)\.html`.
- Region confirmation: a localized session's result URLs carry a country
  tag in the encoded `pdp_npi` string — confirmed live in two different
  positions: `!sea!CY!` on organic results, `!ct!CY!` on sponsored/p4p
  results (`/!(?:sea|ct)!([A-Z]{2})!/` matches both) — this is a more
  reliable localization signal than the page's hostname (a CY-region
  cookie session can still get redirected to a `tr.aliexpress.com` or
  similar hostname via `gatewayAdapt=...`, which does *not* mean the
  region/currency reverted — check the encoded region tag or the rendered
  currency, not the hostname).
- Cookies set once for a region (via `context.addCookies()`) persist in the
  browser profile on disk across separate tool-call sessions, even after
  the underlying browser process was restarted — confirmed live (EUR
  pricing was still active on a fresh navigation after the Chrome process
  had been killed and relaunched). Don't assume region cookies need
  re-applying every run; check the rendered currency first.
- Trust filter: prefer `rating >= 4.5` **and** a sold count in the
  hundreds or more. A very low price paired with a low rating and a
  handful of sales (observed: 3.3★, 3 reviews, 18 sold) is a low-trust
  listing regardless of how cheap it looks — don't surface it as a top pick
  without flagging that explicitly.
- Genuine branded products are often **not listed under their retail
  name** — e.g. searching "Xiaomi Mi Body Composition Scale 2" by its exact
  retail name surfaced only unrelated generic scales; the actual current-gen
  equivalent (Xiaomi's own "Mijia S200/S400" line) only turned up under a
  broader query. Absence of the exact name is not evidence the
  product/category doesn't exist on the platform — broaden the query before
  concluding that.
