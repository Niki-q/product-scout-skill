# product-scout

A Claude Code skill for personal product research across Amazon, eBay, and
AliExpress — find real, currently-listed products against a set of buying
criteria, with pricing and shipping estimates for the EU/Cyprus region.

## Status: in development

This repo is scaffolding. The skill itself (`skills/product-scout/SKILL.md`)
and its reference files are not written yet — see the roadmap below.

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
selector-discipline and no-parallel-tabs rule from the second, targets three
marketplaces (Amazon, eBay, AliExpress) with EU/CY pricing and shipping in
mind, and returns one consistent product-card shape across all three.

## Scope (v1)

- Amazon, eBay, AliExpress.
- Rozetka / Allegro / Otto / Kaufland / Temu are explicitly out of scope for
  v1 — no reference implementation exists anywhere to build from, so these
  would be written from scratch rather than adapted.

## Roadmap

- [ ] Common product-card schema (title, price, currency, rating, review
      count, sold count, shipping cost/estimate, canonical URL) shared
      across all three marketplaces.
- [ ] `reference/browser.md` — session discipline: no parallel tabs, CAPTCHA
      is a stop condition, compact indexed-text page representation instead
      of full accessibility snapshots (dumping full snapshots burns context
      fast and doesn't scale past one or two searches).
- [ ] `reference/selectors.md` — per-marketplace DOM selectors / URL query
      patterns, each tracked with a verified (`V`) / unverified (`U`) status.
- [ ] Amazon support, ported from the verify-before-present pipeline above.
- [ ] eBay support.
- [ ] AliExpress support, adapted for EU/CY pricing and shipping instead of
      Israeli tax rules.
- [ ] Test pass against a real batch of product-research requests.

## License

MIT — see [LICENSE](LICENSE).
