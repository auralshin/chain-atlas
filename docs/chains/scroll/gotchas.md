# Gotchas

Scroll is the closest of any rollup to "just use the Ethereum indexer." The gotchas are operational + L1-fee-formula-shifts.

## Bytecode equivalence does not mean RPC equivalence

ETH-equivalent EVM execution doesn't mean every ETH-equivalent RPC quirk is preserved. Notable differences:

- `BLOCKHASH` may have a smaller window of accessible historical hashes {{unsourced: confirm}}.
- L2 timestamp is sequencer-set; can drift from L1.
- `PREVRANDAO` returns a sequencer-supplied value, not Ethereum L1 RANDAO.

For indexers tracking contract behavior that uses these opcodes, the value source differs.

## L1 fee formula changes per upgrade

Scroll has shifted the L1-fee formula at Bernoulli (blob support), Curie, and likely Darwin. If you recompute L1 fees from receipt inputs:

```rust
fn l1_fee(receipt: &ScrollReceipt, version: FormulaVersion) -> U256 {
    match version {
        FormulaVersion::PreBernoulli => /* calldata-only formula */,
        FormulaVersion::Bernoulli => /* blob + base fee formula */,
        FormulaVersion::Curie => /* refined */,
        FormulaVersion::Darwin => /* compression-aware */,
    }
}
```

Same shape as OP Stack's L1-fee versioning. Pin the activation timestamps from the chain config.

## L1-aliased deposits look like regular txs

L1→L2 messages execute on L2 as regular calls, with `from` set to the aliased L1 address. There is no deposit tx type; receipts look standard.

**Indexers must detect L1 origin by either:**
- Cross-referencing with L1 `L1MessageQueue.QueueTransaction` events.
- Pattern-matching the `from` address for the L1-alias offset (`0x1111…1111`).

Common indexer bug: attributing L1-deposit-induced contract calls to "user X submitted this tx" — but the user submitted it on L1, not on L2.

## L1 message queue ordering can differ from sequencer ordering

The sequencer chooses when to include L1 messages in L2 batches. There can be substantial delay (minutes to hours) between an L1 `sendMessage` call and its L2 execution. Indexers tracking "deposit happened" should distinguish:
- **L1 sendMessage** observed on L1 (fast).
- **L2 execution** observed on L2 (delayed).

A user-facing UI for "your deposit" must show the in-flight stage.

## Withdrawals are two-step on L1 (not three-step like zkSync)

Unlike zkSync Era's commit/prove/execute three-stage flow with an additional execution delay, Scroll's withdrawal flow is:

1. L2 burn (`L2ScrollMessenger.SentMessage`).
2. Wait for the batch to be committed AND finalized on L1 (single condition, since finalize is the proof-verifying step).
3. User calls `L1ScrollMessenger.relayMessageWithProof` with Merkle proof.
4. Funds release.

Total wait: typically a few hours.

## Bytecode equivalence has subtle exceptions

Scroll's zkEVM circuit has historically had small gaps in opcode coverage during early forks (e.g. some Cancun opcodes activated on Scroll later than on Ethereum). For indexers:
- **Static analysis tools** assuming "Scroll == Ethereum" can miss opcodes that are not yet supported.
- **Contract simulation** for predictive features should run against an actual Scroll node, not an Ethereum-equivalent EVM library.

Verify per-fork opcode availability against the Scroll specs.

## Single-sequencer fragility

Scroll has had brief operational outages typical of single-sequencer chains. Indexers should:
- Monitor sequencer liveness independently of L2 head advancement.
- Not crash on stale heads.
- Read official channels for any retroactive state changes (rare but possible).

## Receipt fee field shape changed at Bernoulli

Pre-Bernoulli receipts: subset of L1-fee fields, calldata-formula-shaped.
Post-Bernoulli receipts: includes `l1BlobBaseFee` and split scalars.

Naive deserializer breaks at the fork boundary. Use `Option<_>` for newer fields and check by block timestamp against the fork.

## Same chain id (534352) — but testnet (Sepolia) is 534351

Indexers configured for mainnet vs testnet need separate chain IDs and separate L1 contract addresses. Common copy-paste mistake when starting development.
