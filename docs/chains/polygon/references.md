# References

## Source code

- **Bor** — [github.com/maticnetwork/bor](https://github.com/maticnetwork/bor) — geth-fork, EVM execution layer.
- **Heimdall** — [github.com/maticnetwork/heimdall](https://github.com/maticnetwork/heimdall) — Tendermint-Cosmos consensus layer.
- **Heimdall-v2** — [github.com/0xPolygon/heimdall-v2](https://github.com/0xPolygon/heimdall-v2) — newer rewrite (Cosmos SDK v0.50+).

## L1 contracts

- **Contracts repo** — [github.com/maticnetwork/contracts](https://github.com/maticnetwork/contracts) — `RootChain`, `RootChainManager`, `StateSender`, `StakeManager`, `DepositManager` (Plasma).
- **PoS Portal** — [github.com/maticnetwork/pos-portal](https://github.com/maticnetwork/pos-portal) — bridge UX layer.

## Documentation

- **Polygon Wiki / Docs** — [docs.polygon.technology](https://docs.polygon.technology/) — official; covers architecture, validators, state syncs, checkpoints.
- **Heimdall REST API** — same site; Cosmos REST + Tendermint RPC.

## Block explorers

- **Polygonscan** — [polygonscan.com](https://polygonscan.com/) — Etherscan-family. Standard ETH receipts.
- **Heimdall explorer** — [polygonscan.com/heimdall](https://polygonscan.com/heimdall) — checkpoint and validator views.

## Operational

- **Polygon Status** — [status.polygon.technology](https://status.polygon.technology/).
- **Validator dashboard** — [staking.polygon.technology](https://staking.polygon.technology/) — POL staking UI.

## Token migration

- **POL migration overview** — [polygon.technology blog](https://polygon.technology/) {{unsourced: pin specific post for the migration}}.
- **Migration contract** — `PolygonMigration` on Ethereum mainnet — verify address from official docs at deep-dive time.

## Where this doc is incomplete

The following claims carry `{{unsourced}}` markers and need primary-source backing before promotion to `deep`:

- Sprint length (claimed 64 blocks) and span length (claimed 6400 blocks).
- Active validator count.
- Block time (claimed 2 s — was historically ~2.2 s; verify the upgrade that changed this).
- Bor fork heights and dates: London, Delhi, Indore, Bhilai, Napoli, Pragati.
- Heimdall block time (claimed ~5 s).
- Heimdall version detection at runtime — REST shape changes between v1 and v2.
- Whether Polygon's Cancun-equivalent fork includes EIP-4788 / beacon-root precompile (probably adapted, given Polygon has no Ethereum beacon chain).
- Validator addresses mapping bor key to heimdall key.
- Specific reorg incidents and their depths (the canonical 128-block claim — find the actual incident retrospective).
- Heimdall halt incidents and durations.
- POL migration date (claimed 2024-09-04).
- Whether bor exposes `safe`/`finalized` block tags (claimed post-Bhilai).

Verify against the bor + heimdall source, official docs, and incident retrospectives before promoting this doc.
