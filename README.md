# CompScan backend

A real (not mock) scanner: pulls active Pokémon card listings from eBay, matches
each one to a catalog entry, and comps it against TCGPlayer market pricing —
sourced via pokemontcg.io, since TCGPlayer closed its API to new developers.

## 1. Get credentials

**eBay (free, self-serve, a few minutes):**
1. Create a developer account at https://developer.ebay.com
2. Go to **My Account → Application Keys**, create a keyset
3. Copy the **Client ID** and **Client Secret** into `.env`

You only need the client-credentials grant (no user login flow) — the Browse
API's search endpoint is public data.

**Catalog / TCGPlayer pricing (free tier, optional key for higher limits):**
1. Register at https://dev.pokemontcg.io
2. Put the key in `.env` as `POKEMONTCG_API_KEY` — works without one at low volume too

## 2. Install & configure

```
cd backend
npm install
cp .env.example .env
# edit .env with your credentials and search terms
```

## 3. Run

```
npm run scan     # one-off scan from the CLI, writes data/deals.json
npm start        # starts the server: serves the frontend + /api/deals + /api/scan
```

Open http://localhost:8787 — the existing frontend will detect the backend
automatically and switch from sample data to live results. The "Scan eBay"
button calls `POST /api/scan` for a fresh pull.

## How the pipeline works

```
eBay Browse API  →  title parser  →  catalog fuzzy match  →  profit calc
  (listings)         (condition,       (pokemontcg.io,        (fee-adjusted
                      lot size,          TCGPlayer market       margin)
                      grading)           price per card)
```

- **`ebayClient.js`** — mints an eBay app token and searches active listings for
  each configured query.
- **`titleParser.js`** — heuristically pulls condition, lot size, grading, and a
  cleaned search string out of a messy seller-written title.
- **`matcher.js`** — searches the catalog and scores candidates against the
  cleaned title using bigram similarity, discarding anything below a
  confidence threshold rather than guessing.
- **`profit.js`** — scales the NM market price down for worse condition,
  applies a lot-size haircut, subtracts TCGPlayer's resale fees, and compares
  to what you'd actually pay on eBay (price + shipping).
- **`scanner.js`** — runs the whole pipeline per query and writes
  `data/deals.json`.

## Known limitations (read before trusting the numbers)

- **Graded cards (PSA/BGS/CGC) are skipped entirely.** A raw TCGPlayer market
  price doesn't reflect what a PSA 10 sells for — comping them the same way
  would be actively misleading. Real grade-specific comps need a different
  data source (e.g. sold PSA 10 listings), which isn't wired in here.
- **Title matching is heuristic, not certain.** Every deal reports a
  `confidence` score (0–1) from the bigram match; low scores mean the parser
  had to guess. Consider filtering to confidence ≥ 0.6 in production.
- **Fee assumptions are approximate.** eBay and TCGPlayer fee schedules vary by
  category and account tier and change over time — check current rates before
  using this for real purchasing decisions:
  - eBay: https://www.ebay.com/help/selling/fees-credits-invoices/selling-fees
  - TCGPlayer: https://help.tcgplayer.com
- **Lot pricing is a rough haircut**, not a per-card breakdown — a 10-card lot
  listing doesn't tell you which 10 cards, so this can't price it precisely.
- **eBay category ID** in `ebayClient.js` (`183454`, Pokémon Individual Cards)
  should be double-checked against eBay's current category tree if results
  look off — categories occasionally get renumbered.
- **Rate limits**: eBay app tokens last ~2 hours (handled automatically);
  pokemontcg.io's free tier is modest, so large scans benefit from the API key.

## Suggested next steps

- Add confidence-based filtering/sorting in the UI (surface it, don't hide it)
- Swap in a real graded-card price source for PSA/BGS/CGC listings
- Move `data/deals.json` to a real database if you want history, not just the
  latest snapshot
- Run `npm run scan` on a schedule (cron, or a hosted scheduler) instead of
  only on-demand
