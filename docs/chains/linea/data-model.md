# Data Model

EVM-equivalent at the data model level, identical to [Scroll](../scroll/data-model.md). Differences are predeploy addresses and minor opcode-cost details.

## Block, transactions, receipts

Same shape as [scroll/data-model.md](../scroll/data-model.md). Standard ETH block. Standard tx types (0/1/2; EIP-7702 if/when ported). Receipts include L1-fee fields.

## Predeploys

Linea uses Linea-specific predeploy addresses (different from Scroll's `0x5300…` range). Verify against [`Consensys/linea-contracts`](https://github.com/Consensys/linea-contracts) or the Linea docs.

| Concept | Linea predeploy address {{unsourced: pin all of these}} |
|---|---|
| L2 Message Service | TBD |
| L1 Fee Oracle (gas pricing) | TBD |
| Bridge router | TBD |

For an indexer, query the Linea docs at deep-dive time and pin the addresses in code config.

## L1 fee receipt fields

Same field set as Scroll/OP Stack: `l1Fee`, `l1GasUsed`, `l1GasPrice`, scalar(s), and post-blob-DA: `l1BlobBaseFee`. Decoder versions per Linea fork era — see [forks-changelog.md](forks-changelog.md).

## L1 → L2 messages

Same flow as Scroll: user sends on L1 → message queued → sequencer includes in L2 batch → executes on L2 with L1-aliased `from`.

## L2 → L1 messages

Same flow as Scroll: L2 emits → batch finalizes on L1 → user claims with Merkle proof.

## Type-2 zkEVM gas-cost differences

The list of opcodes with gas costs that differ from Ethereum is small but non-empty. Verify against Linea's specs at deep-dive time. Most relevant for execution simulation, not for tx-level indexing.

## State

Same MPT as Ethereum. State trie format compatible.

## Genesis

Linea genesis at L2 block 0, dated 2023-07-11. Genesis allocation includes Linea predeploys and any initial system state.
