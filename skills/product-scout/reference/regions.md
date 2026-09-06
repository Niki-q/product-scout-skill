# Supported regions: Cyprus (CY), Ukraine (UA), Moldova (MD)

This skill was built and tested against these three delivery regions. All
findings below are from live testing during development (dates noted per
marketplace), not documentation or assumption. Marketplace files
(`amazon.md`, `ebay.md`, `aliexpress.md`) point here for the region
mechanics; this file is the one place that tracks per-region status so it
doesn't drift out of sync across three files.

## Summary

| Marketplace | CY | UA | MD |
|---|---|---|---|
| Amazon | ✅ native, no action needed | ✅ via "Deliver to" dialog | ✅ via "Deliver to" dialog |
| eBay | ✅ via "Ship to" dialog (country present) | ✅ (country present) | ✅ (country present) |
| AliExpress | ✅ native, EUR | ✅ via header country switch, but **currency becomes UAH, not EUR** | ❌ **broken** — see below |

## Amazon — V (2026-09-06)

`amazon.com`'s own "Deliver to" dialog (header button → "or ship outside
the US" — confirmed present, ~200 countries) lists all three: **Cyprus**,
**Ukraine**, **Moldova, Republic of** (that's its exact label — match on
"Moldova", not an exact string). No separate country-domain (`.de`,
`.co.uk`) is needed for any of the three — the default `amazon.com`
storefront handles international delivery to all of them directly.

Confirmed live on one test product (an iPhone case, ASIN `B0DFH8SNK5`):
switching delivery to Cyprus, Ukraine, and Moldova in turn all returned the
**identical price, EUR 11.41** — Amazon's international-buyer pricing
doesn't obviously differentiate between these three at least for this
product. Don't generalize this to "always identical" without spot-checking
a given product — it wasn't tested across a wide product sample, and a
separate "Shipping & Import Charges" line (seen for Cyprus on a different
product in earlier testing this project) did not appear for this
particular marketplace-fulfilled offer, for any of the three countries —
that may be offer-specific (Amazon-fulfilled vs. third-party marketplace
seller), not region-specific. Confirm the shipping/import-charge line
per-listing rather than assuming its presence or absence.

The dialog's confirm button is labeled **"Done"**, not "Apply" — "Apply"
in that same dialog belongs to a separate US-zip-code input field, not the
country selector.

## eBay — V (country list) / U (post-selection currency effect)

`ebay.com`'s header has a **"Ship to"** button → opens "Set your shipping
location" → its own "Ship to: <country>" button opens a country picker
(`menuitemradio` list) that includes **Cyprus**, **Ukraine**, and
**Moldova, Republic of** — confirmed present in the live DOM, twice,
independently.

**Never pass location via URL query parameters** (`_stpos`, `LH_PrefLoc`)
— confirmed to trigger eBay's bot-check. Use the UI dialog above instead.

**What's still unverified:** the actual price/currency effect *after*
selecting a country and confirming ("Done") could not be confirmed this
session — every attempt (there were several, across different approaches)
to run a search immediately after changing the ship-to country hit eBay's
bot-check interstitial (`ebay.com/splashui/challenge`), even with a bare
keyword query and no extra parameters at all. This looks less like a
reaction to any specific action and more like eBay's bot detection
tightening up under sustained automated traffic in one session/profile —
treat eBay as the **most bot-check-prone of the three marketplaces** in an
automated session: space out requests, expect to hit the interstitial more
than the other two, and stop cleanly (never retry immediately) when it
happens per `browser.md` §2.

**Practical implication:** the country selector is real and does list all
three target regions, but a caller can't currently assume the resulting
search prices are in the buyer's local currency without a live check on
each run — flag prices as "shown in whatever currency the ship-to
selection last applied, unconfirmed this run" until someone gets a clean
verification through.

## AliExpress — V (2026-09-06)

Region switch via the header's country/flag icon (`[aria-label*="select a
country"]`) → panel with delivery country / language / currency → clicking
the current country opens a full country list → selecting a country
auto-sets language and currency → "Сохранить"/"Save" applies it (causes a
reload/navigation).

- **Cyprus**: native/default in this project's session — EUR, works
  cleanly, region tag `!sea!CY!` / `!ct!CY!` in `pdp_npi` (see
  `selectors.md`).
- **Ukraine**: works the same way as Cyprus mechanically, but **currency
  auto-sets to UAH, not EUR** — this is a real difference from CY, not a
  bug: don't assume a UA-region session's prices are in EUR. `pdp_npi`
  confirmed to carry `!sea!UA!`. Cookie `aep_usuc_f` shows `region=UA`,
  `site=glo` (not `tur` — confirms `site` is an unrelated geo-sharding
  field, not a region/country indicator; see `browser.md`). Search and
  product pages stay on `www.aliexpress.com` (no redirect to a country
  subdomain). A real product page showed genuine Ukrainian carriers (Nova
  Poshta, Meest) and delivery estimates with no obvious restriction.
- **Moldova: does not work — a confirmed, hard limitation, not a
  guess.** Selecting "Moldova" in the picker looks normal right up until
  "Save" is clicked — at which point AliExpress redirects to
  **`aliexpress.ru`** (`gatewayAdapt=glo2rus`; there is no
  Moldova-specific gateway). `aliexpress.ru` is a separate, Russia-market
  joint-venture site (built around RU/BY/KZ/AM) with **no Moldova option in
  its own country list at all** — the session that lands there gets
  silently reset to `region=RU`, `site=rus`, `c_tp=RUB`, an auto-filled
  city of "Moscow", and the product actually showed **"Not deliverable to
  Moscow."** A `pdp_npi` region tag for MD was never obtainable — no path
  reached a product page with `region=MD` at all.

  **Practical implication:** for an MD buyer, do not attempt AliExpress
  region selection — it silently degrades to a Russia-region session that
  has nothing to do with the buyer's actual location. Either fall back to
  the default/CY-localized session with an explicit caveat that its
  pricing/shipping may not reflect Moldova, or skip AliExpress for MD
  requests entirely and rely on Amazon/eBay (both of which work cleanly
  for MD) — prefer the latter unless the user explicitly accepts the
  caveat.
