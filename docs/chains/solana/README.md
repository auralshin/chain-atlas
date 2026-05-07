# Solana

**Status:** stub
**Family:** non-EVM L1
**Mainnet launch:** 2020-03-16 (mainnet beta)
**Native node clients:** agave (Anza fork of solana-labs validator), firedancer (Jump, in progress)

## Anchor facts (verify in deep dive)

- Slot-based, not block-based — slots are ~400ms; not every slot produces a block.
- No "events" in the EVM sense. Programs emit log messages and inner instructions; indexers parse instructions.
- Account model is account-with-data; programs are accounts with executable flag.
- Streaming via Geyser plugin (gRPC) is the production path — JSON-RPC polling does not scale.
- Confirmed → finalized status is explicit (`commitment` levels). Indexers must pick one; "processed" is unsafe for state.
- High volume — saturates Postgres faster than any chain in this repo. ClickHouse or columnar store typically required.

## To populate

Use [template](../_template/). Instruction-vs-event parsing is the headline section. Geyser/gRPC streaming gets its own subsection in `rpc-surface.md`.
