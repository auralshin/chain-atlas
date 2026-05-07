# Data Model

Nitro post-2022-08-31 is EVM-equivalent at the contract level but carries Arbitrum-specific tx types, receipt fields, block fields, and a set of system precompiles unique to ArbOS.

## Block

Standard Ethereum execution block fields, **plus**:

| Field | Type | Meaning |
|---|---|---|
| `l1BlockNumber` | u64 | The L1 block number observed at the time this L2 block was produced. **Lags real L1 by some amount** (sequencer's view of L1, not strictly current). |
| `sendRoot` | bytes32 | Merkle root of L2→L1 messages emitted in this block (for outbox proofs). |
| `sendCount` | u64 | Cumulative count of outbox messages up to and including this block. |
| `mixHash` | bytes32 | Aliased to additional Arbitrum-specific data — varies by ArbOS version; do not rely on RANDAO semantics. |

**`l1BlockNumber` is the L2 block's view of L1, not the L1 block where this L2 block was *batched*.** Indexers that need the batch L1 origin must read it from `SequencerInbox` events.

## Transaction types

| Type | Hex | Origin | Notes |
|---|---|---|---|
| Legacy / EIP-2930 / EIP-1559 / EIP-7702 | `0x00`–`0x04` | User | Standard. |
| **`ArbitrumDepositTx`** | `0x64` (100) | System | ETH deposit from L1. |
| **`ArbitrumUnsignedTx`** | `0x65` (101) | System | L1-originated unsigned tx (e.g. owner ops). |
| **`ArbitrumContractTx`** | `0x66` (102) | System | L1 contract calling L2 contract directly. |
| **`ArbitrumRetryTx`** | `0x68` (104) | System | Redemption of a retryable ticket (auto or manual). |
| **`ArbitrumSubmitRetryableTx`** | `0x69` (105) | System | Creation of a retryable ticket (the tx that submits, not the redeem). |
| **`ArbitrumInternalTx`** | `0x6A` (106) | System | ArbOS internal — first tx of every L2 block, sets L1 block context, handles housekeeping. |

**No signature on system tx types.** Indexers validating signatures must skip them. The `from` is set explicitly by ArbOS (and L1-aliased for L1-contract origins, same `0x1111…1111` offset as OP Stack).

### Retryable tickets — the most distinctive Arbitrum thing

A user "deposits" ETH-or-tokens-with-callvalue from L1 by calling `Inbox.createRetryableTicket(...)`. This emits an L1 message that becomes:

1. **An `ArbitrumSubmitRetryableTx` (`0x69`) on L2** when the inbox message is processed. This *creates* the ticket but does **not** execute the call.
2. **An `ArbitrumRetryTx` (`0x68`) on L2** if and when the ticket is redeemed. The redemption can be:
   - **Auto-redeem**: included automatically if the user paid for L2 gas at submission time.
   - **Manual redeem**: anyone can call `ArbRetryableTx.redeem(ticketId)` within 7 days. After 7 days the ticket expires.

A successful flow has **two** L2 txs for one L1 deposit. A flow with insufficient gas at submit has one L2 tx (`0x69`) and a pending ticket; the ticket might never be redeemed.

**Indexer consequence:** "deposit happened" is not the same as "L2 contract was called." Track both states.

## Receipts

Standard ETH receipt **plus**:

| Field | Type | Meaning |
|---|---|---|
| `gasUsedForL1` | u64 | Soft accounting of how much of `gasUsed` represents the L1 DA cost component. |
| `effectiveGasPrice` | u256 | L2 gas price actually paid (may differ from base fee due to ArbOS pricing quirks). |
| `l1BlockNumber` | u64 | L1 block at the time of inclusion. |

**Arbitrum does not split L1 fee into a separate `l1Fee` field like OP Stack does.** Instead, the L1 cost is folded into `gasUsed` × `effectiveGasPrice`, with `gasUsedForL1` exposing the split. Indexers building "L1 fee in USD" dashboards compute it as `gasUsedForL1 * effectiveGasPrice`.

## Logs

Standard ETH logs. No Arbitrum-specific format changes.

Notable system events to index:

| Event | Source | Why |
|---|---|---|
| `MessageDelivered` | L1 `Bridge` | All L1→L2 inbox messages |
| `InboxMessageDelivered` | L1 `Inbox` | Delivered message body |
| `SequencerBatchDelivered` | L1 `SequencerInbox` | Each batch boundary on L1 |
| `L2ToL1Tx` | L2 `ArbSys` (`0x064`) | Each outbox message creation |
| `OutBoxTransactionExecuted` | L1 `Outbox` | Outbox message execution (after challenge window) |
| `NodeCreated` / `NodeConfirmed` (legacy) | L1 `RollupProxy` | Assertion lifecycle |
| `EdgeAdded` / `EdgeConfirmed` (BoLD) | L1 `EdgeChallengeManager` | BoLD assertion lifecycle |

## Predeploys / system addresses (ArbOS precompiles)

All in the `0x000…0064` (= 100) – `0x000…006D` range (note: low addresses, not the OP-style `0x4200…` range).

| Address | Name | Purpose |
|---|---|---|
| `0x0000…0064` | `ArbSys` | L2→L1 message send, block number queries, version info |
| `0x0000…0065` | `ArbInfo` | Account info — code size, balance |
| `0x0000…0066` | `ArbAddressTable` | Compressed-address registry (calldata savings) |
| `0x0000…0067` | `ArbBLS` (deprecated) | BLS keys; deprecated post-Nitro |
| `0x0000…0068` | `ArbosTest` | Testing only — not on mainnet |
| `0x0000…0069` | `ArbAggregator` (legacy) | Pre-Nitro aggregator info; mostly historical |
| `0x0000…006A` | `ArbFunctionTable` | Compressed-fn-selector registry |
| `0x0000…006B` | `ArbDebug` | Debug methods — disabled on mainnet |
| `0x0000…006C` | `ArbGasInfo` | Gas pricing parameters |
| `0x0000…006D` | `ArbOwner` | Chain owner ops; restricted |
| `0x0000…006E` | `ArbStatistics` | Chain stats |
| `0x0000…006F` | `NodeInterface` | Special — provides outbox proof construction, gas estimation. **Cannot be called by contracts**, only via RPC. |
| `0x0000…006B` | `ArbRetryableTx` | Retryable ticket redemption / cancellation. **Most-called precompile by users.** |

Full list and source: [`OffchainLabs/nitro/precompiles`](https://github.com/OffchainLabs/nitro/tree/master/precompiles).

`NodeInterface` is unusual: it's not deployed on chain, only synthesized by the RPC node. Indexers cannot index calls to it; it's an RPC-side helper.

## State

Same MPT as Ethereum, with the ArbOS-specific accounts (precompiles) at the addresses above. State trie format identical.

**Stylus contracts** are stored as regular contract code, but with a custom prefix indicating WASM. Activation upgrades them from "stored bytecode" to "executable WASM module." Indexers that snapshot contract code should not assume it's all EVM bytecode.

## Genesis

Nitro's genesis block on Arbitrum One is the migration block from Classic — **block 22,207,818** (per [`arbitrum_chain_info.json`](https://github.com/OffchainLabs/nitro/blob/master/cmd/chaininfo/arbitrum_chain_info.json) field `chain-config.arbitrum.GenesisBlockNum`). Indexers should treat this as block 0 for Nitro purposes; pre-Nitro Classic blocks are a separate decoder problem.
