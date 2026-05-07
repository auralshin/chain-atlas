# References

Inherits all references from [optimism/references.md](../optimism/references.md). Base-specific sources below.

## Base-specific specifications

- **Base docs** — [docs.base.org](https://docs.base.org/) — official.
- **Superchain Registry** — [github.com/ethereum-optimism/superchain-registry/tree/main/superchain/configs/mainnet](https://github.com/ethereum-optimism/superchain-registry/tree/main/superchain/configs/mainnet). **Note:** Base is NOT included in the canonical Superchain Registry as of this writing — there is no `base.toml`. Base operates as an OP Stack chain outside the registry; canonical Base chain parameters live in op-node defaults and in [base-org/contract-deployments](https://github.com/base-org/contract-deployments).

## Operational

- **Base status page** — [status.base.org](https://status.base.org/)
- **Basescan explorer** — [basescan.org](https://basescan.org/) — Etherscan-family L2 explorer for Base.
- **Public RPC** — [mainnet.base.org](https://mainnet.base.org/) — Coinbase-operated, rate-limited; not for production indexing.

## Bridge (L1 contracts on Ethereum mainnet)

Base's L1 contracts have addresses **distinct** from OP Mainnet's. Pull the current canonical set from the superchain-registry. The contract source code is the same OP Stack source as Optimism — only deployment addresses differ.

## Verified in this pass

- **Fork activation timestamps** Canyon → Isthmus pulled from `base/node` releases and op-geth release notes. Base activates **at the same UTC timestamp as OP Mainnet** for each upgrade.
- **Coinbase Cloud as sole sequencer / batcher / proposer** — confirmed via Base docs and outage postmortems.
- **Fee revenue split** — greater of 2.5% sequencer revenue or 15% net on-chain sequencer revenue to OP Collective per the [August 2023 agreement](https://www.optimism.io/blog/welcoming-base).
- **2023-09-05 outage**: 43 min, internal infrastructure issue. **2025-08-05 outage**: 33 min, Conductor failover misconfig.

## Still incomplete

- **Fault proof rollout timeline on Base** — verify against the Base docs at deep-dive time.
- **Public RPC URLs and status page URLs** — drift-prone; check at deep-dive time.
- **Jovian fork activation on Base** — assumed same as OP Mainnet's 2025-12-02 16:00:01 UTC, but unverified.
- **Archive node disk size** — drift-prone; do not pin a GB figure in this doc.

## Inherited references

For every aspect not listed above (OP Stack specs, derivation pipeline, deposit txs, predeploys, fault proofs mechanism, type libraries, etc.) the canonical sources are in [optimism/references.md](../optimism/references.md).
