# Amazon pipeline

Ported from `jlave-dev/agent-skills`' "Amazon Shopping" skill: the pipeline
discipline (criteria first, verify every result, rank by review count) is
the same; the actual DOM extraction below was independently re-verified
live against `amazon.com` search results (see `selectors.md`).

## 1. Gather criteria before searching

Before running any search, know: budget ceiling, the actual use case (not
just a product category), and any deal-breakers (must ship to the buyer's
region, must be a specific size/color/material, etc). Don't skip this and
go straight to "search for X" — a card that's technically the right
product but blows the budget or ignores a stated deal-breaker isn't a
useful result.

## 2. Search and extract candidates

Navigate to `amazon.com/s?k=<query>` (or the relevant country storefront —
`amazon.co.uk`, `amazon.de`, etc. — if the buyer's region has one and it
matters for pricing/availability). Extract with the `div[data-asin]`
pattern in `selectors.md`, deduping by ASIN. This gives *candidates*, not
verified cards.

## 3. Verify every candidate on its real product page — mandatory, no exceptions

For each candidate worth carrying forward, load `amazon.com/dp/<ASIN>` and
confirm the page title contains the candidate's title and the displayed
price matches. This step exists specifically because of a confirmed,
reproducible bug: a **sponsored** search-result card's only in-card link is
a `/sspa/click?...` tracking redirect, not the product page — scraping that
link (instead of building the URL from the ASIN) is exactly how a result
ends up pointing at the wrong listing. Never skip this step for a sponsored
card on the theory that "the ASIN is right there, it must be fine" — the
ASIN is right there, but the *scraped link* is what fails, which is why
`selectors.md` says to derive the URL from the ASIN and verify by loading
that derived URL, not by trusting anything scraped in-card.

## 4. Rank by review count, not raw rating

Apply the rule in `card-schema.md`: when ratings sit within ~0.2 of each
other, the higher review-count candidate ranks first. A 4.8★ sponsored
listing with an unknown/low review count is not automatically better than
a 4.6★ organic listing with 19,000+ ratings — in a live test run, the
latter was the actual best pick despite the lower star rating.

## 5. Shipping / region

For CY/UA/MD specifically, see `regions.md` — no separate storefront
switch is needed: the default `amazon.com` "Deliver to" dialog (header
button → "or ship outside the US") lists all three directly and was
confirmed live to return identical pricing across them on a test product.
Amazon's own checkout (or the "Shipping & Import Charges" line, where
shown) has the real, final shipping/import estimate for the buyer's
delivery address — more reliable than guessing from the product page
alone, and worth checking per-listing since its presence wasn't consistent
across every offer type in testing (see `regions.md`'s caveat on this).
For any region outside CY/UA/MD, either use the same "Deliver to" dialog
if it lists that country, or note plainly that the shown price is for the
default region and may not reflect the buyer's actual landed cost.
