# References

## Source code

- **`agave`** — [github.com/anza-xyz/agave](https://github.com/anza-xyz/agave) — Anza's fork of `solana-labs/solana`; the canonical validator since 2024.
- **`solana-labs/solana`** — [github.com/solana-labs/solana](https://github.com/solana-labs/solana) — original; pre-2024 history. Many references still point here.
- **`firedancer`** — [github.com/firedancer-io/firedancer](https://github.com/firedancer-io/firedancer) — Jump's high-performance validator (C/C++).
- **SPL programs** — [github.com/solana-program/](https://github.com/solana-program/) (multiple repos for SPL Token, Token-2022, Associated Token Account, Stake Pool, etc.) {{unsourced: confirm path}}.

## Specifications

- **Solana docs** — [docs.solana.com](https://docs.solana.com/) — official; covers consensus, account model, programs, RPC, Geyser.
- **`solana-foundation/specs`** — [github.com/solana-foundation/specs](https://github.com/solana-foundation/specs) {{unsourced: confirm correct repo}} — protocol specifications.
- **Solana Cookbook** — [solanacookbook.com](https://solanacookbook.com/) — practical guides.

## Type / SDK libraries

- **`solana-sdk`** (Rust) — [crates.io/crates/solana-sdk](https://crates.io/crates/solana-sdk) — Pubkey, Signature, transaction types.
- **`@solana/web3.js`** (TypeScript) — official TS SDK.
- **`solders`** (Python) — Rust-backed Python SDK.
- **`solana-go`** (Go) — community Go SDK.

## Geyser / streaming

- **Yellowstone gRPC** — [github.com/rpcpool/yellowstone-grpc](https://github.com/rpcpool/yellowstone-grpc) — Triton's open Geyser plugin and gRPC schema.
- **`yellowstone-grpc-proto`** / **`yellowstone-grpc-client`** (Rust) — canonical client crates.
- **Geyser plugin interface** — [`solana-geyser-plugin-interface`](https://docs.rs/solana-geyser-plugin-interface) — for self-hosting custom plugins.

## Geyser providers

- **Triton** — [triton.one](https://triton.one/) — high-throughput Geyser.
- **Helius** — [helius.dev](https://helius.dev/) — Geyser + enriched APIs (parsed instructions, NFT metadata).
- **Jito** — [jito.network](https://jito.network/) — MEV-aware bundle submission + Geyser.
- **Shyft** — [shyft.to](https://shyft.to/) — alternative.
- **dRPC**, **QuickNode**, **Alchemy** — also offer Geyser tiers.

## Block explorers

- **Solana Explorer** — [explorer.solana.com](https://explorer.solana.com/) — official.
- **Solscan** — [solscan.io](https://solscan.io/) — community-favorite; richer instruction parsing.
- **Solana Beach** — [solanabeach.io](https://solanabeach.io/) — validator-focused.
- **SolanaFM** — [solana.fm](https://solana.fm/) — alternative.

## Operational

- **Solana Status** — [status.solana.com](https://status.solana.com/) — outage notifications.
- **Solana Beach** — [solanabeach.io](https://solanabeach.io/) — also functions as a network monitor.
- **Solana Compass** — [solanacompass.com](https://solanacompass.com/) — validator and stake distribution metrics.

## Anchor and program standards

- **Anchor framework** — [anchor-lang.com](https://www.anchor-lang.com/) — most Solana programs are written with Anchor.
- **IDLs (Anchor)** — programs publish a JSON IDL describing their instructions, accounts, and events. Indexers consume IDLs to decode automatically.
- **OpenBook / Serum / Jupiter / Drift / Marginfi etc.** — major dApps; their GitHub repos contain IDLs for indexer use.

## BigQuery / archive

- **Solana on Google BigQuery** — public dataset of historical Solana txs and blocks. Useful for deep historical backfill without running an archive validator.

## Where this doc is incomplete

The following claims carry `{{unsourced}}` markers:

- Exact slot duration and tick rate (`~400ms` is approximate).
- Epoch length (claimed 432,000 slots).
- Maximum CPI depth (claimed 4).
- Address Lookup Table mainnet activation date.
- Token-2022 launch date.
- `return_data` syscall activation.
- Specific halt incidents and durations (2022 halts, etc.).
- Validator count (claimed 1500-2000+).
- Solana specs repo path.

Verify against the agave source (`feature_set.rs`), Solana docs, and Solana Foundation announcements before promoting this doc to `deep`.
