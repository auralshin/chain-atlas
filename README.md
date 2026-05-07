<div align="center">

# chain-atlas

**A research guide to building blockchain indexers — correctly.**

EVM · ZK-EVM · non-EVM. Node architectures, finality models, RPC surfaces, fork timelines, storage tradeoffs, and the gotchas that bite indexer authors.

[**Concepts**](docs/concepts) · [**Chains**](docs/chains) · [**Template**](docs/chains/_template) · [**Contributing**](CONTRIBUTING.md)

</div>

---

> **Not a runtime.** There are no production indexers in this repo. It is a reference. Code blocks (Rust) illustrate patterns; they are not meant to compile as a single binary.

## Layout

- [`docs/concepts/`](docs/concepts) — cross-chain ideas: finality models, reorgs, storage tradeoffs, trace APIs, proxy resolution. Cited from every chain doc.
- [`docs/chains/`](docs/chains) — per-chain deep dives. Every chain follows the same template.
- [`docs/chains/_template/`](docs/chains/_template) — canonical structure. Read this first if contributing.

## Chain status

<div align="center">

| Chain | Family | Status |
|:--|:--|:--:|
| [Ethereum](docs/chains/ethereum) | EVM L1 | `in-progress` |
| [Optimism](docs/chains/optimism) | EVM L2 (OP Stack rollup) | `in-progress` |
| [Arbitrum](docs/chains/arbitrum) | EVM L2 (Nitro rollup) | `in-progress` |
| [Base](docs/chains/base) | EVM L2 (OP Stack rollup) | `in-progress` (delta on optimism/) |
| [BSC](docs/chains/bsc) | EVM sidechain (PoSA) | `in-progress` |
| [Polygon](docs/chains/polygon) | EVM sidechain (PoS, checkpointed) | `in-progress` |
| [zkSync Era](docs/chains/zksync-era) | ZK rollup (custom VM) | `in-progress` |
| [Scroll](docs/chains/scroll) | ZK-EVM rollup | `in-progress` |
| [Linea](docs/chains/linea) | ZK-EVM rollup | `in-progress` (delta on scroll/) |
| [StarkNet](docs/chains/starknet) | ZK rollup (Cairo VM) | `in-progress` |
| [Solana](docs/chains/solana) | non-EVM | `stub` |
| [Cosmos](docs/chains/cosmos) | non-EVM family (Tendermint/IBC) | `stub` |
| [Aptos](docs/chains/aptos) | non-EVM (Move) | `stub` |
| [Sui](docs/chains/sui) | non-EVM (Move, object-centric) | `stub` |

</div>

**Legend** — `stub`: placeholder, key facts only · `in-progress`: partial sections written · `deep`: full template populated and reviewed.

## How to use this repo

1. Read `docs/concepts/` to internalize the vocabulary and the cross-chain machinery.
2. Pick a chain in `docs/chains/`. Start with its `README.md` and walk through the template.
3. Cross-reference fork events in `forks-changelog.md` against the chain's primary specifications listed in `references.md`.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md). New chains must follow the structure in `docs/chains/_template/`. Every behavioral claim about a chain should be backed by a primary source in `references.md`.

## License

Dual-licensed. See [LICENSE](LICENSE) for full text.

- **Prose, tables, diagrams** — [CC-BY-4.0](https://creativecommons.org/licenses/by/4.0/). Reuse freely with attribution.
- **Code samples (Rust blocks, etc.)** — [MIT](https://opensource.org/licenses/MIT). Copy into your indexer; just keep the notice.

Copyright © 2026 auralshin.

<div align="center">
<sub>Built as a reference for engineers writing their first indexer for a given chain. Not a tutorial. Not a runtime.</sub>
</div>
