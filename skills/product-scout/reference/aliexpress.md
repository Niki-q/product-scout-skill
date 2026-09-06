# AliExpress pipeline

Architecture ported from `danielrosehill/Aliexpress-Israel-Skills` — the
selector-verification discipline (§`browser.md`, `selectors.md`) and the
"no parallel tabs, CAPTCHA is a stop" rules — but **not** its tax logic.
That skill is built around Israeli VAT bands (the $75 de-minimis, ILS/FX
handling) and Hebrew/`he.aliexpress.com` locale handling; none of that
applies here. This adapts the same architecture for the regions this
project targets — Cyprus, Ukraine, Moldova — instead; see `regions.md` for
per-region status (short version: CY and UA work, MD does not).

## 1. Region must be real, not assumed

AliExpress prices and shipping estimates are meaningless unless the
session is actually localized to the buyer's region. A localized session
carries a country tag in its result URLs (see below); an
unauthenticated/default session may show USD or a different region's
numbers with no warning, and even a *correctly* localized session's
currency isn't always the one you'd expect (Ukraine localizes to UAH, not
EUR — see `regions.md`).

The region switch is a normal logged-in-user UI flow, not something
requiring a special cookie export in general — click the header's
country/flag icon, pick a country from the list, save. A pre-authenticated
cookie jar (as used to build this pipeline, applied via
`context.addCookies()`) just saves re-doing that click-through and any
login every run; it's a convenience, not a requirement. Either way,
confirm the session actually localized to the target region before
trusting a price (`browser.md` §4) — **and for Moldova specifically, don't
even attempt this: see `regions.md`, region selection is broken and
silently redirects to an unrelated Russia-market site.** For CY/UA, see
`regions.md` for the confirmed mechanics and currency-per-region details.

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

## 6. Shipping cost

Unlike eBay (per-listing, seller-set), AliExpress shipping to a localized
address (confirmed for CY and UA) is usually either genuinely free (common
on "Shipped by AliExpress"-fulfilled listings) or a flat estimate shown
directly on the search card / product page for the localized session. Read
whichever figure the localized session actually shows rather than assuming
a fixed shipping cost across listings — it varies by seller and
fulfillment method even within one search. For Moldova, this doesn't apply
— see §1 and `regions.md`, there's no working localized MD session to read
a shipping figure from in the first place.
