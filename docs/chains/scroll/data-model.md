# Data Model

EVM-equivalent at the data-model level. Standard tx types, standard receipts, standard logs. The deltas are L1-fee receipt fields and a small set of system events.

## Block

Standard Ethereum execution block. Same fields as Ethereum's post-Cancun block. No Scroll-specific block-level fields exposed via JSON-RPC.

## Transactions

Standard Ethereum types only:
- `0x00` Legacy
- `0x01` EIP-2930
- `0x02` EIP-1559
- `0x04` EIP-7702 (post-Pectra-equivalent — verify Scroll activation {{unsourced}})

**No Scroll-specific transaction types** — unlike OP/Arbitrum, there is no system tx type. L1→L2 deposit messages execute as **regular calls** with the `from` set to an aliased L1 address.

## Receipts

Standard ETH receipt **plus** L1-fee fields (similar to OP Stack):

| Field | Type | Meaning |
|---|---|---|
| `l1Fee` | U256 | Fee paid (in L2 ETH) for L1 DA |
| `l1GasUsed` | U256 | L1 gas estimate for this tx |
| `l1GasPrice` | U256 | L1 base fee at production |
| `l1FeeScalar` | string / fields | L1 fee scaling parameters |
| `l1BlobBaseFee` | U256 | Post-blob-DA: L1 blob base fee |

The exact fields depend on the Scroll fork era — Bernoulli changed L1 fee accounting, Curie introduced blob support, etc. **Version your decoder by upgrade.**

## Logs

Standard ETH logs.

Notable system events:

| Event | Source | Why |
|---|---|---|
| `SentMessage` | L2 `L2ScrollMessenger` | L2→L1 message creation (withdrawal) |
| `RelayedMessage` | L2 `L2ScrollMessenger` | L1→L2 message executed |
| `CommitBatch` | L1 `ScrollChain` | Batch posted to L1 |
| `FinalizeBatch` | L1 `ScrollChain` | Batch's ZK proof verified, state canonical |
| `QueueTransaction` | L1 `L1MessageQueue` | L1→L2 message queued |
| `SentMessage` | L1 `L1ScrollMessenger` | L1→L2 message bridge entry |
| `RelayedMessage` | L1 `L1ScrollMessenger` | L2→L1 message released on L1 |

## Predeploys / system addresses

Scroll uses high-address predeploys (similar to OP Stack pattern but at different addresses {{unsourced: confirm exact ranges}}):

| Address | Name | Purpose |
|---|---|---|
| `0x5300000000000000000000000000000000000000` | (system zero) | Reserved {{unsourced}} |
| `0x5300000000000000000000000000000000000002` | `L1GasPriceOracle` | L1 fee parameters |
| `0x5300000000000000000000000000000000000005` | `L2ScrollMessenger` | L2→L1 + L1→L2 messaging |
| `0x5300000000000000000000000000000000000006` | `L2GatewayRouter` | Native bridge router |

Verify exact addresses against [`scroll-tech/scroll-contracts`](https://github.com/scroll-tech/scroll-contracts).

## L1 → L2 message format

When the sequencer includes an L1 message in an L2 batch, the resulting L2 tx looks like a **regular tx** in JSON-RPC with:

- `from` = aliased L1 address (`L1_address + 0x1111000000000000000000000000000000001111`).
- `to` = the message target.
- `value` and `data` from the L1 `sendMessage` call.

Indexers must detect L1-originated messages by either:
- Cross-referencing with the L1 `L1MessageQueue.QueueTransaction` events.
- Recognizing the L1-aliased `from` pattern (offset `0x1111…1111`).

## State

Same MPT as Ethereum. State trie format identical (Scroll uses standard EVM state for SNARK-friendliness via the zkEVM circuit, not a different trie).

## Genesis

Scroll genesis is at L2 block 0, dated 2023-10-17. Genesis allocates the system predeploys at the addresses above.

There is no pre-Scroll era to worry about; Scroll Sepolia testnet has its own separate history.
