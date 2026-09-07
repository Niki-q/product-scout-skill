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
   each of those files points at. **For eBay specifically, check first
   whether the `ebay_search_items`/`ebay_get_item` MCP tools are
   available** (the `ebay-browse-mcp` server) — if so, use those instead of
   Playwright per `ebay.md` §0; that path also collapses steps 4 and 5
   below into the one `ebay_get_item` call, since it returns authoritative,
   already-verified data with real shipping included. Playwright stays the
   fallback for eBay when that server isn't configured, and remains the
   only path for Amazon/AliExpress.
4. **Verify every candidate worth keeping** on its real product page before
   it counts as a result — this is not optional and not skippable under
   time pressure. See `browser.md` §5 and each marketplace file's own
   verification note for why. **Read the actual shipping cost off that
   same product page while you're there** — every marketplace file has its
   own confirmed pattern for this (`amazon.md` §5, `ebay.md` §4,
   `aliexpress.md` §6). This is mandatory, not a nice-to-have: a live test
   found an Amazon item at €17.21 with **€15.04** shipping to Cyprus —
   skipping this would have understated the real cost by 87%.
5. **Normalize into the shared card shape** in `reference/card-schema.md`,
   compute `total_price` (item + shipping) for every card, and **rank by
   `total_price`, not `price` alone**, whenever budget is a criterion — see
   `card-schema.md`'s worked example of why item price alone can flip the
   actual best deal. Separately, **rank by review/sold count over raw
   rating** when ratings are close — a high star rating on a tiny sample
   size is not more reliable than a slightly lower one on a large sample.
6. **Present up to the number of cards asked for** (default 5 if unstated),
   each with: title, item price, **shipping cost and the resulting total
   price** (or an explicit note that shipping was genuinely unavailable —
   never just the item price silently standing in for the full cost),
   rating/review or sold count if available, which region the shipping
   figure is for, and the verified canonical product URL. Never present an
   unverified candidate as if it were a verified result.

## Region and shipping

Three regions are supported end-to-end and were live-tested during
development: **Cyprus (CY)**, **Ukraine (UA)**, **Moldova (MD)** — see
`reference/regions.md` for the exact per-marketplace mechanics and status.
A price or shipping figure is only meaningful if you've confirmed which
region it was rendered for (`browser.md` §4 covers the general principle).
Default to the buyer's stated region; if none was stated, ask rather than
silently assuming US pricing applies. For any other region, the same
mechanics likely apply (each marketplace's region-switch UI is generic),
but nothing beyond CY/UA/MD has been live-tested — say so if asked about a
region outside these three.

**One hard limitation to know up front:** AliExpress does not work for
Moldova — selecting Moldova in its region picker silently redirects to a
separate Russia-market site with no Moldova option at all, resetting the
session to a Russian region/currency. For an MD buyer, skip AliExpress
(Amazon and eBay both work cleanly for MD) unless the user explicitly
accepts an unlocalized/CY-approximate AliExpress result with that caveat
stated plainly. See `regions.md` for the full finding.

## Scope

Amazon, eBay, AliExpress only, v1, for CY/UA/MD delivery regions. Temu and
regional marketplaces (Rozetka, Allegro, Otto, Kaufland) are explicitly out
of scope — no reference implementation exists for them anywhere, adapting
one of the above patterns to them without live verification first would be
guessing, not scouting.
