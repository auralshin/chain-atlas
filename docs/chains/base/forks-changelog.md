# Forks and Changelog

Base inherits the OP Stack upgrade sequence (Bedrock → Canyon → Delta → Ecotone → Fjord → Granite → Holocene → Isthmus). Activation **timestamps** are Base-specific and typically lag OP Mainnet by a few days.

## Base-specific upgrade timeline

Always verify against [`superchain-registry/superchain/configs/mainnet/base.toml`](https://github.com/ethereum-optimism/superchain-registry/blob/main/superchain/configs/mainnet/base.toml) {{unsourced: confirm path}}.

| Upgrade | Base activation | OP Mainnet activation | Indexer impact |
|---|---|---|---|
| **Genesis** | 2023-08-09 | 2023-06-06 (Bedrock) | — |
| **Canyon** | {{unsourced: 2024-01 — verify}} | 2024-01-11 {{unsourced}} | Same as OP |
| **Delta** | {{unsourced: 2024-02 — verify}} | 2024-02-22 {{unsourced}} | Same as OP |
| **Ecotone** | {{unsourced: 2024-03 — verify}} | 2024-03-14 | Same as OP — blob batches |
| **Fjord** | {{unsourced: 2024-07 — verify}} | 2024-07-10 {{unsourced}} | Same as OP — Brotli, RIP-7212 |
| **Granite** | {{unsourced: 2024-09 — verify}} | 2024-09-11 {{unsourced}} | Same as OP |
| **Holocene** | {{unsourced: 2025 Q1}} | {{unsourced: 2025 Q1}} | Same as OP |
| **Isthmus** | {{unsourced: 2025}} | {{unsourced: 2025}} | EIP-7702 + Pectra L2 changes |

The semantic content of each upgrade is documented in [optimism/forks-changelog.md](../optimism/forks-changelog.md) — Base does not introduce upgrade content of its own.

## Pre-genesis history

There is none in scope for indexers. Base's testnet era (Goerli, Sepolia) is irrelevant to mainnet indexers and has its own contract addresses if you do need to test.

## Operational upgrades not in the canonical changelog

Periodically Coinbase performs operational changes (sequencer software upgrades, batcher reconfiguration) that are **not** consensus-relevant but can change observable behavior:

- **Batch posting cadence changes** — Coinbase has tuned batch frequency a few times to reduce L1 cost; the `safe` head's lag distribution shifts when this happens.
- **Sequencer software version bumps** — usually invisible, occasionally introduce reorg-rate spikes.

These are documented in operational changelogs (Coinbase's status page) rather than the protocol spec. Indexers should not encode behavior dependencies on them.

## What an indexer must do per upgrade

Same checklist as OP Mainnet (see [optimism/forks-changelog.md#what-an-indexer-must-do-per-upgrade](../optimism/forks-changelog.md#what-an-indexer-must-do-per-upgrade)). The rollup config Base ships in `op-node` carries the activation timestamps — pin them at startup.
