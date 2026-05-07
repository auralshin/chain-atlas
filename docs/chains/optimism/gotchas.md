# Gotchas

Things that break ETH-trained indexers when they hit OP Stack. Each entry is a real failure mode someone has hit.

## L1 alias on deposit `from`

Deposit transactions originating from an L1 contract have their `from` field aliased: `aliased = original + 0x1111000000000000000000000000000000001111` (mod 2^160). This is a **security feature** (prevents L1 contract spoofing of L2 EOAs) — but indexers that build "addresses interacted with this L2 contract" lists from `from` end up with phantom addresses that don't exist on either chain.

**Fix:** for tx type `0x7E` with the alias offset present, store **both** the aliased and the unaliased address. UIs should display the unaliased L1 address with an "(L1)" badge.

## L2 timestamp can be ahead of L1 origin timestamp

The sequencer is allowed to advance the L2 timestamp ahead of its L1 origin's timestamp by up to **`max_sequencer_drift`** (600s in current deployments). Indexers that join L2 events to L1 events on `timestamp == timestamp` get drift-induced mismatches.

**Fix:** join on **L1 block number** (the L1 origin), not timestamps.

## Pre-Bedrock data is a different chain

OP Mainnet pre-2023-06-06 has different tx types, different receipt formats, and a different state root format. **Indexers cannot stream from genesis on a single decoder.** Either:
- Skip pre-Bedrock entirely (most products do this — explain to users that their pre-2023 history is unavailable).
- Import a one-time snapshot at the Bedrock activation block.

Do **not** try to stream OVM 1.0/2.0 with the same code as Bedrock+. They fundamentally differ.

## L1 fee formula changes per fork

Pre-Ecotone:
```
l1Fee = l1GasUsed * l1GasPrice * scalar
where l1GasUsed = compressionEstimate(rlp(tx))
```

Post-Ecotone:
```
l1Fee = l1GasUsed * (16 * l1BaseFee * baseFeeScalar + l1BlobBaseFee * blobBaseFeeScalar) / 16e6
```

Post-Fjord adds FastLZ-based size estimation:
```
estimatedSize = max(minTxSize, intercept + slope * fastLzSize)
```

If your indexer recomputes the L1 fee (instead of just storing the receipt's `l1Fee` field), **version the formula by L2 block timestamp** against the activation timestamps. Don't keyword-match by `l1FeeScalar` shape — there are blocks at the boundary where the receipt format changes.

## Receipts have two L1-fee field shapes

Pre-Ecotone receipts: `l1FeeScalar` (string, decimal scaled by 1e6).
Post-Ecotone receipts: `l1BaseFeeScalar`, `l1BlobBaseFeeScalar` (both u32-sized).

A naive deserializer breaks at the fork. Use a tagged-union or a receipt schema that gracefully accepts either set with `Option<_>`.

## `eth_getBlockReceipts` semantics changed pre/post-Ecotone {{unsourced: confirm}}

Some Bedrock-era nodes returned the system deposit receipt as a separate first entry; others omitted it. Validate that your decoder counts `len(block.transactions) == len(receipts)` for every block, including the system tx.

## "Reorg" mostly means the sequencer rewinding

ETH-trained reorg logic assumes: reorg = consensus disagreement, depth = ~3 blocks, rare. On L2, "reorg" mostly means the sequencer published a different `unsafe` chain than what derivation later confirms. Frequency: not rare on a per-second basis. Depth: usually 1–3 L2 blocks but sometimes more. **Treat unsafe-head changes as routine**, not as anomalies.

See [reorgs-finality.md](reorgs-finality.md) for the recommended commit policy.

## Withdrawal flow is two-stage on L1

Many indexer bugs in bridge accounting come from treating `MessagePassed` (L2 burn) as the end of the withdrawal flow. It isn't. The user must:

1. Burn on L2 (`MessagePassed` event on L2).
2. Wait for the output root containing that block to be posted on L1 (`OutputProposed` or `DisputeGameCreated`).
3. Wait for the challenge window (7 days post-permissioned-fault-proofs).
4. Call `proveWithdrawalTransaction` on L1 (`WithdrawalProven` event).
5. Call `finalizeWithdrawalTransaction` on L1 (`WithdrawalFinalized` event).

Step 4 is *separate* from step 5 — and a user can be stuck at step 4 for the 7-day window. Track the state machine, don't infer.

## Predeploy bytecode can change at upgrades

`L1Block`, `GasPriceOracle`, etc. are upgraded *in place* during network upgrades. Indexers that cache contract code by address get stale ABIs. **Re-fetch predeploy bytecode at every upgrade activation block.**

## ETH on L2 is "minted" via deposits, not bridged

L2 ETH balance changes from a deposit show up as a **deposit tx with `mint` set**. There is no `Transfer` event because L2 ETH is the native currency, not an ERC-20. Indexers that build ETH-balance histories from `Transfer` logs miss this entirely.

**Fix:** credit balances from deposit-tx `mint` field; debit from withdrawal `MessagePassed` events; and apply standard tx value/gas accounting.

## Sequencer fee splits across three vaults

L2 fee revenue is split between:
- `BaseFeeVault` (`0x42…001A`) — base fee.
- `L1FeeVault` (`0x42…001B`) — L1 DA fee.
- `SequencerFeeVault` (`0x42…0011`) — priority fee tip.

A single tx pays into all three. Total tx revenue = sum of vault deltas, not just `tx.gasUsed * tx.gasPrice`.

## Genesis is not block 0

Bedrock's "genesis" is at the activation block, not block 0. Block 0 of OP Mainnet exists in the legacy data set. Indexers that key on `block_number = 0` as genesis pick up legacy data. **Use the rollup config's `genesis.l2.number` and `genesis.l2.hash` from `optimism_rollupConfig`.**
