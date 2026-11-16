# History of Forks: data and methodology

This folder publishes the data behind the charts and figures in the four chapters of
[History of Forks](https://dave-rodt.github.io/fork-history/), and explains how it was
collected and why each choice was made. The aim is that anyone, person or AI, can check the
numbers and challenge the choices.

What is published: the derived datasets every chart is drawn from (`data/`), this write-up,
and an audit guide for AI tools ([`CLAUDE.md`](CLAUDE.md)). What is not published: the
collection scripts and the raw crawl (several GB of per-block files). Every raw input comes
from a public API, listed below, so the datasets can be rebuilt independently.

Most of this document is about Chapter 4 (Bitcoin BLAKE2b / XBT), whose numbers come from a
first-party crawl. Chapters 1 to 3 use public datasets and are covered briefly at the end.

## Naming

- **BTC**: the chain with the most proof of work after the August 8, 2026 split (the chain
  that did not adopt BIP-110 and kept SHA-256).
- **XBT**: the chain that enforced BIP-110 and on August 30, 2026 switched to BLAKE2b proof of
  work. Its only USD listing in the data is Neoxa, under the pair name `BTCB2/USDC`.
- The chapter does not decide which chain is "Bitcoin"; the datasets use `sha256` (BTC) and
  `blake2b` (XBT) as chain keys.

## Chapter 4

### Window and cutoffs

| | First block | Last block | Last block time (UTC) |
|---|---|---|---|
| BTC | 961,632 | 975,113 | 2026-11-08 23:39:25 |
| XBT | 961,632 | 981,156 | 2026-11-08 23:32:01 |

Block 961,631 is the last block both chains share. Both crawls end at the last block of
November 8, 2026 UTC. Each block is dated by its own timestamp in UTC. Block timestamps are
not strictly increasing, so a block stamped after November 8 inside the window is counted on
November 8.

The two chains share block heights only up to 961,631; after that each chain numbers its
own blocks.

XBT's first 8 blocks (961,632 to 961,639, August 8 to 28) were still SHA-256. They are in
every XBT total. The daily XBT charts start on August 30, the first BLAKE2b block (961,640),
and the August 8 to 29 days are added to the XBT tiles only when the selected range starts on
August 8.

### Transactions and block space (`spam_breakdown_daily.csv`)

Source: for every block, the transaction list with each transaction's size, fee, data flags
and OP_RETURN outputs, from the mempool explorers' REST APIs: mempool.space for BTC and
mempool.guide (built from the same code) for XBT
(`/api/block-height/:height`, `/api/v1/block/:hash/summary`). Output values per transaction
come from each block's raw data (`/api/block/:hash/raw`).

- **Shared transactions.** A transaction is "shared" when the same txid confirmed on both
  chains inside the window. Matching is by txid only.
- **Size.** Block space is virtual bytes (vbytes). A shared transaction is given BTC's vsize
  on both chains: mempool.space reports exact weight / 4, mempool.guide rounds to whole vbytes,
  and the difference over the whole window was 0.6 MvB.
- **One category per transaction, first match wins** (mempool's per-transaction flags are
  the bit numbers shown):
  1. `coinbase`: the first transaction of each block (its witness-commitment OP_RETURN is
     not counted as data).
  2. `op_return_over_83`: any OP_RETURN output larger than 83 bytes (BIP-110's limit,
     checked early so everything BIP-110 rejects for this reason lands here).
  3. `bpub`: an XBT-only transaction labeled as a BPUB reveal or fanout (see below).
  4. `inscription`: an Ordinals envelope in the witness (flag bit 26), or an XBT-only
     transaction carrying an envelope without OP_IF, which mempool's flag misses (8 txs).
  5. `stamp`: a fake script hash, the Stamps format (flag bit 27).
  6. `bpub`: any other transaction flagged as a fake public key (flag bit 25). On BTC only 5
     transactions in the window carry this flag.
  7. `rune`: an OP_RETURN output starting with the Runes marker (`OP_RETURN OP_13`, hex `6a5d`).
  8. `other_data`: any other OP_RETURN of 83 bytes or less, or a Taproot annex (flag bit 28).
  9. `other`: everything else.
- **Spam** is every category except `coinbase` and `other`. This follows the "Spam" filter
  group of mempool's block explorers (any OP_RETURN of any size, so all Runes; fake pubkey;
  inscription; fake scripthash; annex; OPNet, which never appears in the data), plus BPUB,
  which mempool's flags catch only in part. The same rule applies to both chains.
- **BPUB** (github.com/djkazic/bpub) hides files in fake multisig keys. A *fanout*
  transaction creates 3 or more P2WSH outputs of 1,000 sats or less; a *reveal* spends them,
  every input a 1-of-N CHECKMULTISIG of 1,000 sats or less whose "public keys" are file data.
  The dust limit keeps real 1-of-N wallets out. These rules were applied with full transaction
  data to every XBT-only transaction. A separate decoder rebuilt the files: through November 8,
  1,057 unique files (153 MB) written as 511 MB on chain; the rules caught 3,909 of the 3,911
  transactions that decode.
- **Columns:** `chain, date, category, txs, replayed_txs, vbytes, fees_sat,
  replayed_vbytes, replayed_fees_sat, value_sat, replayed_value_sat, bip110_txs,
  bip110_vbytes, bip110_fees_sat, bip110_value_sat`. "replayed" means shared with the other
  chain. `spam_breakdown_blocks.csv` gives blocks per chain per day.

### What BIP-110 would have rejected (`bip110_daily.json`, `bip110_*` columns)

BIP-110's seven consensus rules ([bips.dev/110](https://bips.dev/110/)) were applied to every
BTC transaction, read from raw blocks (Blockstream and mempool.space), as if BIP-110 had been
active on BTC from block 961,632. As the BIP specifies, inputs spending coins created before
activation are exempt from rules 2 to 7; a coin counts as "created before" if its txid is not
in the crawl from 961,632 on.

Known approximations, all of which could shift the result slightly:

- Script type is read from the witness shape, because previous outputs were not fetched: a
  Taproot script path is a last item that looks like a control block; P2WSH is any other
  multi-item witness whose last item is not a 33-byte key.
- Rule 7 flags OP_IF / OP_NOTIF anywhere in a tapscript, executed or not.
- In a scriptSig, the last push is treated as a possible P2SH redeem script.
- Rule 3 for undefined segwit versions (v2+) needs the previous output script and is not
  checked.

Every one of BTC's 13,482 blocks in the window was checked. Transactions caught per rule (a
transaction can break more than one): R1 44,851; R2 59,645; R4 13; R5 8; R7 691,583.

The same check on XBT blocks 973,440 to 981,156 finds no violations. BIP-110 took effect on
XBT at block 961,640 (the BLAKE2b fork); the 961,640 to 973,439 range was not checked.

### Value moved and fees

"Value moved" sums every transaction output, including change returned to the sender, so it
overstates what changed hands. USD figures use each day's closing price
(`daily_usd_close.csv`): BTC from Yahoo Finance (BTC-USD), XBT from Neoxa's daily candles,
which begin September 1, 2026. XBT's value and fee figures in the text count from August 30
(the BLAKE2b switch); its USD figures exist only from September 1.

### Prices (`xbt_btc_price_hourly.json`)

Hourly closes: XBT from Neoxa's `BTCB2/USDC` candles, BTC from Coinbase Exchange `BTC/USD`
candles. Neoxa's first three hourly candles printed $78,201, $68,000 and $49,263, near BTC's
own price, before a market for XBT existed; they are dropped, as is any XBT close above
$10,000. Hours with no XBT trade are missing, not filled. `t` is the Unix time of the hour,
`b2` the XBT close and `btc` the BTC close in USD.

### Nodes (`node_counts_2026-11-02.csv`)

- **Our crawl:** reachable IPv4 and IPv6 nodes of the shared peer-to-peer network on
  November 2, 2026, found by version handshake and repeated address requests until no new
  addresses appeared: 8,925 nodes. Each node's chain is read from its reported block height
  (tips that day: BTC 974,288, XBT 980,188); nodes reporting 961,639 are stuck on the
  BIP-110 SHA-256 chain. Software and version come from the self-reported user agent.
- **Luke Dashjr's count:** his per-client node counts, which add an estimate of nodes that
  do not accept incoming connections (101,179 in total), split by the share of each client
  setting service bits 28 (BLAKE2b) and 27 (BIP-110).
- Bitnodes was not used: since late October 2026 its snapshots list only nodes verified on
  BTC, so XBT and stuck nodes are absent.

### Other figures

- Mining-pool concentration on BTC: Foundry USA, AntPool and F2Pool mined 2,747 of 4,419
  blocks (62%) in the month to November 13, 2026, from mempool.space's mining-pool data.
- Historical claims (signaling levels, meeting dates, release dates, the eCash mainnet) cite
  their sources in the chapter's "Sources and methods" list.

### Choices a reader could reasonably challenge

These are judgment calls, stated so they can be tested:

1. **The spam definition is broad.** It counts every OP_RETURN, so all Runes, as spam,
   following mempool's filter group. A narrower definition (only what BIP-110 rejects) gives
   about 3% of BTC block space instead of 44%; both figures are shown.
2. **First match wins.** A transaction with both an inscription and a large OP_RETURN counts
   only once, as the larger OP_RETURN. Order changes the per-category split, not the total.
3. **BPUB was searched with full transaction data only on XBT-only transactions.** On BTC
   (and on shared transactions), BPUB-like transactions are caught only where mempool flags a
   fake public key, so BTC's BPUB share is a lower bound.
4. **Shared means same txid.** Two transactions that spend the same coin differently on each
   chain are not shared. Replays that confirmed after November 8 are not counted.
5. **Value moved includes change.** It is a measure of on-chain activity, not of economic
   transfer.
6. **The BIP-110 counterfactual is static.** It asks which of the transactions BTC actually
   confirmed would have been rejected; it does not model how users would have adapted.
7. **XBT's price is thin.** One small exchange; hourly prices can move on small trades.
8. **Node counts measure listening clearnet nodes.** Tor and I2P nodes are not in our crawl,
   and user agents are self-reported.

## Chapters 1 to 3

- **Prices, transaction counts, hashrate, fees:** CoinMetrics Community API (free tier),
  aggregated to months. Fees in USD are each day's native fees times that day's price, summed
  by month.
- **Chapter 1 (BCH vs. BTC):** BCH fee rate and value moved from Blockchair's daily block
  dumps. `coinmetrics_btc_bch_daily.json` is the raw daily pull; the other files are the
  monthly series drawn in the chapter.
- **Chapter 2 (ETH vs. ETC):** developer activity from Electric Capital's open-dev-data
  ecosystem lists, counting distinct GitHub commit authors per month. ETC's full repo list is
  used; ETH's is capped to its 300 most-starred repos. Replayed transactions come from a
  first-party crawl of both chains from block 1,920,000 through March 2017, matching
  transaction hashes.
- **Chapter 3 (BTG vs. BTC):** CoinMetrics monthly series; `btg_algo` marks the July 2018
  algorithm change.

## Public sources for rebuilding the data

- BTC blocks and transactions: `https://mempool.space/api/` (REST, documented at
  mempool.space/docs/api/rest); raw blocks also from `https://blockstream.info/api/`.
- XBT blocks and transactions: `https://mempool.guide/api/` (same API; send a browser-like
  User-Agent header).
- XBT prices: `https://neoxa.exchange/api/exchange/candles/BTCB2_USDC?interval=1h&limit=1000`
  (and `interval=1d`); at most 1,000 candles per call.
- BTC prices: Coinbase Exchange candles (hourly), Yahoo Finance BTC-USD (daily).
- BIP-110 rules: [bips.dev/110](https://bips.dev/110/).
- Node counts: Luke Dashjr's
  [Bitcoin Node Software](https://luke.dashjr.org/programs/bitcoin/files/charts/software.html).
- Chapters 1 to 3: [CoinMetrics API v4](https://docs.coinmetrics.io/api/v4/),
  Blockchair dumps, Electric Capital
  [open-dev-data](https://github.com/electric-capital/open-dev-data).
