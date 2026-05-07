# Architecture

Polygon PoS is a **sidechain** anchored to Ethereum via periodic checkpoints. State transitions happen entirely off Ethereum; only succinct commitments are posted to L1.

## Two-layer node

Indexer-relevant: every Polygon node runs **two processes**.

| Component | Role | Based on |
|---|---|---|
| **`bor`** | Block production (EVM execution) | `go-ethereum` fork |
| **`heimdall`** | Consensus, validator coordination, L1 checkpoint posting | Tendermint fork (with Cosmos SDK modules) |

`bor` and `heimdall` talk to each other via gRPC. Neither alone is enough to validate the chain; an indexer must trust an RPC provider that runs both, or run both itself.

## Block production: spans and sprints

Validators are elected for a **span** (6400 blocks). Within a span, block production is split into **sprints**. One validator produces all blocks in a sprint, then production rotates to the next. Both span and sprint lengths are configurable per chain via heimdall; values below are the BorMainnetChainConfig defaults.

| Parameter | Value |
|---|---|
| Block time (period) | 2 s from genesis through Madhugiri (block 80,084,800); 1 s thereafter ([BorMainnetChainConfig](https://github.com/maticnetwork/bor/blob/develop/params/config.go) `Period` map) |
| Sprint length | 64 blocks (genesis–Delhi); 16 blocks from Delhi (block 38,189,056) onward ([BorMainnetChainConfig](https://github.com/maticnetwork/bor/blob/develop/params/config.go) `Sprint` map) |
| Span length | 6400 blocks (heimdall default) |
| ProducerDelay | 6 s (genesis–Delhi); 4 s from Delhi onward ([BorMainnetChainConfig](https://github.com/maticnetwork/bor/blob/develop/params/config.go) `ProducerDelay` map) |
| Active validator set | Varies; staked validators eligible per epoch via heimdall election. {{unsourced: pin canonical max-active-set value}} |
| Producer rotation | Per sprint within span |

If the assigned producer for a sprint is offline, the sprint is **skipped** — blocks for that sprint period are not produced. This is part of why bor reorgs deep: late blocks from a missed producer can re-enter the chain.

## Heimdall: validator coordination + checkpoints

Heimdall is a Tendermint-Cosmos chain whose state machine includes:

- **Staking**: validator registration, stake updates (deposited via L1 `StakeManager`).
- **Span election**: chooses bor's producer set for each span.
- **Checkpoint generation**: every ~30 minutes, heimdall validators sign a Merkle root of recent bor blocks + state syncs. The signed checkpoint is posted to L1's `RootChainManager` / `RootChain` contract.
- **State sync**: ingests L1 deposit events and replays them on bor.
- **Slashing**: handles double-sign and downtime.

Heimdall has its own block height (separate from bor's), its own RPC (Tendermint RPC + Cosmos gRPC), and its own data structure. **Indexers caring about checkpoint timing must read heimdall, not bor.**

## State sync (L1 → L2)

The Polygon-native bridge mechanism. Steps:

1. User calls `RootChainManager.depositFor(...)` on Ethereum. (Or any other contract that calls into `StateSender`.)
2. L1 emits `StateSynced(id, contractAddress, data)`.
3. Heimdall validators observe the L1 event and inject a corresponding state-sync record into the heimdall chain.
4. At each new sprint on bor, heimdall pushes the queued state-syncs to bor.
5. Bor executes the state-syncs as **synthetic transactions** at the start of the block — but they have no transaction hash, no `from`, no `to` from the standard JSON-RPC perspective. They show up as events on the receiving contract.

**This is the most-broken thing about Polygon for ETH-trained indexers.** State changes happen with no on-chain tx to track. See [gotchas.md](gotchas.md).

## Checkpoint (L2 → L1 anchoring)

Every ~30 minutes:

1. Heimdall validators sign a Merkle root over bor blocks `[start, end]`.
2. The proposing validator submits this root to Ethereum via `RootChain.submitCheckpoint(...)`.
3. The checkpoint is recorded on L1, including the bor block range and the Merkle root.

Checkpoints are the **only** L2-state commitment to L1. There is no fraud proof, no validity proof. **Trust is in the heimdall validator set's 2/3+ honesty.**

For indexers, the checkpoint provides:
- **Strong finality signal**: a bor block contained in a checkpoint cannot be reorged short of L1 reorg + heimdall validator collusion.
- **Withdrawal anchor**: L2→L1 burns are claimed on L1 by proving inclusion against a checkpointed Merkle root.

## Withdrawals (L2 → L1)

User burns on L2 → block included in next checkpoint → user submits Merkle proof on L1 → L1 contract releases funds. There is **no fault-proof challenge window** beyond the checkpoint cadence — once checkpointed, the user can exit immediately.

Total time: typically ~30 min (checkpoint cadence) to a few hours (delays / waiting for next checkpoint).

## Where the data lives

| Data | Where to fetch | Notes |
|---|---|---|
| Bor blocks, txs, receipts, logs | `bor` JSON-RPC | Standard ETH RPC |
| Validator set, span info | `heimdall` Cosmos REST/gRPC | `/staking/validators`, `/bor/span/<id>` |
| Checkpoint history | L1 `RootChain.NewHeaderBlock` events | Or heimdall `/checkpoints/list` |
| Pending state syncs | `heimdall` `/clerk/event-record/list` | Queued from L1 to bor |
| L1 deposit events | L1 `StateSender.StateSynced` | Source of all state-sync records |
| L1 staking events | L1 `StakeManager` | Validator registration, stake changes |

A complete indexer reads **bor + heimdall + L1**. Three streams, three reorg-handling regimes.

## Decentralization

Heimdall has 100+ validators (Tendermint-style BFT). Validator set is permissioned-but-bonded: anyone can become a validator by staking POL, but the active set is bounded.

Bor producers are a subset of heimdall validators selected per span.

Compared to Ethereum: smaller validator set, weaker decentralization assumptions. Compared to BSC: more validators, more decentralized.
