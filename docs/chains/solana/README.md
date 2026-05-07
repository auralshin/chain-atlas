# Solana

**Status:** in-progress (template populated — pending review)
**Family:** non-EVM L1 — Proof-of-History + Proof-of-Stake (Tower BFT)
**Mainnet launch:** 2020-03-16 (mainnet beta)
**Native node clients:** [`agave`](https://github.com/anza-xyz/agave) (Anza fork of `solana-labs/solana`, since 2024), [`firedancer`](https://github.com/firedancer-io/firedancer) (Jump, partially live as Frankendancer 2024–2025).

## TL;DR for indexers

- **Account-based, not state-trie-based.** Every entity is an account at a public key (32-byte ed25519). State is the union of account states; no MPT.
- **Programs are accounts with `executable=true`.** Smart contracts live at fixed program-id addresses; instance data lives in separate accounts.
- **Transactions enumerate every account they touch.** No implicit account access. Each instruction names exactly which program + which accounts (read or write) it operates on.
- **No events in the EVM sense.** Programs emit:
  - `msg!()` text logs (visible in tx logs, free-form strings).
  - CPI logs / Anchor-style program logs (binary-encoded, discriminator-prefixed).
  - Inner instructions (the call tree of CPIs).
  Indexers reconstruct "events" by parsing **logs + inner instructions + account state diffs**, not by scanning a `Log` table.
- **Confirmation is explicit.** `processed` / `confirmed` / `finalized` commitment levels. **Never index on `processed` for state-bearing data.**
- **High volume.** Solana produces a slot every ~400 ms with up to 64 transactions per slot. Postgres saturates fast; ClickHouse or a columnar store is typical.
- **Streaming via Geyser plugin (gRPC).** JSON-RPC polling does not scale to mainnet; production indexers use Geyser-tier providers (Triton, Jito, Helius, etc.) or self-host with a Geyser plugin.

## Read order

1. [architecture.md](architecture.md) — slots, leaders, Tower BFT, account model
2. [data-model.md](data-model.md) — accounts, programs, instructions, CPIs, transactions
3. [reorgs-finality.md](reorgs-finality.md) — three commitment levels, reorgs at fork boundaries
4. [rpc-surface.md](rpc-surface.md) — JSON-RPC + Geyser gRPC + WebSocket subscriptions
5. [forks-changelog.md](forks-changelog.md) — feature gates, Token-2022, validator client transitions
6. [gotchas.md](gotchas.md) — log parsing, CPI traversal, account-state-as-events, lookup tables
7. [storage.md](storage.md) — ClickHouse-first; Postgres for canonical metadata
8. [skeleton.md](skeleton.md) — illustrative Rust
9. [references.md](references.md) — primary sources

## What is *not* in scope here

- **Sealevel-only programs (specific dApps)** — the indexer patterns transfer; specific program IDLs (Serum, Raydium, Jupiter, Drift, Marginfi, etc.) are application-specific decode work.
- **Solana's economic security parameters** — interesting but not indexer-relevant.
- **Solana Pay / Solana Mobile / etc.** — application layer.
- **Eclipse, Sonic, MagicBlock, etc.** — Solana-VM L2s; same SVM, different validator sets and DA. Indexer patterns transfer; addresses do not.
