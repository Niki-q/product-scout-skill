# Browser session discipline

These rules exist because of a concrete failure: running several marketplace
searches through parallel subagents sharing one Playwright browser produced
cards whose title/price/link belonged to *different* products — one agent's
navigation stomped another's mid-extraction. Every rule below closes that
gap or a variant of it.

## 1. One search at a time, never parallel tabs against the same session

If several searches are needed (multiple marketplaces, multiple queries),
run them **sequentially** in the same conversation, not as concurrent
subagents sharing a browser. If you must parallelize across marketplaces,
give each its own isolated browser context/profile — never let two
in-flight searches share one login/cookie session, because a shared session
means a shared "current page," and whoever navigates last wins.

## 2. CAPTCHA means stop, not solve

Any interstitial that looks like a bot check (image challenge, "verify
you're human", an unusual redirect to a security page) is a hard stop for
that search. Report it plainly to the user — "hit a CAPTCHA on <site>,
couldn't continue" — and do not attempt to solve it, retry through a proxy,
or route around it. This applies even under time pressure or an autonomy
instruction — a CAPTCHA is the platform telling you to stop, and continuing
anyway can get the account flagged.

## 3. Don't dump full snapshots into context — extract structured data instead

`browser_snapshot` on a marketplace search-results page routinely produces
100-300KB+ of accessibility tree. Reading that whole file (or worse,
several, one per query) burns context fast and doesn't scale past one or
two searches. Default to `browser_evaluate` with a JS function that walks
the DOM and returns a **compact JSON array** — id, title, price, rating,
sold count, URL per candidate — never the raw tree. Treat a full snapshot
read as a fallback for when the compact extraction script needs to be
written or debugged, not the normal per-search path.

**A verified technique, not a guess:** on AliExpress, per-item price data is
more reliably read from the product link's own query string than from
visible DOM text nodes — visible price text sits next to rating/sold-count
text and naive regexes over `textContent` can concatenate them into
garbage (`"€41,254.4800"` instead of `€41.25`). The link carries a `pdp_npi`
param encoding `dis!<CUR>!<orig>!<sale>` (sometimes with a repeated
currency-and-space prefix on each number, sometimes not — match both). This
pattern was confirmed live against real AliExpress search results:

```js
() => {
  const seen = new Set();
  const results = [];
  document.querySelectorAll('a[href*="/item/"]').forEach(a => {
    const rawHref = a.getAttribute('href');
    const m = rawHref.match(/item\/(\d+)\.html/);
    if (!m) return;
    const id = m[1];
    if (seen.has(id)) return;
    const h = a.querySelector('h1,h2,h3');
    const title = h ? h.textContent.trim() : '';
    if (!title) return;
    seen.add(id);
    let decoded = '';
    try { decoded = decodeURIComponent(rawHref); } catch(e) { decoded = rawHref; }
    const pm = decoded.match(/dis!([A-Z]{3})!(?:[A-Z]{3}\s)?([\d.]+)!(?:[A-Z]{3}\s)?([\d.]+)/);
    const regionM = decoded.match(/!(?:sea|ct)!([A-Z]{2})!/);
    results.push({
      id, title: title.slice(0, 140),
      currency: pm ? pm[1] : null,
      orig: pm ? pm[2] : null,
      sale: pm ? pm[3] : null,
      region: regionM ? regionM[1] : null,
      href: 'https://www.aliexpress.com/item/' + id + '.html'
    });
  });
  return JSON.stringify(results);
}
```

Amazon and eBay don't expose price this way (see `amazon.md` / `ebay.md` for
their own patterns) — this technique is AliExpress-specific, not a general
rule to try blindly on every marketplace.

## 4. Region and currency must match the target before trusting a price

A search result's price/shipping numbers are only meaningful for the
region they were rendered for. Before trusting a card's `price.currency`
and `shipping.ships_to`, confirm the browser session is actually
localized to the target region (e.g. currency shown as EUR, a "Deliver to
<region>" indicator, or region-tagged tracking params in result URLs like
AliExpress's `!sea!CY!` on organic results, `!ct!CY!` on sponsored ones —
see `selectors.md`) — a session showing USD prices with no regional
signal is not localized, and its shipping estimates are not usable for a
different-region buyer. If the session isn't localized, either switch it
through the site's own region/currency picker, or say plainly that the
listed prices may not reflect the buyer's actual region before presenting
them.

## 5. Verify every card on its real product page before presenting it

Search-results extraction (rule 3) gets you candidates, not verified cards.
Before a card is marked `verified: true` per `card-schema.md`, load its
actual product-page URL and confirm the title and price displayed there
match what the search page claimed. This is the single most important rule
in this file — it's what prevents the exact mismatch bug that motivated
rule 1. A card that hasn't been through this check is a candidate, not a
result, and must not be presented as one.

## 6. Authenticated sessions — local cookie files, never in the repo

Amazon and AliExpress both work better with the buyer's real, logged-in
session than anonymous browsing — confirmed live: an authenticated Amazon
session's saved address carries a real city/postal code (`amazon.md` §6)
instead of just a country, and an authenticated AliExpress session keeps
region localization working identically to anonymous (`aliexpress.md`).

**Convention: check for a local cookie file before asking the user to
paste cookies into the conversation.** Per marketplace, a plain JSON array
(same shape browser cookie-export extensions produce — `name`, `value`,
`domain`, `path`, `secure`, `httpOnly`, `sameSite`, `expirationDate`) at:

- `~/.product-scout/cookies/amazon.json`
- `~/.product-scout/cookies/aliexpress.json`

(on Windows, `~` is `C:\Users\<user>\`). At the start of a session, if the
file for the marketplace you're about to use exists, read it and apply it
with `context.addCookies()` before navigating — don't wait for the user to
paste cookies again just because they aren't already in this
conversation's context. If the file doesn't exist, fall back to anonymous
browsing and say so explicitly (`browser.md` §4 still applies — an
anonymous session's region still needs confirming).

**These files live outside this repo and must never be committed** — they
are real, sensitive session credentials, not test fixtures. If a cookie
file's cookies turn out to be expired (login check fails, e.g. no account
greeting where one's expected), say so and ask the user for a fresh
export — don't silently keep retrying, and don't treat an expired file as
evidence the loading mechanism itself is broken.
