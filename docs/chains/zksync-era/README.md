# zkSync Era

**Status:** in-progress (template populated — pending review)
**Family:** ZK rollup — non-bytecode-equivalent EVM (EraVM)
**Mainnet launch:** 2023-03-24
**Native node clients:** [`zksync-era`](https://github.com/matter-labs/zksync-era) (Matter Labs)

## TL;DR for indexers

- **Hardest EVM-adjacent chain to index correctly.** Most ETH-trained indexers misdecode it on day one.
- **Native account abstraction.** Every L2 account can be a contract; there are **no EOAs in the Ethereum sense**. Tx validation is delegated to the sender's contract account.
- **Custom transaction types.** Type `0x71` (EIP-712-style zkSync transactions) is dominant. Standard ETH tx types (0/1/2) work but are not the typical case.
- **Paymasters are first-class.** `fee_payer ≠ sender` is the **common** case, not the exception. Indexers attributing "who paid for this tx" by `tx.from` are wrong.
- **EraVM ≠ EVM.** Solidity compiles via `zksolc` to EraVM bytecode. Some opcode semantics differ. **Bytecode-level analysis** (decompilation, bytecode similarity) requires EraVM-aware tooling.
- **Validity proofs, not fault proofs.** Finality after L1 batch + state-transition proof + execute call. Two-step proof flow: prove (zk verification) then execute.

## Read order

1. [architecture.md](architecture.md) — sequencer, prover, EraVM, three-stage finality
2. [data-model.md](data-model.md) — type 0x71 txs, native AA, paymaster, system contracts
3. [reorgs-finality.md](reorgs-finality.md) — three-stage L1 commit (commit / prove / execute)
4. [rpc-surface.md](rpc-surface.md) — `zks_*` namespace, EIP-712 helpers
5. [forks-changelog.md](forks-changelog.md) — Boojum proof migration, Era v22+/v23+ protocol versions
6. [gotchas.md](gotchas.md) — paymaster attribution, system tx, native AA validation
7. [storage.md](storage.md) — Postgres + ClickHouse with paymaster + AA tracking
8. [skeleton.md](skeleton.md) — illustrative Rust
9. [references.md](references.md) — primary sources

## What is *not* in scope here

- **zkSync Lite** (formerly zkSync 1.0) — separate chain; payment-only; deprecated for new development. Out of scope.
- **ZK Stack chains** (other Hyperchains built on zkSync's stack) — same patterns; chain-specific config differs. The patterns transfer; addresses do not.
- **Validium / Volition modes** — alternate DA modes. Same execution layer, different DA assumption.
