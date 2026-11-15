# The radar-2 model

This note documents the model `lib/model.ts` runs, so a reader can check every number
the dashboard shows without reading the code first. Nothing here is a second source of
truth: the code is the model, and this note describes it. If the two ever disagree, the
code wins and this note is stale.

## The question

A protocol with no token on record may issue one some day. The board asks what that
token would be worth if the protocol's record stayed as it is — its TVL, its recent
growth, how long it has run with no token, and how many chains it runs on. It does not
predict whether or when a token is issued. Every implied value is this radar's own
claim, never make.fun's and never DefiLlama's.

## Who is ranked

A protocol is ranked when it is **pre-token** — its snapshot entry records no market
cap, no CoinGecko id and no symbol — and when the watchlist's own rule keeps it: not a
wrapped, staked or bridged product of a token that already trades, not an exchange, and
not a versioned pool of another protocol. The rule is applied to the whole board, not
just the floor, so WBTC, Binance staked ETH and the canonical bridges are out: each is
a product of a token that already trades, not a protocol waiting on a launch of its own.

The board carries the whole 40-protocol watchlist (every protocol at or above the
$50,000,000 floor that the rule keeps) plus every other rankable pre-token protocol,
above or below the floor. On this snapshot that is 198 rows. The snapshot's live-token
protocols are set aside with their market caps shown, and they do a second job: they are
the comparables the model is calibrated on.

## The formula

```
implied = round( tvl_usd x category_multiple
                 x (1 + clamp(change_7d, -60, 120) / 100 x 0.5)
                 x (0.6 + min(age_days, 1095) / 1095 x 0.4)
                 x (1 + 0.05 x min(chains - 1, 3)) )
```

Each factor comes from the snapshot's own figures:

- **TVL** (`tvl_usd`) is the protocol's size. A bigger protocol's token starts from a
  bigger base, and the multiple below converts size into a token value.
- **Category multiple** is the only fitted part, and it is fitted on the snapshot
  itself. For each category, take every live-token protocol in that category, compute
  market cap / TVL, winsorise each ratio to [0.01, 5], and take the median. Then shrink
  that median toward the global median (the median over all 119 live comparables,
  0.246x on this snapshot) with a prior weight of 3:

  ```
  multiple = (n x category_median + 3 x global_median) / (n + 3)
  ```

  then clamp it to [0.05, 1.5]. A category with many live comparables keeps its own
  median; a category with one or two is pulled most of the way to the global median;
  a category with none uses the global median outright. No multiple is a hand-picked
  constant, and no protocol name ever enters the calculation.
- **Growth** is the protocol's seven-day TVL change in per cent, clamped to [-60, 120]
  so one violent week cannot set the ranking. A protocol growing 20% in a week gets a
  1.10x growth multiplier; one shrinking 20% gets 0.90x.
- **Maturity** is how long the protocol has run with no token on record, measured as
  the days since DefiLlama first listed it. The multiplier runs from 0.6x for a brand
  new listing to 1.0x at three years. A protocol that has run for years with no token
  has had longer to build the user base a launch would price; a protocol listed last
  month has not. When the snapshot reports no listing date, the model uses 365 days.
- **Breadth** is the number of chains the protocol's TVL sits on: +5% per extra chain,
  up to three extra chains (1.15x at most). More chains is more surface for a launch.

## Worked example: Steakhouse Financial, rank 1

Steakhouse Financial records no token. On this snapshot it holds $2.75B of TVL, fell
2.19% over seven days, was first listed about 1.5 years ago, and runs on 11 chains.

- Category multiple: Risk Curators has 1 live comparable, so its median is shrunk
  hard toward the global median: (1 x 2.97 + 3 x 0.246) / 4 = 0.93x (0.9282 at four
  decimals, as the board stores it).
- Growth multiplier: 1 + (-2.19 / 100 x 0.5) = 0.989x.
- Maturity multiplier: 0.6 + 543 / 1095 x 0.4 = 0.798x.
- Breadth multiplier: 1 + 0.05 x min(11 - 1, 3) = 1.15x.
- Implied: the model multiplies at full precision, which gives $2,321,173,844. The
  factors above are rounded for display, so multiplying them by hand lands within a
  fraction of a percent of that number.

The dashboard's detail view shows this breakdown for every row, and the page
recomputes each row from the snapshot figures and checks the number `board.json`
carries, so the screenshots and the code agree.

## The calibration table

The dashboard shows the full calibration: for every category, how many live-token
protocols set its multiple, their median market-cap / TVL, and the multiple actually
applied. Categories with thin evidence show it plainly — a multiple from one
comparable is labelled with its `n`, not hidden behind a round number.

## The farm plans

Each row carries six steps, derived from the protocol's own record (category, chains,
TVL, age) through the playbook table in `lib/model.ts` (mirrored in
`dashboard/app.js`, so the page regenerates exactly the plan `board.json` carries):

1. A fresh wallet that holds only the funds being farmed, seed phrase offline.
2. The category's main action, with its capital and gas: $250 of capital for a
   protocol over $5B of TVL, $150 over $1B, $100 over $100M, $50 below that.
3. The category's weekly qualifying action, about $2 of gas a week, 10 minutes a week.
4. The hold window: 60 days for a protocol under 6 months old, 45 days under a year,
   30 days above that, so the activity lands in more than one snapshot.
5. The claim check: the protocol's own points or rewards page, only from the link it
   publishes, never a paid claim.
6. The farm log: wallet, dates, every transaction, so the position can be repeated
   or unwound.

Every step states what it costs and how long it takes. The steps are a plan for
farming a protocol that may issue a token, not a promise that it will.

## What the model does not do

- It does not use any figure outside the snapshot: no prices, no social data, no
  points program, no rumour.
- It does not rank wrapped, staked or bridged products, exchanges, or versioned
  pools, for the reason above.
- It does not say a protocol has no token. It says the snapshot records no token for
  it. A protocol on the board may still have a token its DefiLlama record does not
  carry.
- It does not predict a launch date, an airdrop, or an allocation. A high implied
  value means the protocol is big, growing, seasoned and broad — the kind of protocol
  whose token, if it ever launched, would start from a large base. That is all it
  means.
