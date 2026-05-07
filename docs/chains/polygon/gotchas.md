# Gotchas

Polygon's data model is mostly Ethereum-equivalent, but the operational and consensus details surprise indexer authors who expect rollup or ETH-style behavior.

## Reorgs are deep — plan for 128+

The folklore is real. Polygon has produced reorgs of 100+ blocks in the wild, and there is **no protocol upper bound** short of the heimdall checkpoint cadence.

**Fix:** confirmation depth of 256 blocks is a reasonable safety margin. For audit-grade systems, wait for L1 checkpoint inclusion (~30 min). Naive 12-block confirmation as on Ethereum is **not** safe for value-bearing data on Polygon.

## State syncs are ghost activity

L1→L2 messages arrive as **synthetic state changes with no transaction**. They emit logs but have no `tx_hash`, no `from`, no `to` from the standard JSON-RPC perspective.

Indexers that scan receipts to attribute on-chain activity miss state syncs entirely. An L2 contract balance that "magically" increases between blocks is almost certainly a state sync.

**Fix:** subscribe to L1 `StateSender.StateSynced` events and to heimdall's `clerk/event-record/list`. Maintain a state-sync table keyed on `state_id`. Cross-reference to L2 logs by destination contract + `state_id`.

## State-sync log indexes shift on reorgs

State syncs are injected at sprint boundaries. When bor reorgs across a sprint boundary, the state sync may be re-injected at a different bor block, with a different `(block_number, log_index)`.

**Fix:** key state-sync rows on `state_id` (heimdall-issued, monotonic, stable across bor reorgs), not on `(block_number, log_index)`.

## `eth_blockNumber` does not equal "current safe block"

On Ethereum, lots of code does `eth_blockNumber - 12` to get a "safely confirmed" block. On Polygon, **`eth_blockNumber - 12` is still in the reorg zone**.

**Fix:** use `eth_getBlockByNumber("safe")` or `eth_getBlockByNumber("finalized")` (post-Bhilai), or maintain your own checkpoint-derived finality state.

## Heimdall is a separate chain you have to index

Many Polygon indexers treat the chain as "just bor" and skip heimdall. That works for analytics-grade indexing of L2 contract activity. It does **not** work for:
- Tracking finality (no checkpoint visibility).
- Indexing state syncs reliably (heimdall is the source of truth for state-sync IDs).
- Tracking validator changes.
- Bridge accounting.

**Fix:** if your indexer is bridge-aware or finality-aware, run a heimdall full node. The Tendermint REST is straightforward.

## MATIC → POL: same chain, different label

The 2024 token migration changed the symbol/name on the L2 native token contract but not its address or balances. Indexers that hardcoded `"MATIC"` need updating; nothing else does.

## No L1 fee receipt fields

OP and Arbitrum receipts include L1-DA fee fields. Polygon receipts **do not** — Polygon is a sidechain, and there is no L1 DA cost per tx. `gas_used * gas_price` is the full fee.

This means cross-chain "L1 fee paid" comparisons need an explicit "Polygon: zero L1 fee component" branch.

## No deposit transaction type

OP/Arbitrum have system tx types for L1→L2 messages. Polygon does **not**. State syncs are not transactions. Indexer code reused from OP/Arbitrum that scans for system tx types finds none on Polygon — and silently misses bridge activity.

## Heimdall halts cascade silently into bor

If heimdall stops finalizing, bor keeps producing blocks for the rest of the current span, but:
- No new spans elected → eventually production stops.
- No checkpoints posted → bor finality stops advancing.
- State syncs queue up but are not delivered to bor (next sprint boundary blocks until heimdall is healthy).

Indexers monitoring only bor see "all is well, blocks keep coming" until production halts. **Monitor heimdall liveness separately.**

## Validator addresses and span elections

Bor's `miner` field is the validator's bor key, which is different from their heimdall key. Mapping requires a heimdall query or a maintained local mapping table.

For UIs showing "produced by validator X", join bor's `miner` to the heimdall validator list using the bor-key field on the heimdall validator record.

## Two kinds of withdrawal: PoS bridge and Plasma bridge

Historical Polygon has two bridges:
- **PoS bridge** (`RootChainManager`): the modern path. State-sync based. Most assets use this.
- **Plasma bridge** (`DepositManager` / `WithdrawManager`): legacy. Limited token support. Different exit semantics.

Indexers building bridge accounting must distinguish. Most assets and most users go through the PoS bridge.

## Bor and heimdall block numbers are independent

Bor block 50,000,000 is **not** related to heimdall block 50,000,000. They have entirely different cadences (bor 2s, heimdall ~5s {{unsourced: confirm heimdall block time}}). **Do not** join bor and heimdall events on block number — join via:
- L1 `RootChain.NewHeaderBlock` event for checkpoint correspondence.
- State-sync ID for state-sync correspondence.
- Validator address for staking correspondence.

## Block time has changed over time

Polygon's configured `Period` was 2 s from genesis through the Madhugiri fork (block 80,084,800), then 1 s thereafter ([BorMainnetChainConfig](https://github.com/maticnetwork/bor/blob/develop/params/config.go) `Period` map). Empirically observed inter-block gaps were higher than the configured period (~2.2 s on average pre-Madhugiri) due to producer-delay events at sprint boundaries, which themselves were tuned at the Delhi fork (block 38,189,056: ProducerDelay 6 → 4, Sprint 64 → 16). Indexers computing "blocks per hour" from a hardcoded block time are wrong for historical analysis.

**Fix:** never hardcode block time. Compute "X hours of history" by walking back from `eth_blockNumber` using actual block timestamps.

## Genesis allocation includes a lot of state

Polygon's genesis block has a substantial state allocation (validators, system contracts, initial token supply). Indexers that "skip the genesis block" because it has no transactions miss the genesis state allocation. This usually does not matter for application indexers, but matters for full state-tree replication.
