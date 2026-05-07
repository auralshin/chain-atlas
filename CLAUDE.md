# CLAUDE.md

Instructions for AI agents working in this repo. Human-facing docs are [README.md](README.md) and [CONTRIBUTING.md](CONTRIBUTING.md) — read those first; this file only adds what an agent needs that humans already know.

## What this repo is

A **research guide** to building blockchain indexers correctly. Per-chain deep dives + cross-chain concepts + illustrative Rust snippets in markdown.

**Non-goals.** Do not turn this into any of these without explicit user approval:
- A runnable indexer / Cargo workspace / production binary.
- A SaaS or multi-tenant query service.
- A general blockchain-101 tutorial. The reader is assumed to know Ethereum.

## Locked decisions (do not re-litigate)

| Decision | Value | Why |
|---|---|---|
| Audience | Engineers building their first indexer for a given chain | Determines depth and tone |
| Format | Markdown docs, Rust code blocks inline | Option (B) — doc-only, no compile |
| Skeleton crates | **None.** Rust lives in `skeleton.md` per chain | Avoids the abstraction-before-instance trap |
| Storage coverage | **Both** Postgres and ClickHouse, in every chain's `storage.md` | User wants tradeoff visibility |
| Multi-chain | One repo, one template, many chain folders | Not multi-tenant SaaS |
| Per-chain template | 10 files, canonical, do not merge | See `docs/chains/_template/` |
| First deep dive | Ethereum (template-defining) | Then Optimism + Solana |

If you think one of these is wrong, surface it before changing it.

## Two parallel workstreams

| Path | Owner | Purpose |
|---|---|---|
| `docs/` | This (research) agent | Source of truth — markdown, Rust snippets, references |
| `web/` | Separate agent (Next.js + react-three-fiber) | Visual companion that consumes `docs/` content |

**Boundary rules:**
- Do not edit `web/` from a docs task, and vice versa, unless the user explicitly bridges them.
- The web app reads from `web/lib/*.ts` (e.g. `chains.ts`, `ethereum-forks.ts`). If a docs change should propagate visually, mention it in your reply — don't silently sync.
- If `docs/` and `web/lib/*.ts` disagree on a fact, **`docs/` is canonical** and the web copy must be updated.

## Per-chain deep-dive rules

Every chain folder contains the 10 files from `docs/chains/_template/`. **Never delete a file because the section "doesn't apply"** — write *why* it doesn't apply (one paragraph) and move on. Empty sections fail review.

Status lifecycle (tracked in root README chain table):
- `stub` — README only, anchor facts marked **verify in deep dive**, no other files populated.
- `in-progress` — some template files populated, not yet reviewed.
- `deep` — all 10 files populated, every behavioral claim cited in `references.md`, user has reviewed.

Promoting a stub fact into a populated file requires citing a primary source. If you can't find one, mark the claim **`{{unsourced}}`** rather than dropping it — leave the next agent a breadcrumb.

## Sourcing (stricter than CONTRIBUTING.md spells out)

- Every numeric claim (reorg depth, finality time, slot duration, fee parameter) needs a primary source.
- Date format `YYYY-MM-DD`, always absolute. Never "last year" / "recently".
- When client behavior diverges from spec, document **both** and link the code or issue.
- Stack Exchange / Twitter / blog posts are supporting only, never sole source.
- If you assert behavior you didn't verify in this turn, prefix with **`{{unsourced}}`**.

## Rust code-block style

- ≤ 40 lines per snippet. If it's longer, the prose is too thin.
- Show the **decoded data shape**, the **minimal handler**, and **what you'd insert into PG or CH**. Nothing else.
- No `clap`, no `tracing` setup, no `main()` boilerplate, no production-grade error handling.
- `Result<_, anyhow::Error>` is fine; full error enums are not.
- It's OK if snippets don't compile in isolation — they're for reading, not running. But the **types must be real** (real crate names, real method signatures). Made-up SDK calls are worse than missing code.

## Today's anchor

User-supplied date in this conversation: **2026-05-08**.

Ethereum is post-Pectra (2025-05-07). When writing about EVM behavior, you must reflect:
- EIP-7702 (set-code transactions, EOAs can attach code)
- EIP-7251 (max effective balance 2048 ETH)
- EIP-4844 blobs are well-established, not novel
- The Merge (2022-09-15) is ancient history; PoW Ethereum is a forks-changelog entry, not a current state

Glamsterdam (EIP-7732 enshrined PBS, etc.) is **scheduled** for H1 2026 — describe as upcoming, not active, unless the user confirms it shipped.

## Things to never do without asking

- Add a `Cargo.toml` / promote skeleton.md into a real crate.
- Delete or rename a template file.
- Reorganize `docs/chains/<chain>/` away from the canonical 10-file layout.
- Touch `web/` during a docs task.
- Commit. The user commits when they want to.
- Push. Ever, unless asked by name.
- Run `npm install` / `cargo build` / any package fetch — this repo has no compile target by design.

## Things to do proactively

- When updating a chain's facts, scan `web/lib/*.ts` for stale copies and **flag them in your reply** (don't auto-edit unless asked).
- When you encounter a `{{unsourced}}` marker and you have a real source, replace it.
- When a stub's anchor fact looks wrong against current state, flag it — don't silently rewrite.
- Keep the chain status table in the root README in sync when you promote stub → in-progress → deep.

## Memory note

User maintains Claude Code auto-memory. The single load-bearing entry today is: *this is a research guide, not a production indexer.* Anything written in memory that contradicts that should be re-read before acting on.
