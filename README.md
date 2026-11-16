[![make.fun · token-radar-01](https://make.fun/badge/token-radar-01.svg)](https://make.fun/bounty/token-radar-01?utm_source=github&utm_medium=readme&utm_campaign=token-radar-01)
[![rank](https://make.fun/badge/token-radar-01/su3679204/token-launch-radar/rank.svg)](https://make.fun/bounty/token-radar-01/leaderboard?utm_source=github&utm_medium=readme&utm_campaign=token-radar-01)
[![reward](https://make.fun/badge/token-radar-01/su3679204/token-launch-radar/reward.svg)](https://make.fun/bounty/token-radar-01?utm_source=github&utm_medium=readme&utm_campaign=token-radar-01)

# Token Radar

A token launch radar: it reads a sealed snapshot of every protocol DefiLlama tracks, works
out which ones still record no token, and ranks them by what their launch would be worth —
each with the steps to farm it. An entry for make.fun's
**Build the best token launch radar** bounty (`token-radar-01`).

## What it does

`main.ts` (Node 22, standard library only) reads `/input/snapshot.json`, scores every
pre-token protocol with the radar-2 model in `lib/model.ts`, and writes:

- `board.json` — the ranked board: the whole 40-protocol watchlist plus every other
  rankable pre-token protocol the watchlist's own rule keeps (198 rows on this snapshot),
  each with an implied value and a six-step farm plan.
- `data.js` — the same board in the compact form the dashboard renders.
- `dashboard/` — the static page (no network, no framework).
- `screenshots/` — the board, a protocol's detail view, and a filtered view.
- `dossier.json` — built with `buildDossier` from `vendor/makefun-board.ts`.

## The model (radar-2)

Implied value follows only from the snapshot's own figures — and, unlike a flat multiple,
it is calibrated on the snapshot itself:

```
implied = round( tvl_usd x category_multiple
                 x (1 + clamp(change_7d, -60, 120) / 100 x 0.5)
                 x (0.6 + min(age_days, 1095) / 1095 x 0.4)
                 x (1 + 0.05 x min(chains - 1, 3)) )
```

- **category_multiple** — the median market-cap / TVL of the live-token protocols in the
  same category, shrunk toward the global median with a prior weight of 3, then clamped to
  `[0.05, 1.5]`. Each live ratio is winsorised to `[0.01, 5]` first, so one tiny-TVL
  outlier cannot set a category. A category with no live comparable uses the global median.
  The dashboard shows the whole calibration table: category, live comparables, median, and
  the multiple applied.
- **growth** — the protocol's seven-day TVL change, clamped to `[-60, 120]` per cent.
- **maturity** — how long it has run with no token on record (age since first listing;
  365 days when unlisted).
- **breadth** — one more chain is more surface: +5% per extra chain, up to three.

No protocol name is ever special-cased: the same function scores every row. A protocol is
pre-token when its record carries no token (no market cap, no CoinGecko id, no symbol), and
the board leaves out wrapped, staked and bridged products of tokens that already trade, by
the watchlist's own rule — WBTC is Bitcoin on Ethereum, not a protocol waiting on a launch
of its own. Live-token protocols are set aside with their market caps shown, and they are
also the comparables the model is fitted on.

## Farm plans

Each row carries six steps, derived from the protocol's own record (category, chains, TVL,
age): a fresh wallet, the category's main action with its capital and gas, a weekly
qualifying action specific to that category, the hold window, the claim check, and the farm
log. Every step states what it costs and how long it takes.

## Run it

```sh
mkdir -p out && MAKEFUN_OUTPUT_DIR=out node main.ts --input ../input/snapshot.json
```

Node 22.18 or later. No install, no network: the program reads the snapshot, writes the
board, and copies `dashboard/` and `screenshots/` into the output directory.

## Layout

```
main.ts                  the entrypoint: snapshot in, board + dashboard + dossier out
lib/model.ts             radar-2: calibration, implied value, board, farm plans
docs/MODEL.md            the model, worked: formula, calibration, farm plans
vendor/makefun-board.ts  make.fun's dossier library, as shipped in the kit
dashboard/               index.html, app.css, app.js — the static page, no network
screenshots/             1-board.png, 2-detail.png, 3-filtered.png
```

## License

MIT. See `LICENSE`.

---

Built for the make.fun bounty — if you want to build entries like this one, start here:
https://make.fun/r/a866kr6r
