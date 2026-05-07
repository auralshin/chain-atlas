# Forks and Changelog

Linea ships upgrades on a versioned cadence (Alpha v0.x → Alpha vN.x → Beta vN.x). Versions roll out to Sepolia first, then mainnet a few days to weeks later. Activation dates below are pulled from [`docs.linea.build/changelog/release-notes`](https://docs.linea.build/changelog/release-notes).

## Indexer-relevant upgrades

### Alpha era (2023-07 → 2024-12)

| Version | Mainnet date | Indexer impact |
|---|---|---|
| **Genesis (Alpha v0.2.0)** | 2023-07-11 | Mainnet launch. Initial zkEVM with halo2-derived prover, push-based messaging, N-N canonical token bridge. |
| **Alpha v0.2.1** | 2023-06-15 (Sepolia/precursor) | Optimized L2 logs in calldata via reduced `MessageSent` event payload — affects historical message decoding. |
| **Alpha v2** | **2024-02-13** | **Proof aggregation** (~30 batches per L1 proof, ~1/30 cost), **data compression** (up to 15:1), **Merkle-tree messaging** anchoring (32× calldata reduction). **Indexers must update L1 verifier ABI** and message-decoding logic. |
| **Alpha v3.0** | **2024-03-26** | **EIP-4844 blob support.** Pubdata moves from calldata to blobs. Block time reduced to 3 s on 2024-03-27. **L1 fee formula changes.** |
| **Alpha v3.1** | 2024-05-14 | Dynamic L1-fee Gwei thresholds; `eth_sendRawTransaction` RPC method. |
| **Alpha v3.2** | 2024-06-04 | Gas optimizations in blob submission, finalization, bridge. Bridge recipient feature enabled. |
| **Alpha v3.3** | **2024-06-11** | **Block time reduced to 2 s**, target 24M gas per block. **Affects "blocks per hour" assumptions** for any indexer using a constant block time. |
| **Alpha v3.4.1** | 2024-09-30 | **`linea_estimateGas` reactivated** (compatibility mode disabled). Indexer UIs that fell back to `eth_estimateGas` can return to `linea_estimateGas` for L1-fee-inclusive estimates. |
| **Alpha v3.5.0** | **2024-10-09** | **`finalized` block tag added to JSON-RPC.** Pre-3.5.0, `eth_getBlockByNumber("finalized")` either errored or returned head. **Mainstream indexers got their first L1-finalization signal at this fork.** |
| **Alpha v3.5.2** | 2024-09-23 | Tx exclusion API (`linea_getTransactionExclusionStatusV1`-class) for sequencer trace-limit queries. |
| **Alpha v3.6** | 2024-09-25 | Block gas limit raised to 30M. |
| **Alpha v4.0** | **2024-12-16** | Contract upgrades; removes `finalizeBlocksWithoutProof`; **enables fallback finalization**. L1 verifier ABI changes — indexers tracking finalization must update. |
| **Alpha v4.1** | 2025-02-04 | Automatic L1→L2 claiming; Linea Token API (alpha). |
| **Alpha v4.2** | 2025-03-26 | Bridge UI; Circle CCTP integration for USDC. |

### Beta era (2025+)

| Version | Mainnet date | Indexer impact |
|---|---|---|
| **Beta v1.3** | 2025-03-03 | Block gas limit raised to **2B**. Affects throughput math. |
| **Beta v1.4** | **2025-04-28** | **`eth_sendBundle` RPC method added.** State recovery mechanism introduced. |
| **Beta v2.0** | **2025-06-09** | **100% of EVM operations covered by ZK proofs.** Resolves the historical "excluded opcodes" issue — the era of sequencer-rejected txs ends here. **Indexers spanning pre-2.0 history must still handle exclusion artifacts.** Dictionary compression reduces L1 data costs by 7.5%. |
| **Beta v3.0** | 2025-08 (early) | **Limitless prover** deployment; eliminates per-batch sequencer module limits. |
| **Beta v3.1** | 2025-08 (early-mid) | Removes sequencer module limits; throughput increase. |
| **Beta v4.0** | **2025-10-22 → 2025-10-28** | **Pectra hard fork — applied in four sub-stages on mainnet:** Paris (2025-10-22), Shanghai (2025-10-23 10:00 UTC), Cancun (2025-10-28 10:00 UTC), Prague (2025-10-28 10:10 UTC). **Adds PUSH0, MCOPY** and the rest of the Ethereum-equivalent opcode set up to Prague. **Replaces Clique with Maru consensus client** (note: this is a sequencer-side change, not user-facing). |
| **Beta v4.3** | 2025-11-17 | Limitless prover upgrade; throughput → 100 Mgas/s. |
| **Beta v4.4** | **2025-12-03** | **Fusaka upgrade** (Ethereum's EVM upgrade). Indexers must port Fusaka-equivalent decoding. |
| **Beta v4.5** | 2026-01-28 | Credible Layer security infrastructure deployed. |
| **Beta v5.0** | **2026-03-23** | **EIP-7702 activated on mainnet.** Set-code transactions now usable. **Indexers must add tx-type 0x04 decoding.** Note: this is **roughly 11 months after Ethereum's Pectra activation** (2025-05-07) — Linea was an EIP-7702 laggard. |
| **Beta v5.1** | 2026-03-30 | Yield Boost protocol mechanism. |
| **Beta v5.2** | 2026-04-01 | Switched prover to small fields (faster proof generation). Throughput → 200 Mgas/s. **Breaking: ENS resolver contract address changed** — update configuration. **New verifier deployment via Security Council.** |
| **Beta v5.3** | 2026-05 (target) | Proof generation ~40% faster. |
| **Beta v5.4** | 2026-05 (target) | **Finality under 30 min** through prover improvements + continuous proof submission. |

If you are reading this after **2026-05-08** (the date this guide was last anchored), more upgrades have likely shipped. Check [docs.linea.build/changelog/release-notes](https://docs.linea.build/changelog/release-notes) for the canonical list.

## Critical fact: 100% EVM coverage didn't arrive until 2025-06-09

A defining characteristic of pre-Beta-v2.0 Linea was **excluded opcodes** — txs using uncovered opcodes were rejected by the sequencer with no on-chain trace.

**Coverage timeline:**
- **Genesis (2023-07) → Beta v2.0 (2025-06-09)**: partial EVM coverage. The set of excluded opcodes shrank with each upgrade but was never empty. Some txs were sequencer-rejected.
- **Post-Beta v2.0 (2025-06-09)**: **100% coverage**. No more opcode-driven rejections.

For indexers covering pre-2.0 Linea history:
- Some "missing" txs in your data may be sequencer-rejected, not lost.
- `linea_getTransactionExclusionStatusV1` (added at Alpha v3.5.2 in 2024-09) returns the rejection reason for a given tx hash.
- Treat the gap window 2023-07 → 2025-06-09 as having an extra failure mode that doesn't apply to Ethereum.

## Pectra adoption was multi-stage

Unlike Ethereum's single Pectra activation (2025-05-07), Linea applied Pectra in **four sub-stages** at Beta v4.0 (October 2025):
1. Paris (2025-10-22)
2. Shanghai (2025-10-23 10:00 UTC)
3. Cancun (2025-10-28 10:00 UTC)
4. Prague (2025-10-28 10:10 UTC)

For indexers: a tx using a Cancun-only opcode (e.g. MCOPY, transient storage) would not work between Linea genesis and 2025-10-28. EIP-7702 (set-code txs, type 0x04) **remained disabled** until Beta v5.0 (2026-03-23) — Linea kept 7702 separate from the Pectra umbrella.

## L1 contract / verifier upgrades

The `ZkEvm.sol` verifier and `MessageService` contracts have been upgraded at most major version bumps. Each upgrade:
- Changes the proof system / circuit version.
- May change event signatures on L1 events (`MessageSent`, `BlocksFinalized`, etc.).

Indexers decoding L1 events should pin the verifier proxy implementation address and refresh ABIs when proxy upgrades are observed.

## What an indexer must do per upgrade

1. **Pin the activation date** for each Linea release relevant to your indexed history — values in the table above are pinned to the Linea release notes as of 2026-05-08.
2. **Branch decoders by date** for: blob DA (Alpha v3.0), block time (Alpha v3.3), `finalized` tag (Alpha v3.5.0), 100% EVM coverage (Beta v2.0), Pectra opcodes (Beta v4.0), Fusaka (Beta v4.4), EIP-7702 (Beta v5.0), ENS resolver address (Beta v5.2).
3. **Update L1 verifier ABI** at major version bumps that touched the verifier (Alpha v2, Alpha v4.0, Beta v4.0, Beta v5.2).
4. **Detect tx exclusion** for pre-2.0 history if your application's "missing tx" diagnostics depend on it.
