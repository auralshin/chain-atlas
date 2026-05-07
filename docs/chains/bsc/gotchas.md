# Gotchas

BSC is mostly Ethereum-shaped. The gotchas are about consensus / operational caveats and the historical Beacon Chain entanglement.

## Two finality regimes — pre and post Plato

Pre-Plato BSC has **no protocol finality**. Post-Plato, BSC has fast Casper-FFG-style finality.

Indexer code that uses `eth_getBlockByNumber("finalized")` against a pre-Plato history block either errors or returns the head, depending on client. **Detect the era** via the chain config and switch finality logic accordingly.

## Validator-collusion threat is real

The active validator set is small (21). The 2022 Cube DAO incident demonstrated coordinated validator action. Audit-grade workloads should not assume the validator set is non-colluding.

**Fix:** for high-stakes use cases, layer additional confirmation depth or cross-signal (e.g. don't credit cross-chain transfers based purely on BSC-side `finalized` — wait for the destination chain's finalization too).

## Block time has changed

Pre-Lorentz: 3 seconds. Post-Lorentz (2025-04-29 05:05:00 UTC, per [BSCChainConfig](https://github.com/bnb-chain/bsc/blob/master/params/config.go)): toward 1.5 seconds. Indexers that hardcode "blocks per hour" or "blocks per day" estimates from a constant block time are wrong post-Lorentz.

**Fix:** never hardcode block time. Compute from actual block timestamps for any calendar-time conversion.

## ExtraData format depends on era

Pre-Plato: `extraData = | vanity | validator addresses (epoch boundary only) | seal |`
Post-Plato: `extraData = | vanity | validators + BLS keys (epoch boundary) + attestation aggregate | seal |`

Parsers must branch on the chain era. The `bsc` source has the canonical decoder.

## System contract upgrades happen via cross-validator consensus

Validators can vote to upgrade the system contracts at `0x0000…10xx`. When this happens, the bytecode at those addresses changes between blocks **without** a contract-creation tx.

Indexers caching contract code at these addresses get stale ABIs. **Re-fetch system contract bytecode at validator-coordinated upgrade events** (announced via official channels).

## BNB Beacon Chain history is mostly noise post-fusion

Pre-2024 BSC has substantial cross-chain event volume from `TokenHub`, `RelayerHub`, `LightClient`, `GovHub`. Most of it is irrelevant to current applications.

Indexers backfilling from genesis spend significant time on this volume. **If your application doesn't need historical cross-chain data, skip these contracts during backfill.**

## Wrapped BNB (`WBNB`) is not the native token

For DeFi protocols, BNB-as-ERC-20 is `WBNB` at `0xbb4CdB9CBd36B01bD1cBaEBF2De08d9173bc095c`. The "native BNB" you see in tx `value` fields is **not** that contract — it's the gas-paying native asset, which has no contract address.

Indexers tracking "all BNB transfers" must distinguish:
- Native BNB transfers (in `tx.value` and traces).
- WBNB ERC-20 transfers (in WBNB contract `Transfer` events).

These are not the same and double-counting is a common bug.

## Validators in the candidate pool ≠ active validators

The active set (21) is chosen from a candidate pool (~41+ post-fusion). An indexer building "validator activity" UIs should distinguish:
- Active in this epoch (producing blocks).
- In the candidate pool (eligible).
- Slashed / jailed (currently barred).

The `ValidatorContract` and `StakeHub` events expose all three states.

## Public RPC limits are tight

Unlike Ethereum where Alchemy / Infura free tiers can handle moderate workloads, BSC public RPCs and free-tier endpoints rate-limit aggressively. Block-range limits on `eth_getLogs` are often **500–2000 blocks**.

**Fix:** self-host or pay for archive. There is no good free path for serious BSC indexing.

## Halts happen — chain may pause for hours

Operator-coordinated halts (Cube DAO recovery, post-incident response) have paused the chain for extended periods. Indexers must:
- **Not crash** when `head` stops advancing.
- **Alert** if head is stale > some threshold.
- **Read official communications** for any retroactive state changes when the chain restarts.

## EIP-1559 came late

BSC adopted EIP-1559 well after Ethereum (Hertz fork). Indexers spanning pre- and post-Hertz history must handle:
- Pre-Hertz: legacy txs only; `gasPrice` is the full price.
- Post-Hertz: type-2 txs introduced; `baseFee` exists; receipts have `effectiveGasPrice`.

Standard Ethereum gas-pricing logic works post-Hertz; pre-Hertz needs the legacy formula.

## BEP-95 burns affect supply tracking

BEP-95 (Bruno fork) burns a portion of gas fees — similar in spirit to EIP-1559's base fee burn but implemented differently. Indexers building supply / circulating-supply dashboards must account for the burn.

The burn happens in a system contract; the precise accounting is in `SystemRewardContract` events.

## opBNB is a separate chain — not a BSC tx type

opBNB is an OP Stack L2 on BSC. **opBNB transactions are not BSC transactions.** A BSC indexer does not see opBNB activity except via L1 batch-posting and bridge contracts on BSC.

Don't conflate. If you're indexing opBNB, you need a separate OP Stack indexer pointed at BSC as the L1 (and following the OP Stack patterns from [optimism/](../optimism/)).
