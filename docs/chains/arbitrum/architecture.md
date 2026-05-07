# Architecture

Arbitrum Nitro stack. Single binary (`nitro`) integrating L2 execution, L1-derived chain reconstruction, and (for full nodes) fraud-proof generation.

## Components

| Component | Role |
|---|---|
| `nitro` (the binary) | Runs the L2 EL (geth-fork) and consumes L1 batches. One process. |
| `nitro-validator` | Watches the rollup, posts assertions, optionally validates and challenges. |
| `arbnode` libraries | Internal — chain reader, batch poster, sequencer, broadcaster, retryable redeemer. |
| L1 contracts | `RollupProxy`, `Bridge`, `Inbox`, `SequencerInbox`, `Outbox`, `EdgeChallengeManager` (BoLD), `ChallengeManager` (legacy) |

The single-binary design is the cleanest contrast with OP Stack: `nitro` does internally what `op-geth` + `op-node` do separately.

## Block production (sequencer)

| Parameter | Value |
|---|---|
| L2 block time | Variable — blocks produced as transactions arrive (with a min interval). Effective average ~250 ms on Arbitrum One {{unsourced: confirm}}. |
| Sequencer | Single, operated by Offchain Labs. Decentralization roadmap exists. |
| L1 batch posting cadence | Every few minutes; size-driven |

The sequencer publishes L2 blocks immediately to a real-time feed (`arb_subscribe`-style WebSocket). Periodically the **batch poster** posts compressed batches to L1 via `SequencerInbox.addSequencerL2Batch`. Pre-EIP-4844 these were L1 calldata; post-Ecotone-equivalent activation, they are blobs {{unsourced: confirm exact Arbitrum activation date}}.

## Derivation pipeline (Nitro)

Nitro's chain is **deterministic in L1 + sequencer-feed data**. Two reconstruction modes:

1. **Trustless (verifier)**: only consume the L1 `SequencerInbox` and `Bridge` (delayed inbox). The chain is reconstructed entirely from L1 data.
2. **Sequencer-feed (fast)**: consume the sequencer's broadcast feed for low-latency reads, while still verifying against L1 batches in the background.

Indexers can run either mode. Most production indexers use the sequencer feed for liveness and the L1 stream for finality.

## Fork choice

There is no LMD-GHOST or BFT consensus on L2. The canonical L2 chain is whatever the L1-derivation function produces. L2 reorgs occur because:

- The sequencer rewinds its broadcast feed (publishes a different unsafe chain).
- L1 reorgs past a `SequencerInbox` batch tx, dropping the batch and forcing re-derivation.
- A successful BoLD/legacy challenge invalidates an assertion (theoretical; has not happened on mainnet to date {{unsourced}}).

See [reorgs-finality.md](reorgs-finality.md).

## Finality model

Three relevant heads:

| Head | Meaning | Reorg requirement |
|---|---|---|
| **Sequencer feed** | Sequencer-published, not yet on L1 | Sequencer rewinds OR equivocates |
| **L1-confirmed** | Batch on L1, L1 block hosting it not reorging | L1 reorg past the batch tx |
| **Finalized** | Assertion challenge window expired (~7 days legacy; configurable shorter under BoLD) | Slashing of L1 stake (≈ ETH finality) |

Pre-BoLD, the challenge window was ~7 days and validation was permissioned. BoLD reduces the challenge window {{unsourced: target ~6.4 days}} and makes validation permissionless via bonded validators.

## Fraud proofs

### Legacy: WAVM + interactive challenges

Pre-BoLD: a validator posts an `Assertion` (state root claim). Challengers bisect the disputed range down to a single WAVM (WebAssembly Virtual Machine) instruction; that instruction is executed on L1 by `OneStepProof.sol`. WAVM is a deterministic Wasm subset; `nitro` compiles itself to WAVM for the proof.

### BoLD (Bounded Liquidity Delay)

Newer system replacing legacy interactive challenges. Permissionless: anyone can post an assertion, anyone can challenge. The challenge resolves in a single bounded time window rather than potentially-unbounded depth-bisection. Mainnet activation: late 2024 / 2025 {{unsourced: confirm Arbitrum One activation date}}.

For an indexer, the **observable difference** is which L1 events to track:
- Legacy: `RollupNewAssertion`, `ChallengeManager` events.
- BoLD: `EdgeChallengeManager` edges, level-by-level confirmation events.

Both produce an "assertion confirmed" signal that finalizes a range of L2 blocks.

## Where the data lives

| Data | Where to fetch | Notes |
|---|---|---|
| L2 blocks, txs, receipts, logs | `nitro` JSON-RPC (`eth_*`) | Standard ETH RPC + Arbitrum extras |
| Sequencer feed | `nitro` WebSocket feed (deserializing custom format) | Fastest unsafe head; not part of `eth_subscribe` |
| L1 batches | L1 `SequencerInbox.addSequencerL2BatchFromOrigin` calls and blobs | Decompress to recover the L2 chain |
| Delayed inbox (deposits, force-includes) | L1 `Bridge.MessageDelivered` events | Each delayed message becomes a tx in a future L2 block |
| Retryable tickets | L1 `Inbox.InboxMessageDelivered` (kind = 9 retryable) | Tracked separately; lifecycle below |
| Outbox messages (L2→L1) | L2 `ArbSys.L2ToL1Tx` event + L1 `Outbox.OutBoxTransactionExecuted` | Two-stage exit, see [reorgs-finality.md](reorgs-finality.md) |
| Assertions / challenges | L1 `RollupProxy`, `EdgeChallengeManager` | For finality |

A complete Arbitrum indexer reads **L1 + L2 in lockstep**, including the L1 inbox / outbox / rollup contracts, similar to OP Stack.

## Arbitrum Nova delta

Same Nitro stack, except batches are posted as **hashes only** to L1; the actual data is held by an AnyTrust Data Availability Committee (DAC). If the DAC fails to provide data on request, the chain falls back to L1 calldata.

Indexer impact:
- **Cannot reconstruct the chain from L1 alone** for Nova in the AnyTrust path. Must query the DAC's `restful` API for batch contents, or use the sequencer feed.
- **Higher trust assumption**: the DAC could collude to withhold data. For most analytics indexers this is acceptable; for trust-minimized indexers, it is not.

Nova has its own `SequencerInbox` and `RollupProxy` on L1, distinct from One.
