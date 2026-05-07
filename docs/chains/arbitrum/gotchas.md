# Gotchas

ETH-trained or OP-trained indexers consistently break on these.

## Retryable tickets are a state machine, not a single tx

A user calling `Inbox.createRetryableTicket` produces **up to two** L2 txs (`0x69` then `0x68`) plus an expiry timer. Indexers that treat "deposit happened" as "L2 contract was called" miss the most important state: did the redemption succeed?

**Fix:** track the retryable lifecycle (pending / redeemed_auto / redeemed_manual / expired / cancelled). UIs should expose all five states.

## `l1BlockNumber` on L2 blocks is **not** the L1 block where this L2 block was batched

The `l1BlockNumber` field on an L2 block is the L1 block the **sequencer was observing at production time**. The L1 block that **batched** this L2 block is determined by `SequencerInbox` events on L1, and is a different (and later) L1 block.

Joining on `l1BlockNumber` to L1 events gets you correlations close in time to L2 production, not the canonical "this L2 block became L1-confirmed at L1 block N" relationship.

**Fix:** maintain a separate mapping from L2 block range → L1 batch tx → L1 batch block, derived from `SequencerBatchDelivered` events.

## L1 block number from `block.number` on L2 ≠ either of the above

Solidity `block.number` on Arbitrum returns an **arbitrary L2 block number** that does **not** strictly increment by 1 between consecutive L2 blocks {{unsourced: confirm semantics}}. Use `ArbSys.arbBlockNumber` or `ArbSys.arbBlockHash` for the canonical L2 block number.

For Arbitrum's L1-block-number semantics inside contracts: `ArbSys.l1BlockNumber()` returns the L1 view from ArbOS — same as the block field but explicit.

**Fix:** at indexing time, prefer the JSON-RPC `block.number` (which is the L2 block index) and ignore Solidity's `block.number` semantics unless you're indexing contract behavior that uses it directly.

## L1 fee is folded into `gasUsed`, not a separate field

OP Stack: `receipt.l1Fee` is a separate field. Arbitrum: there is **no `l1Fee`**. The L1 cost is recovered from `gasUsedForL1 * effectiveGasPrice`, where `gasUsedForL1` is a soft accounting field.

Indexers building cross-chain "L1 fee paid" comparisons must compute it differently per chain.

**Fix:**
```rust
fn l1_fee_paid(receipt: &ArbReceipt) -> U256 {
    U256::from(receipt.gas_used_for_l1) * receipt.effective_gas_price
}
```

## Outbox proofs are constructed by the RPC node, not the chain

To call `Outbox.executeTransaction` on L1, a user needs a Merkle proof of their L2→L1 message against a confirmed `sendRoot`. This proof is constructed by `nodeinterface_constructOutboxProof` — an RPC method that synthesizes the proof from `sendRoot` history.

If your node is a non-archive or has gaps, `constructOutboxProof` can fail. **Indexers helping users with withdrawals must run a full archive node** that has continuous `sendRoot` history.

## `NodeInterface` is a "virtual precompile"

The address `0x0000…006F` looks like a precompile, but it is **not deployed on chain**. It exists only at the RPC layer. Calls to it from on-chain contracts **revert**. Calls to it from JSON-RPC `eth_call` work as if it were a contract.

Indexers cannot ingest "calls to NodeInterface" because there are no on-chain calls to it.

## Two-stack data: Classic and Nitro

Pre-2022-08-31 data on Arbitrum One uses the AVM stack. **Cannot be decoded with Nitro tooling.** Indexers that try to stream from genesis end up corrupted.

**Fix:** start streaming at the Nitro genesis block — height **22,207,818** on Arbitrum One (per [`arbitrum_chain_info.json`](https://github.com/OffchainLabs/nitro/blob/master/cmd/chaininfo/arbitrum_chain_info.json)). Skip Classic data entirely, or import a one-time snapshot.

## Arbitrum One vs Arbitrum Nova vs Orbit chains

Same Nitro stack, different chains:
- **Arbitrum One**: chain ID 42161, full DA on L1.
- **Arbitrum Nova**: chain ID 42170, AnyTrust DAC.
- **Orbit chains**: any chain ID; their own contracts on a parent chain.

Hardcoding L1 contract addresses or chain IDs from One does not work on Nova or Orbit. Pull from the chain's onchain config.

For Nova specifically: the indexer **cannot reconstruct the chain from L1 calldata alone**, because the DAC holds the actual data. Either run a Nova full node (which queries the DAC) or use the sequencer feed directly.

## Sequencer feed and `eth_subscribe("newHeads")` can disagree briefly

The sequencer's broadcast feed has the lowest latency. The standard `eth_subscribe("newHeads")` follows the feed but can lag slightly during high-load. They will converge; in the gap, an indexer subscribed to both might see a "new head" twice.

**Fix:** dedupe by block hash; treat `newHeads` as authoritative for the standard RPC view.

## Predeploy bytecode changes at ArbOS upgrades

ArbOS precompiles are **upgraded in place** at ArbOS version bumps. Bytecode at `0x000…0064` etc. changes. Cached ABIs go stale.

**Fix:** at each known ArbOS activation timestamp, refresh predeploy ABIs. Or use `ArbSys.arbosVersion()` at startup and pin the version-specific ABIs.

## ETH on L2 — same caveat as OP Stack, different mechanism

L2 ETH balance changes from a deposit show up as an `ArbitrumDepositTx` (`0x64`) or via the retryable's value field. There is no `Transfer` event because L2 ETH is the native currency.

**Fix:** credit balances from system tx `value`; debit from `ArbSys.sendTxToL1` calls (the L2→L1 burn).

## Sequencer-side fee accounting differs from user-paid fee

The sequencer collects priority fee + base fee (in L2 ETH). Some of this is forwarded to the L1 batch poster as reimbursement. The accounting is in `ArbosTest` / `ArbOwner` storage and is not directly visible from receipts.

For **gross** L2 fee revenue per block, sum receipts. For **net** sequencer revenue, you need to read internal ArbOS accounting state — `ArbGasInfo.getCurrentTxL1GasFees()` and related calls give snapshots.

## Stylus contracts — `eth_getCode` returns a header, not WASM bytecode

For an activated Stylus contract, `eth_getCode` returns a small EVM-format header that the EVM uses to dispatch into the WASM module. Indexers that snapshot contract code and try to disassemble it as EVM bytecode will fail.

**Fix:** detect the Stylus header (specific opcode prefix) and treat it as "WASM contract"; don't decompile.

## Precompile addresses are low (`0x000…00xx`), not high (`0x4200…`)

Indexer code copying OP Stack patterns sometimes scans for system events at `0x4200…` addresses. Arbitrum's precompiles are at the **low** end of the address space (`0x064`–`0x06F`). Different scan range entirely.
