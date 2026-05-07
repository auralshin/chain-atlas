# Contributing

This repo is a research guide. Quality of references > quantity of chains.

## Adding a new chain

1. Copy `docs/chains/_template/` to `docs/chains/<chain-name>/`.
2. Fill in every file. If a section does not apply to this chain, write **why** rather than deleting it.
3. Add a row to the chain status table in the root [README.md](README.md).
4. Open a PR. The reviewer's job is to find unsourced claims.

## Sourcing rules

Every behavioral claim ("reorgs deeper than N blocks happen", "finality is X seconds", "method Y returns Z") must cite a primary source in `references.md`.

Acceptable primary sources:
- Chain specifications (yellow paper, design docs, EIP-equivalents)
- Official client documentation and source code
- On-chain governance proposals
- Public post-mortems
- Validator/sequencer dashboards and explorers

Stack Exchange, Twitter, and blog posts are acceptable as **supporting** evidence, never as the only source.

When client behavior differs from the spec, document both and link the relevant code or issue.

## Code blocks

Code is illustrative, not runnable. Use Rust by default for clarity. Show:

- The shape of the data type after decoding
- The minimal handler (block / log / instruction stream)
- What you'd insert into Postgres or ClickHouse

Do **not** include unrelated dependency setup, CLI argument parsing, or production-grade error handling. If a snippet exceeds ~40 lines, the explanation is too thin.

## Style

- Prefer tables for structured data (fork lists, RPC method matrices, finality timing).
- One file per concern. Do not merge `architecture.md` and `data-model.md` even if it feels redundant.
- Write for someone building their first indexer for this chain — assume they know Ethereum but not your chain.
- Date format: `YYYY-MM-DD`. Always absolute, never relative.
