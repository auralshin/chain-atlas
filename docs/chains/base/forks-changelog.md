# Forks and Changelog

Base inherits the OP Stack upgrade sequence (Bedrock → Canyon → Delta → Ecotone → Fjord → Granite → Holocene → Isthmus). **Base activates each upgrade at the same UTC timestamp as OP Mainnet** — they are coordinated within the Superchain.

> Earlier versions of this doc claimed "Base activations lag OP Mainnet by a few days." That's **wrong** — verified against [`base/node`](https://github.com/base/node/releases) and op-geth release notes, the activation timestamps are identical.

## Base-specific upgrade timeline

Activation timestamps are pulled from the canonical Base node config / op-geth release notes. Note: Base is **not** in the canonical [`superchain-registry`](https://github.com/ethereum-optimism/superchain-registry); Base's chain parameters live in [`base/node`](https://github.com/base/node) and in the op-node defaults bundled with releases.

| Upgrade | Activation (UTC, same as OP Mainnet) | Unix timestamp | Indexer impact |
|---|---|---|---|
| **Genesis** | 2023-08-09 (mainnet launch) | — | — |
| **Canyon** | **2024-01-11 17:00:01 UTC** | 1704992401 | Shapella-equivalent: withdrawals on L2 (predeploy), `PUSH0`, fee recipient updates |
| **Delta** | **2024-02-22 00:00:00 UTC** | 1708560000 | **Span batches** — new batch encoding compresses ranges of L2 blocks |
| **Ecotone** | **2024-03-14 00:00:01 UTC** | 1710374401 | Cancun-Deneb equivalent: **EIP-4844 blobs** for batch DA, transient storage, MCOPY, BLS precompiles, beacon-block-root precompile. **`op-batcher` switches from calldata to blobs** |
| **Fjord** | **2024-07-10 16:00:01 UTC** | 1720627201 | **RIP-7212** (P256 secp256r1 verify), **Brotli compression** for batch channels, FastLZ-based L1 cost function |
| **Granite** | **2024-09-11 16:00:01 UTC** | 1726070401 | KZG point-evaluation precompile gas reduction, fault-proof improvements, bn254 precompile gas changes |
| **Holocene** | **2025-01-09 18:00:01 UTC** | 1736445601 | **Derivation determinism**: invalid batches no longer halt derivation but are dropped. **Configurable EIP-1559 parameters** per chain |
| **Isthmus** | **2025-05-09 16:00:01 UTC** | 1746806401 | **EIP-7702** (set-code transactions), other Pectra L1 changes ported to L2 |
| **Jovian** | 2025-12-02 16:00:01 UTC (per [op.toml](https://github.com/ethereum-optimism/superchain-registry/blob/main/superchain/configs/mainnet/op.toml); Base coordinates) {{unsourced: confirm Base activation matches}} | — | Continued Ethereum-equivalence porting |

The semantic content of each upgrade is documented in [optimism/forks-changelog.md](../optimism/forks-changelog.md) — Base does not introduce upgrade content of its own.

## Pre-genesis history

There is none in scope for indexers. Base's testnet era (Goerli, Sepolia) is irrelevant to mainnet indexers and has its own contract addresses if you do need to test.

## Operational changes (not protocol forks)

Periodically Coinbase performs operational changes (sequencer software upgrades, batcher reconfiguration) that are **not** consensus-relevant but can change observable behavior:

- **Batch posting cadence changes** — Coinbase has tuned batch frequency a few times to reduce L1 cost; the `safe` head's lag distribution shifts when this happens.
- **Sequencer software version bumps** — usually invisible, occasionally introduce reorg-rate spikes.
- **Backup-sequencer / Conductor reconfigurations** — Base uses a "Conductor" tool to manage sequencer failover. The 2025-08-05 outage was traced to a Conductor misconfiguration that routed traffic to a non-production sequencer.

These are documented in Coinbase's status page rather than the protocol spec. Indexers should not encode behavior dependencies on them.

## What an indexer must do per upgrade

Same checklist as OP Mainnet (see [optimism/forks-changelog.md#what-an-indexer-must-do-per-upgrade](../optimism/forks-changelog.md#what-an-indexer-must-do-per-upgrade)). The rollup config Base ships in `op-node` carries the activation timestamps — pin them at startup. Because Base shares timestamps with OP Mainnet, a multi-chain indexer can use a single timestamp source for the Superchain (excluding the few chains that diverge, like Mode for some upgrades).
