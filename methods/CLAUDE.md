# Guide for AI tools: rebuild and audit History of Forks

You are looking at the data behind [History of Forks](https://dave-rodt.github.io/fork-history/),
a four-chapter study of blockchain forks. Your job, if a user points you here, is one or both
of:

1. **Rebuild** the headline figures from the files in `data/` and confirm they match the
   pages.
2. **Audit** the data and the choices behind it: find errors, weak assumptions, missing
   cases and claims the data does not support.

Read [`METHODOLOGY.md`](METHODOLOGY.md) first. It defines every column, the cutoffs, and the
classification rules, and it lists the choices the author already considers debatable.

## Ground rules

- Treat the pages as claims and the data as evidence. Where they disagree, report it.
- Cite rows, dates and columns when you report a finding, so it can be checked.
- Separate three kinds of finding: an **error** (the data does not support the number), a
  **choice** (defensible, but another choice gives a different answer: say what, and how much
  it moves), and a **gap** (something the data cannot answer).
- The scripts and the multi-GB raw crawl are not published. If a finding needs raw blocks,
  say so and name the public API call that would settle it (see the end of
  `METHODOLOGY.md`).
- Chain names: `sha256` = BTC, `blake2b` = XBT. Neither is assumed to be "Bitcoin".

## Files

| File | What it is |
|---|---|
| `data/chapter4/spam_breakdown_daily.csv` | Per chain, UTC day and transaction category: counts, vbytes, fees, value moved, the part shared with the other chain, and the part BIP-110 would reject (BTC only). The source of most Chapter 4 numbers. |
| `data/chapter4/spam_breakdown_blocks.csv` | Blocks per chain per day. |
| `data/chapter4/bip110_daily.json` | BIP-110 check on BTC per day: rejected vbytes and txs, day totals, rejected vbytes by category, transactions caught per rule, blocks checked. |
| `data/chapter4/daily_usd_close.csv` | Daily USD close used to price value and fees (empty = no market price yet). |
| `data/chapter4/xbt_btc_price_hourly.json` | Hourly closes: `t` Unix hour, `b2` XBT, `btc` BTC, in USD. |
| `data/chapter4/node_counts_2026-11-02.csv` | Node groups from our crawl and from Luke Dashjr's estimate. |
| `data/chapter1/*.json` | BCH vs. BTC monthly series (CoinMetrics, Blockchair). |
| `data/chapter2/*.json` | ETH vs. ETC monthly series and replayed-transaction counts. |
| `data/chapter3/*.json` | BTG vs. BTC monthly series. |

## Rebuild: Chapter 4 checks and expected results

Load `spam_breakdown_daily.csv` with pandas or the csv module. Spam = every `category` except
`coinbase` and `other`. All sums are over the whole file unless a date filter is given.

| Figure on the page | How to compute it | Expected |
|---|---|---|
| BTC transactions, block space | sum `txs`; sum `vbytes` / 1e6 for `chain = sha256` | 62,757,315 txs; 13,236.4 MvB |
| BTC spam share of block space | spam `vbytes` / all `vbytes`, sha256 | 44.46% |
| Runes, inscriptions on BTC | category `vbytes` / all `vbytes` | 39.39%, 2.89% |
| XBT spam share; BPUB share | same for `blake2b` | 22.24%; 18.19% |
| Shared transactions | sum `replayed_txs` (equal on both chains) | 4,485,982 |
| Share of XBT txs also on BTC | blake2b `replayed_txs` / `txs` | 98.0% |
| Share of BTC txs also on XBT | sha256 `replayed_txs` / `txs` | 7.1% |
| BIP-110 rejects on BTC | sum `bip110_vbytes` / `vbytes`; `bip110_txs` / `txs` | 3.01%; 1.17% (397.9 MvB; 736,388 txs) |
| Share of BTC spam BIP-110 stops | spam rows: `bip110_vbytes` / `vbytes` | 6.73% |
| BTC value moved, fees | sum `value_sat`, `fees_sat` / 1e8 | 84.79M BTC; 264.41 BTC |
| BTC value and fees in USD | each day's sats / 1e8 × that day's BTC close | $6.67 trillion; $20.71 million |
| Runes' share of BTC fees | rune `fees_sat` / all `fees_sat` | 9.62% |
| BIP-110 share of value, fees | `bip110_value_sat`, `bip110_fees_sat` over totals | 0.03% (29,234 BTC); 1.5% (3.94 BTC, ≈ $311,000) |
| XBT value, fees from Aug 30 | blake2b rows with `date >= 2026-08-30` | 11.74M XBT; 39.60 XBT |
| XBT value, fees in USD | same, × XBT daily close (prices start Sep 1) | ≈ $4.0 billion; ≈ $12,000 |
| XBT shared share of value, fees; BPUB share of fees | from Aug 30 | 92%; 89%; 7.9% |
| XBT peak price as share of BTC | max of `b2 / btc` in the hourly file | 2.09% (Sep 5, $1,670) |
| Nodes following XBT | XBT groups / total, per source | 5.1% (crawl); 5.2% (Luke) |

If any of these does not reproduce, that is a finding.

## Audit: where to push

These are the places a careful reviewer should press. For each, say whether the choice
changes a conclusion, and by how much.

1. **Spam definition.** All OP_RETURN outputs count, so all Runes. Recompute BTC's "spam"
   without Runes and compare with the BIP-110 figure.
2. **Classification order.** First match wins. Check whether reordering (e.g. Runes before
   large OP_RETURN) moves any category share by more than a rounding error.
3. **BPUB on BTC.** Full BPUB detection ran only on XBT-only transactions; BTC relies on
   mempool's fake-pubkey flag. Ask what BTC's BPUB share would be with the same rules, and
   whether that matters for any claim.
4. **Shared = same txid.** Look at days where `replayed_txs` jumps or drops. Consider
   conflicting spends and replays confirmed after the cutoff.
5. **Value moved includes change.** Test whether any conclusion depends on value rather than
   fees or counts.
6. **BIP-110 approximations.** Script types are inferred from witness shape; rule 7 flags
   OP_IF anywhere in a tapscript; rule 3 for segwit v2+ is unchecked. Rule 7 dominates the
   rejections (`bip110_daily.json` → `rules`). Ask how sensitive 3.01% is to the rule 7
   reading.
7. **XBT self-check range.** XBT was checked for BIP-110 violations only from block 973,440,
   while BIP-110 took effect there at 961,640.
8. **Dates and cutoffs.** Days are UTC block timestamps, which are not monotonic; the two
   chains' last blocks are seven minutes apart. Check whether any daily comparison is
   sensitive to this.
9. **XBT's SHA-256 period.** Blocks 961,632 to 961,639 (Aug 8 to 28) are in XBT totals but
   not in its daily chart. Check the effect on XBT shares.
10. **Prices.** XBT trades on thin venues; three opening candles were dropped. Check whether
    the peak-price claim survives using daily closes instead of hourly.
11. **Nodes.** Clearnet listening nodes only; self-reported user agents; two very different
    totals (8,925 vs. 101,179). Check whether "about 5%" holds under both.
12. **What the data cannot show.** Economic weight (exchanges, custodians), miners' private
    intentions, and which chain "is Bitcoin" are not measurable here. Flag any page sentence
    that implies otherwise.

## How the site is built (if you need to edit or rebuild a page)

- Each chapter is one self-contained HTML file in the repo root (`bch-vs-btc.html`,
  `eth-vs-etc.html`, `btg-vs-btc.html`, `blake2b-retrospective.html`), with its CSS, chart
  data and chart code inline. There is no build step.
- Chart data is inlined as JavaScript constants (in Chapter 4: `PRICE`, `SP`, `SV`, `BIP`,
  `BIP_BY`, `COUNTS`). They are derived from the files in `data/`.
- Pages are bilingual. In Chapter 4, each translated element carries its Spanish text in a
  `data-es` attribute, and chart text uses a `T(english, spanish)` helper.
- Notes are numbered `[1]`… and link to `#fn04-n`; sources are `S12` (`F12` in Spanish) and
  link to `#src-n`.
- Section 04's timeline can be drawn three ways (Neutral, BTC, XBT) so the chart does not
  decide which chain continues Bitcoin.
