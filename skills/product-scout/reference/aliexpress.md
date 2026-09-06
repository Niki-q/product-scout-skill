# AliExpress pipeline

Architecture ported from `danielrosehill/Aliexpress-Israel-Skills` — the
selector-verification discipline (§`browser.md`, `selectors.md`) and the
"no parallel tabs, CAPTCHA is a stop" rules — but **not** its tax logic.
That skill is built around Israeli VAT bands (the $75 de-minimis, ILS/FX
handling) and Hebrew/`he.aliexpress.com` locale handling; none of that
applies here. This adapts the same architecture for EU/Cyprus pricing and
shipping instead.

## 1. Region must be real, not assumed

AliExpress prices and shipping estimates are meaningless unless the
session is actually localized to the buyer's region. A localized session
shows EUR pricing and carries a country tag in its result URLs (see below);
an unauthenticated/default session may show USD or a different region's
numbers with no warning. If the buyer can provide a cookie export from a
logged-in session already set to their region (as was done to build this
pipeline — a `region=CY, currency=EUR` cookie set applied via
`context.addCookies()`), use it; otherwise flag explicitly that shown
prices/shipping may not reflect the buyer's actual region and cost
(`browser.md` §4).

**A regional cookie session can still land on a different-looking hostname
** (e.g. `tr.aliexpress.com` via a `gatewayAdapt=...` redirect) without the
region/currency actually changing — don't read the hostname as the
localization signal. Read the rendered currency, or the country tag
embedded in result URLs (see below).

## 2. Search and extract candidates

Navigate to `aliexpress.com/wholesale?SearchText=<query>`. Use the
extraction snippet in `browser.md` §3 — it reads price from the product
link's `pdp_npi` query parameter rather than visible DOM text, which is the
verified-reliable approach (visible price/rating/sold-count text nodes sit
close enough together that naive text-based regexes concatenate them into
garbage numbers like `"€41,254.4800"`). The same encoded string carries a
country tag (`!sea!CY!` on organic results, `!ct!CY!` on sponsored ones)
that confirms which region's pricing is
actually being shown — check that instead of trusting the hostname.

## 3. Verify on the real product page

Same principle as every other marketplace: load the item's own
`aliexpress.com/item/<id>.html` page and confirm title/price before
presenting it as a result.

## 4. Trust filtering — AliExpress-specific

Price and even a high star rating can both be misleading in isolation —
apply both a rating floor **and** a sold-count floor together:
- Prefer `rating >= 4.5` **and** sold count in the hundreds or more.
- A cheap listing with a low rating and a handful of sales (observed
  live: 3.3★, 3 reviews, 18 sold) is low-trust — don't surface it as a top
  pick without saying so explicitly, even if it's the cheapest match.

## 5. Genuine branded products may not appear under their retail name

Searching for a specific retail product by its exact marketing name can
come up empty or return only unrelated generic listings, even when the
brand/category is well represented on the platform under a different,
current-generation name. Confirmed live: searching "Xiaomi Mi Body
Composition Scale 2" by name surfaced nothing relevant, while the actual
current equivalent line (Xiaomi's own "Mijia S200/S400" smart scales)
turned up clearly once the query was broadened. Treat an empty/irrelevant
result for an exact product name as a cue to broaden the search (brand +
category, drop the specific model name) before concluding the platform
doesn't carry it.

## 6. Shipping cost — EU/Cyprus specifics

Unlike eBay (per-listing, seller-set), AliExpress shipping to an EU/CY
address is usually either genuinely free (common on "Shipped by
AliExpress"-fulfilled listings) or a flat estimate shown directly on the
search card / product page for the localized session. Read whichever
figure the localized session actually shows rather than assuming a fixed
shipping cost across listings — it varies by seller and fulfillment method
even within one search.
