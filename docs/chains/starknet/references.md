# References

## Specifications

- **`starknet-specs`** — [github.com/starkware-libs/starknet-specs](https://github.com/starkware-libs/starknet-specs) — canonical JSON-RPC spec. Versioned releases.
- **Cairo language docs** — [book.cairo-lang.org](https://book.cairo-lang.org/) — Cairo 1/2 language and contract programming model.
- **StarkNet docs** — [docs.starknet.io](https://docs.starknet.io/) — official; covers architecture, AA, fee model, RPC, classes vs contracts.
- **OpenZeppelin Cairo contracts** — [github.com/OpenZeppelin/cairo-contracts](https://github.com/OpenZeppelin/cairo-contracts) — reference account contracts and standards.

## Node clients

- **`pathfinder`** — [github.com/eqlabs/pathfinder](https://github.com/eqlabs/pathfinder) — Equilibrium, Rust.
- **`juno`** — [github.com/NethermindEth/juno](https://github.com/NethermindEth/juno) — Nethermind, Go.
- **`papyrus`** — [github.com/starkware-libs/papyrus](https://github.com/starkware-libs/papyrus) — StarkWare, Rust; reference implementation.

The sequencer is closed-source.

## Type / SDK libraries

- **`starknet-rs`** — [github.com/xJonathanLEI/starknet-rs](https://github.com/xJonathanLEI/starknet-rs) — canonical Rust SDK. Tracks RPC spec; exposes typed clients and provider abstractions.
- **`starknet.js`** — [github.com/starknet-io/starknet.js](https://github.com/starknet-io/starknet.js) — TypeScript SDK.
- **`starknet.go`** — Nethermind / community Go SDK.
- **`starknet-py`** — Python SDK.

For Rust indexers, use `starknet-rs` as the foundation. The `Felt` type and the `Transaction` enum from `starknet-core` are the canonical types — do not roll your own.

## L1 contracts

- **`Starknet Core Contract`** — on Ethereum mainnet at `0xc662c410C0ECf747543f5bA90660f6ABeBD9C8c4` {{unsourced: confirm}}.
- Token bridge contracts — separate L1 contracts per token.
- Source: [github.com/starkware-libs/cairo-lang](https://github.com/starkware-libs/cairo-lang) (older; the L1 Solidity may be elsewhere) {{unsourced: confirm L1 contract source location}}.

## Block explorers

- **Voyager** — [voyager.online](https://voyager.online/) — most popular StarkNet explorer.
- **StarkScan** — [starkscan.co](https://starkscan.co/) — alternative.
- **ViewBlock** — [viewblock.io/starknet](https://viewblock.io/starknet) — alternative.

All three handle Cairo 0 + Cairo 1, fee unit display, AA, etc.

## Operational

- **StarkNet status** — [status.starknet.io](https://status.starknet.io/) {{unsourced: confirm URL}}.
- **L2BEAT StarkNet page** — [l2beat.com/scaling/projects/starknet](https://l2beat.com/scaling/projects/starknet) — independent monitor.

## Public RPC

Many providers offer StarkNet endpoints: Blast, Infura, Alchemy, dRPC, ChainStack. Free tiers are rate-limited; archive tiers are paid.

## Where this doc is incomplete

The following claims carry `{{unsourced}}` markers and need primary-source backing:

- Exact dates / heights for V0.10, V0.11, V0.12, V0.13.x, V0.14 protocol versions.
- RPC spec progression (v0.5 → v0.6 → v0.7 → v0.8) and per-spec method changes.
- Cairo 0 declaration deprecation timing.
- STRK fee token rollout date and L1 contract addresses.
- Current L2 block time (claimed ~30 s historically; reduced over time).
- Subscription method support per node client (`pathfinder` vs `juno` vs `papyrus`).
- `Starknet Core Contract` L1 address.
- Force-include path for sequencer downtime.
- ETH and STRK fee token contract addresses on L2.
- `replace_class` syscall semantics and how often it's used.

Verify against the StarkNet docs, the `starknet-specs` repo, and node-client release notes before promoting this doc to `deep`.
