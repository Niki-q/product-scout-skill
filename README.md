# product-scout

A Claude Code skill for personal product research across Amazon, eBay, and
AliExpress — find real, currently-listed products against a set of buying
criteria, with every result verified on its actual product page, and
pricing/shipping estimates for the buyer's real region (built and tested
against EU/Cyprus).

## Status: v1 implemented

The skill (`skills/product-scout/SKILL.md` + its `reference/` files) is
written and has been live-tested against Amazon, eBay, and AliExpress. See
the roadmap below for what's done and what's explicitly out of scope.

## Install

Copy (or symlink) `skills/product-scout/` into your Claude Code skills
directory, e.g.:

```sh
git clone https://github.com/Niki-q/product-scout-skill
cp -r product-scout-skill/skills/product-scout ~/.claude/skills/product-scout
```

The skill is entirely self-contained — `SKILL.md` plus its `reference/`
files, no external dependencies, no API keys. It only needs a browser
automation tool (e.g. the Playwright MCP server) available to the agent.

## Why

Existing community skills solve half the problem each:

- **Amazon-only skills** (e.g. `jlave-dev/agent-skills` "Amazon Shopping")
  have a solid pipeline — gather criteria, search, extract, and crucially
  **verify every result on its real product page** before presenting it,
  ranking by review count over raw rating. But Amazon-only.
- **AliExpress-focused skills** (e.g. `danielrosehill/Aliexpress-Israel-Skills`)
  have good discipline — a table of DOM selectors with a verified/unverified
  status, a strict "CAPTCHA means stop" rule, and a sequential (never
  parallel-tab) browser session. But wired to Israeli VAT/ILS, not EU/CY.
- Nothing covers eBay well for personal shopping (existing eBay skills are
  either seller-tooling or paid API wrappers).
- Nothing covers Rozetka/Allegro/Otto/Kaufland at all.

`product-scout` borrows the verification pipeline from the first, the
selector-discipline and no-parallel-tabs rule from the second, targets
three marketplaces (Amazon, eBay, AliExpress) with region-aware pricing and
shipping, and returns one consistent product-card shape across all three.

## Example run

Query: *"tongue scraper, stainless steel, under $15"* — run live against
all three marketplaces during development:

- **Amazon**: best pick by the review-count-over-rating rule was a 4.6★
  listing with 19,235 ratings at $4.30 — ranked above several sponsored
  4.7-4.8★ cards with much smaller (or untracked) review counts.
- **eBay**: cards extracted with price, sold count, and a shipping caveat —
  one candidate's search-card price ($5.80) turned out on its real product
  page to be a converted estimate of a native GBP 4.29 listing that "may
  not ship to United States" at all. Verification caught it before it
  would've been presented as a clean US-shippable result.
- **AliExpress**: 5+ candidates in the €1.43-€12.02 range, region-confirmed
  (`CY` tag in the result data) and rating/sold-count filtered.

A second live run (*"windproof automatic umbrella, compact"*, AliExpress
only) returned a top pick at €23.61 with 61,515 sold — filtering out lower
-trust cheap listings the same way.

## Scope (v1)

- Amazon, eBay, AliExpress.
- Rozetka / Allegro / Otto / Kaufland / Temu are explicitly out of scope for
  v1 — no reference implementation exists anywhere to build from, so these
  would be written from scratch rather than adapted. Tracked as backlog
  ideas in the project's own tracker, not in this repo.

## What's verified live vs. documented-only

See each `reference/*.md` file's own notes for exactly what was confirmed
against a real page during development versus carried over from a
community skill's description without independent testing — most notably,
**eBay search-URL filter parameters (price range, condition, buying
format, location) could not be verified**: two separate live attempts both
triggered eBay's bot-check interstitial. The pipeline works around this by
filtering extracted candidates client-side instead of relying on
server-side query parameters — see `reference/ebay.md` §2.

## Roadmap

- [x] Common product-card schema (title, price, currency, rating, review
      count, sold count, shipping cost/estimate, canonical URL) shared
      across all three marketplaces.
- [x] `reference/browser.md` — session discipline: no parallel tabs, CAPTCHA
      is a stop condition, compact indexed-text page representation instead
      of full accessibility snapshots.
- [x] `reference/selectors.md` — per-marketplace DOM selectors / URL query
      patterns, each tracked with a verified (`V`) / unverified (`U`) status.
- [x] Amazon support, ported from the verify-before-present pipeline above.
- [x] eBay support (built from scratch, no reference implementation existed).
- [x] AliExpress support, adapted for EU/CY pricing and shipping instead of
      Israeli tax rules.
- [x] Live test pass (tongue scraper across all three marketplaces; a
      windproof umbrella query on AliExpress) — see "Example run" above.
- [ ] Regional marketplaces (Rozetka/Allegro/Otto/Kaufland) and Temu — no
      reference implementation to build from; deferred, see the project's
      own backlog.

## License

MIT — see [LICENSE](LICENSE).
