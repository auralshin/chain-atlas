# Architecture

ZK-EVM rollup. L2 state transitions are proven on L1 by SNARKs over the EVM execution trace; the same EVM bytecode that runs on Ethereum runs on Scroll.

## Components

| Component | Role |
|---|---|
| **Sequencer** | Orders txs, executes EVM, produces L2 blocks |
| **Prover** | Generates zkEVM proofs over the L2 execution trace |
| **`l2geth`** | Modified geth executing on L2; provides JSON-RPC |
| **L1 contracts** | `ScrollChain` (state commitments), `L1ScrollMessenger`, `L1GatewayRouter`, `L1MessageQueue` |
| **L2 system contracts** | `L2ScrollMessenger`, `L1GasPriceOracle`, predeploys |

Single sequencer/prover, currently operated by Scroll Foundation. Decentralization is on the roadmap.

## Block production

| Parameter | Value |
|---|---|
| L2 block time | ~3 s {{unsourced: confirm — has been tuned}} |
| Sequencer | Single, Scroll Foundation |
| Batch cadence | Every few minutes; size-driven |
| Settlement layer | Ethereum L1 |

Sequencer publishes L2 blocks immediately; periodically posts batches to L1 via `ScrollChain.commitBatch`. Provers generate ZK proofs of these batches; once a proof is submitted via `ScrollChain.finalizeBatchWithProof`, the batch is canonical on L1.

## Two-stage L1 finality

| Stage | L1 event | What's settled |
|---|---|---|
| **Committed** | `CommitBatch` | Batch's data on L1 (calldata or blob), state root proposed |
| **Finalized** | `FinalizeBatch` (with proof verification) | ZK proof verifies the state transition; canonical |

Compared to zkSync Era's three-stage flow, Scroll's is two-stage — there is no "execute" delay. Once finalized, the state is canonical on L1 immediately.

## Derivation

L2 chain is derived from:
- L1 batch data (committed batches define the canonical L2 history).
- L1 message queue (L1→L2 deposits / messages, included by sequencer in batches).

A verifier-mode L2 node reads L1 to reconstruct the L2 chain trustlessly. Most third parties run Scroll's hosted RPC; self-hosting is possible but uncommon.

## Fork choice

No L2-level consensus. Canonical L2 chain is whatever the L1 batch sequence defines. Reorgs occur because:
- Sequencer rewinds the sequencer-feed head.
- L1 reorg invalidates a committed batch (rare; requires deep L1 reorg).

ZK proofs ensure no invalid state can be finalized — there is no fault-proof challenge needed.

## L1 → L2 messaging (deposits)

User calls `L1ScrollMessenger.sendMessage` (or `L1StandardERC20Gateway.depositERC20` etc.) on L1. The message is queued in `L1MessageQueue`. The sequencer includes it in a future L2 batch.

L2-side: the message is delivered as a tx **with a special `from` indicating L1 origin** (aliased L1 address). It executes against the message's target contract.

## L2 → L1 messaging (withdrawals)

L2 contract calls `L2ScrollMessenger.sendMessage`. This emits an event; the message is included in the next L2 block's `L1MessageQueue` snapshot.

When the batch containing this withdrawal is finalized on L1, the user can call `L1ScrollMessenger.relayMessageWithProof` with a Merkle proof of inclusion. Funds release on L1.

There is no challenge window — once the batch is `finalized`, the withdrawal is claimable immediately.

## L1 fee oracle

Scroll has an `L1GasPriceOracle` predeploy (analogous to OP's `GasPriceOracle`) that tracks the L1 base fee + blob base fee. Receipts include L1-fee fields computed via this oracle.

Indexers must be aware of fee-formula changes at upgrades (similar to OP Stack: Bernoulli, Curie, Darwin shifts).

## Where the data lives

| Data | Where to fetch | Notes |
|---|---|---|
| L2 blocks, txs, receipts, logs | `l2geth` JSON-RPC | Standard ETH RPC + extras |
| L2 finality state | L1 `ScrollChain.CommitBatch` + `FinalizeBatch` events | Two stages |
| L1→L2 deposits | L1 `L1MessageQueue.QueueTransaction` | Each becomes a tx on L2 |
| L1→L2 messages | L1 `L1ScrollMessenger` events | Bridge entry |
| L2→L1 withdrawals | L2 `L2ScrollMessenger.SentMessage` | Two-step exit on L1 |

A complete Scroll indexer reads L1 + L2 in lockstep (same pattern as OP Stack and zkSync Era).
