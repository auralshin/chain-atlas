# Sui

**Status:** stub
**Family:** non-EVM L1 — Move (object-centric dialect)
**Mainnet launch:** 2023-05-03
**Native node clients:** sui-node

## Anchor facts (verify in deep dive)

- Move language, but **object-centric** — fundamentally different from Aptos's account-based Move. State is a graph of objects, not balances on accounts.
- Two consensus paths: fast-path for owned-object transactions (Mysticeti / Bullshark family), full consensus for shared-object transactions.
- Events emitted from Move modules; subscribe via JSON-RPC or gRPC.
- Checkpoints, not blocks, are the primary unit for indexers — a checkpoint contains a batch of executed transactions.

## To populate

Use [template](../_template/). Object model and checkpoint-oriented streaming get headline treatment. **Do not** assume Aptos knowledge transfers — the data model is genuinely different.
