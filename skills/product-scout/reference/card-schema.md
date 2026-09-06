# Product card schema

Every marketplace adapter (`amazon.md`, `ebay.md`, `aliexpress.md`) must
normalize its results into this one shape before they're compared or
presented. Fields marked "required" must be non-null on every card that
survives verification (see `browser.md`) — a card missing a required field
did not verify cleanly and should not be presented as a result.

```
{
  "marketplace":     "amazon" | "ebay" | "aliexpress",   // required
  "id":              string,   // required — native id (ASIN / eBay item id / AliExpress product id)
  "title":           string,   // required — read off the PRODUCT PAGE, not the search snippet
  "url":             string,   // required — canonical product-page URL, tracking params stripped
  "price": {
    "amount":        number,   // required
    "currency":      string    // required — ISO 4217, e.g. "EUR"
  },
  "original_price":  { "amount": number, "currency": string } | null,  // pre-discount list price, if any
  "rating":          number | null,   // 0-5
  "review_count":    number | null,   // drives ranking — see below
  "sold_count":      number | null,   // AliExpress/eBay expose this; Amazon usually doesn't
  "condition":       "new" | "used" | "refurbished" | null,  // mainly relevant on eBay
  "shipping": {
    "cost":          number | "free" | null,   // required attempt — see "Shipping cost is mandatory" below
    "currency":      string | null,
    "eta_days":      string | null,      // e.g. "11-27" — a range, not a false-precise single number
    "ships_to":      string              // region the estimate is for, e.g. "CY" — always state this explicitly, never assume the reader knows. See reference/regions.md for the three regions (CY/UA/MD) this skill actually supports and their per-marketplace status.
  },
  "total_price": {   // required — item price + shipping.cost, same currency; see below
    "amount":        number | null,   // null only if shipping.cost is null (genuinely unavailable)
    "currency":      string
  },
  "verified":        boolean,   // required — true only once the page-level check in browser.md passed
  "verified_at":     string     // required when verified=true — ISO 8601 timestamp
}
```

## Shipping cost is mandatory, not a nice-to-have

A card's `shipping.cost` must be **read from the candidate's own verified
product page** (not guessed, not left null by default) before the card is
presented — this isn't optional the way `rating`/`sold_count` genuinely can
be unavailable. Confirmed live why this matters: an Amazon listing at
**€17.21** showed **€15.04** in its own "Shipping & Import Charges to
Cyprus" line — shipping added **87% to the item price**. Presenting the
€17.21 alone as "the price" without that figure would have been actively
misleading, not just incomplete.

`shipping.cost` is only genuinely `null` when the product page itself
doesn't expose a number for the target region (rare, but happens) — that's
different from not having looked. Compute `total_price` = `price.amount +
shipping.cost` (0 when `shipping.cost` is `"free"`) whenever both are
known, and **rank/sort by `total_price`, not by `price.amount` alone**,
whenever the buyer has stated a budget or price is a comparison criterion
— a cheaper item with expensive shipping can easily lose to a pricier one
with free shipping once landed cost is compared. Always show `total_price`
alongside the item price when presenting a card, not price alone.

Country-level shipping estimates (not a specific city/postal code) are
sufficient for this skill's three supported regions (`regions.md`) — all
three (CY, UA, MD) are small enough that carrier shipping rates don't vary
within the country the way they might in, say, the continental US.
Confirmed for Amazon: its shipping/import-charge estimate is shown against
the country selected in "Deliver to", with no postal code entry involved.
Don't build postal-code/address-entry automation for these three regions —
that would be solving a problem this skill doesn't have.

## Ranking rule

Ported from the `jlave-dev/agent-skills` "Amazon Shopping" skill: **when two
candidates' `rating` values are within 0.2 of each other, the one with the
higher `review_count` (or `sold_count` where rating data is thin, as is
common on AliExpress) ranks first, not the one with the marginally higher
star rating.** A 4.9 rating on 12 reviews is not more reliable than a 4.7 on
4,000 — raw rating alone is noise at low sample sizes. Never present a
candidate whose `review_count` and `sold_count` are both null/zero as a top
pick without saying so explicitly.

## Never fabricate a field

If a real value can't be read from the actual page, the field is `null`,
not a guess and not the value from a similar listing. This schema exists so
a caller can distinguish "no shipping estimate available" from "free
shipping" — collapsing the two loses real information.
