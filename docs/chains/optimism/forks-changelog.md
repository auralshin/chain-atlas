# Forks and Changelog

OP Stack network upgrades, listed in order. Each upgrade has an L1 *and* L2 activation timestamp; for clarity this table uses the L2 activation block / timestamp on **OP Mainnet** (other OP Stack chains apply the same upgrade at chain-specific timestamps — see [base/forks-changelog.md](../base/forks-changelog.md)).

## Eras

| Era | Span | Indexer impact |
|---|---|---|
| **OVM 1.0** | 2021-12 → 2021-11 (testnet→mainnet rollout) {{unsourced: tighten}} | Custom VM, custom data model. **Out of scope for this guide.** |
| **OVM 2.0** | 2021-11 → 2023-06-06 | Geth-equivalent VM under a translator. Distinct receipts, distinct tx types. **Out of scope for this guide.** |
| **Bedrock+** | 2023-06-06 → present | EVM-equivalent execution, deposit tx type, derivation pipeline. **This guide covers this era.** |

If you need pre-Bedrock data, the canonical recommendation is to load the [pre-Bedrock historical archive](https://blog.oplabs.co/introducing-optimism-bedrock/) as a one-time snapshot and start streaming from Bedrock activation.

## Bedrock-era upgrades

Activation timestamps below are pulled from the canonical [`op.toml` in the superchain-registry](https://github.com/ethereum-optimism/superchain-registry/blob/main/superchain/configs/mainnet/op.toml) — the single source of truth for OP Mainnet hardfork activations.

| Upgrade | L2 activation (OP Mainnet) | Source | Indexer impact |
|---|---|---|---|
| **Bedrock** | 2023-06-06 | [Bedrock launch post](https://blog.oplabs.co/introducing-optimism-bedrock/) | Foundational; everything in this guide assumes Bedrock+. |
| **Regolith** | 2023-06-06 (with Bedrock) | [Specs: Regolith](https://specs.optimism.io/protocol/regolith/overview.html) | Minor: deposit tx gas accounting fix. Receipt fields stabilized. |
| **Canyon** | 2024-01-11 17:00:01 UTC | [op.toml](https://github.com/ethereum-optimism/superchain-registry/blob/main/superchain/configs/mainnet/op.toml) | Shapella-equivalent: withdrawals on L2 (predeploy), `PUSH0`, fee recipient updates. |
| **Delta** | 2024-02-22 00:00:00 UTC | [op.toml](https://github.com/ethereum-optimism/superchain-registry/blob/main/superchain/configs/mainnet/op.toml) | **Span batches** — new batch encoding that compresses ranges of L2 blocks. Indexers that decode batches must handle both legacy + span. |
| **Ecotone** | 2024-03-14 00:00:01 UTC | [op.toml](https://github.com/ethereum-optimism/superchain-registry/blob/main/superchain/configs/mainnet/op.toml) | Cancun-Deneb equivalent: **EIP-4844 blobs** for batch DA, transient storage (TSTORE/TLOAD), MCOPY, BLS precompiles, beacon-block-root precompile. **`op-batcher` switches from calldata to blobs.** |
| **Fjord** | 2024-07-10 16:00:01 UTC | [op.toml](https://github.com/ethereum-optimism/superchain-registry/blob/main/superchain/configs/mainnet/op.toml) | **RIP-7212** (P256 secp256r1 verify precompile), **Brotli compression** for batch channels (replaces zlib), FastLZ-based L1 cost function, gas accounting tweaks. |
| **Granite** | 2024-09-11 16:00:01 UTC | [op.toml](https://github.com/ethereum-optimism/superchain-registry/blob/main/superchain/configs/mainnet/op.toml) | KZG point-evaluation precompile gas reduction, fault-proof improvements (max-bond, anchor state), bn254 precompile gas changes. |
| **Holocene** | 2025-01-09 18:00:01 UTC | [op.toml](https://github.com/ethereum-optimism/superchain-registry/blob/main/superchain/configs/mainnet/op.toml) | **Derivation determinism**: invalid batches no longer halt derivation but are dropped. **Configurable EIP-1559 parameters** per chain. |
| **Isthmus** | 2025-05-09 16:00:01 UTC | [op.toml](https://github.com/ethereum-optimism/superchain-registry/blob/main/superchain/configs/mainnet/op.toml) | **EIP-7702** (set-code transactions), other Pectra L1 changes ported to L2. |
| **Jovian** | 2025-12-02 16:00:01 UTC | [op.toml](https://github.com/ethereum-optimism/superchain-registry/blob/main/superchain/configs/mainnet/op.toml) | Continued Ethereum-equivalence porting; specifics in op-stack specs. |

If you are reading this after **2026-05-08** (the date this guide was last anchored), at least one further upgrade may have shipped. Check the [op-stack/specs](https://specs.optimism.io/) and the [superchain-registry](https://github.com/ethereum-optimism/superchain-registry/) for the canonical list.

## Fault proofs rollout

Independent of the named upgrades:

| Date | Event |
|---|---|
| **2024-06-10** {{unsourced: confirm}} | **Permissioned fault proofs** on OP Mainnet — `DisputeGameFactory` deployed; output roots posted via dispute games but only a permissioned set can challenge. |
| **{{unsourced}}** | **Permissionless fault proofs** — anyone can challenge. (Verify status on OP Mainnet at deep-dive time.) |

`L2OutputOracle` was retained for some time as a fallback / for chains that hadn't migrated. Indexers that index output roots must handle **both** sources during the migration window.

## Per-chain upgrade timing

OP Stack upgrades are scheduled per-chain. **OP Mainnet, Base, Mode, Zora, etc. usually share an L2 activation timestamp** for a given upgrade, but not always. Always read the activation timestamp from the chain's `SystemConfig` or the rollup config in `op-node`'s superchain registry — never hardcode dates from another chain.

The canonical superchain registry: [`ethereum-optimism/superchain-registry`](https://github.com/ethereum-optimism/superchain-registry).

## What an indexer must do per upgrade

For each fork, an indexer needs:

1. **Activation timestamp** for the chain it's indexing — fetched from the rollup config or hardcoded with the source cited.
2. **Decoder versioning** — channel/batch decoders, deposit tx parsers, fee formulas all change. Maintain version-per-block, not version-per-chain.
3. **Backfill plan** — if the indexer stores derived facts (e.g. L1 fee in USD), recompute the affected blocks when a fork changes the formula.

Hardest historical example: the **Ecotone L1 fee formula change**. Pre-Ecotone L1 fee = `l1GasUsed * l1GasPrice * scalar`. Post-Ecotone L1 fee uses blob base fee + base fee with separate scalars. Receipts contain the inputs but the *interpretation* depends on which fork was active.
