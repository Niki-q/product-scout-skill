---
name: product-scout
description: Find real, currently-listed products on Amazon, eBay, or AliExpress against a set of buying criteria (budget, use case, deal-breakers), with every result verified on its actual product page before being presented, and pricing/shipping estimates for the buyer's real region. Use when asked to find/compare/recommend specific purchasable products or product listings ("find me a...", "what's a good X under $Y", "search AliExpress/Amazon/eBay for...") — not for general shopping advice with no need for real current listings.
---

# product-scout

Cross-marketplace product research: Amazon, eBay, AliExpress. Every result
this skill presents has been verified against its own live product page —
a candidate that fails verification is dropped or clearly flagged, never
silently presented as a match.

## Why this exists, briefly

Built after two concrete failures: (1) parallel subagents sharing one
browser produced cards whose title/price/link belonged to *different*
products, and (2) a naive Amazon scrape once returned a sponsored card's
tracking-redirect link instead of the real product page. Every rule in
`reference/browser.md` and the marketplace-specific files exists to close
one of these gaps, confirmed by reproducing the actual failure live before
writing the fix — see each file's "confirmed live" notes rather than
taking any of this on faith.

## Pipeline (same shape for every marketplace)

1. **Gather criteria first.** Budget, the actual use case, and any
   deal-breakers (must ship to the buyer's region, size/color/material,
   condition, buying format, etc). Don't jump straight to searching on a
   bare product name — a technically-matching result that blows the stated
   budget or ignores a named deal-breaker isn't useful.
2. **Pick the marketplace(s).** If the user names one, use it. If not, and
   nothing rules the others out, prefer AliExpress for generic/commodity
   items shipping to a non-US buyer (usually cheaper, often free shipping),
   Amazon for branded items with fast/reliable regional fulfillment, eBay
   for used/refurbished or when the exact item is discontinued elsewhere.
   Say which you picked and why in one line.
3. **Search and extract candidates** — read `reference/browser.md` first
   (session discipline: no parallel tabs, CAPTCHA is a stop, extract
   compact structured data instead of dumping full page snapshots), then
   the relevant marketplace file (`reference/amazon.md`, `reference/ebay.md`,
   `reference/aliexpress.md`) for the actual extraction pattern —
   `reference/selectors.md` has the concrete, version-dated DOM patterns
   each of those files points at.
4. **Verify every candidate worth keeping** on its real product page before
   it counts as a result — this is not optional and not skippable under
   time pressure. See `browser.md` §5 and each marketplace file's own
   verification note for why.
5. **Normalize into the shared card shape** in `reference/card-schema.md`
   and **rank by review/sold count over raw rating** when ratings are
   close — a high star rating on a tiny sample size is not more reliable
   than a slightly lower one on a large sample.
6. **Present up to the number of cards asked for** (default 5 if unstated),
   each with: title, price + currency, rating/review or sold count if
   available, a real shipping estimate for the buyer's actual region (or an
   explicit note that the session wasn't localized and the figure may not
   apply), and the verified canonical product URL. Never present an
   unverified candidate as if it were a verified result.

## Region and shipping

Every marketplace file has its own region caveats (`browser.md` §4 covers
the general principle) — a price or shipping figure is only meaningful if
you've confirmed which region it was rendered for. Default to the buyer's
stated region; if none was stated, ask rather than silently assuming US
pricing applies.

## Scope

Amazon, eBay, AliExpress only, v1. Temu and regional marketplaces
(Rozetka, Allegro, Otto, Kaufland) are explicitly out of scope — no
reference implementation exists for them anywhere, adapting one of the
above patterns to them without live verification first would be guessing,
not scouting.
