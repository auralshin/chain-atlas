# References

Inherits all references from [optimism/references.md](../optimism/references.md). Base-specific sources below.

## Base-specific specifications

- **Base docs** — [docs.base.org](https://docs.base.org/) — official.
- **Superchain Registry, Base config** — [github.com/ethereum-optimism/superchain-registry/tree/main/superchain/configs/mainnet](https://github.com/ethereum-optimism/superchain-registry/tree/main/superchain/configs/mainnet) — the canonical chain parameters file. {{unsourced: confirm exact path for `base.toml`}}

## Operational

- **Base status page** — [status.base.org](https://status.base.org/) {{unsourced: confirm URL}}
- **Basescan explorer** — [basescan.org](https://basescan.org/) — Etherscan-family L2 explorer for Base.
- **Public RPC** — [mainnet.base.org](https://mainnet.base.org/) — Coinbase-operated, rate-limited; not for production indexing.

## Bridge (L1 contracts on Ethereum mainnet)

Base's L1 contracts have addresses **distinct** from OP Mainnet's. Pull the current canonical set from the superchain-registry. The contract source code is the same OP Stack source as Optimism — only deployment addresses differ.

## Where this doc is incomplete

The following Base-specific claims carry `{{unsourced}}` markers in this folder:

- Exact activation timestamps for Canyon, Delta, Ecotone, Fjord, Granite, Holocene, Isthmus on Base mainnet.
- Coinbase's role and identity as the sole sequencer / proposer / batcher.
- Fee revenue split between Coinbase and OP Collective.
- Fault proof rollout timeline on Base.
- Specific outage incidents and their correspondence to indexer impact.
- Public RPC URLs and status page URLs.

Verify against the superchain-registry, Base docs, and Coinbase's L2 governance writeups before promoting this doc to `deep`.

## Inherited references

For every aspect not listed above (OP Stack specs, derivation pipeline, deposit txs, predeploys, fault proofs mechanism, type libraries, etc.) the canonical sources are in [optimism/references.md](../optimism/references.md).
