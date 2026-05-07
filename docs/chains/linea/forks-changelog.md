# Forks and Changelog

Linea's upgrade history is younger and shorter than Scroll's.

## Major upgrades

| Upgrade | Approx date | Indexer impact |
|---|---|---|
| **Genesis** | 2023-07-11 (alpha) / 2023-08 (public) | Block 0. Initial Vortex/PLONK proof system. |
| **Beta v1** {{unsourced}} | {{unsourced: 2023-Q4}} | Performance improvements; minor RPC additions. |
| **Beta v2** / **Mainnet** {{unsourced}} | {{unsourced: 2024}} | More opcodes covered; reduced exclusion list. |
| **Pectra-equivalent** {{unsourced}} | {{unsourced: 2025}} | EIP-7702 if/when ported. |

Linea's upgrade documentation is under [docs.linea.build](https://docs.linea.build/) and within the [`Consensys/linea-contracts`](https://github.com/Consensys/linea-contracts) repo. Verify exact heights / timestamps at deep-dive time.

## Opcode coverage shifts

A defining characteristic of Linea's evolution has been **expanding the set of opcodes covered by the prover**. Early Linea had a list of "excluded opcodes" — txs using them would be rejected by the sequencer. Each upgrade has shrunk this list.

For an indexer:
- **Pre-coverage upgrade**: txs using uncovered opcodes were excluded; you may see `linea_getTransactionExclusionStatusV1` returning `excluded` for these.
- **Post-coverage upgrade**: same txs are now included.

This is **Linea-unique**. Indexers spanning early Linea history must be aware that some txs that "should have worked" were rejected by the sequencer.

## L1 contract upgrades

The `ZkEvm.sol` verifier and `MessageService` contracts have been upgraded multiple times. Each upgrade:
- Changes the proof system (verifier circuit version).
- May change event signatures or fields on L1 events.

Indexers that decode L1 events must update ABIs at upgrade boundaries.

## What an indexer must do per upgrade

1. **Pin upgrade activation timestamps** at startup.
2. **Track opcode coverage** — keep a mapping of "which opcodes are usable per fork era" if you simulate execution off-chain.
3. **Update L1 verifier ABI** when verifier upgrades.
4. **Reprocess affected blocks** if any derived data depends on opcode availability or fee formula changes.
