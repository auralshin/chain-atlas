# References

## Source code

- **bsc** (client) — [github.com/bnb-chain/bsc](https://github.com/bnb-chain/bsc) — geth fork, single binary.
- **Parlia consensus** — [github.com/bnb-chain/bsc/tree/develop/consensus/parlia](https://github.com/bnb-chain/bsc/tree/develop/consensus/parlia) — PoSA consensus engine. Authoritative for `extraData` parsing.
- **System contracts** — [github.com/bnb-chain/bsc-genesis-contract](https://github.com/bnb-chain/bsc-genesis-contract) — `ValidatorContract`, `SlashContract`, `SystemRewardContract`, etc.
- **StakeHub (post-fusion)** — [github.com/bnb-chain/bsc-genesis-contract](https://github.com/bnb-chain/bsc-genesis-contract) {{unsourced: confirm path}} — post-2024 native staking.

## BEPs (specifications)

- **BEP repo** — [github.com/bnb-chain/BEPs](https://github.com/bnb-chain/BEPs) — canonical proposals.
- **BEP-126** — Fast finality (Plato). The single most important BEP for indexers.
- **BEP-95** — Real-time burning a portion of gas fees.
- **BEP-131** — Discontinue beacon chain {{unsourced: confirm number}}.
- **BEP-333** — Beacon chain sunset {{unsourced: confirm number}}.

## Documentation

- **BNB Chain Docs** — [docs.bnbchain.org](https://docs.bnbchain.org/) — official; covers Parlia, fast finality, system contracts, staking.
- **Validator docs** — within docs.bnbchain.org.

## Block explorers

- **BscScan** — [bscscan.com](https://bscscan.com/) — Etherscan-family L1 explorer.
- **OKLink** — [oklink.com](https://www.oklink.com/) — alternative.

## Operational

- **BNB Chain Status** — [status.bnbchain.org](https://status.bnbchain.org/) {{unsourced: confirm URL}}.
- **Validator dashboard** — [bscscan.com/validators](https://bscscan.com/validators) {{unsourced: confirm URL}}.
- **Public RPC** — `https://bsc-dataseed.bnbchain.org/` and round-robin alternatives. Rate-limited.

## Historical incidents

- **2022-10-06 Cube DAO incident** — search `bsc cube dao 2022 retrospective` for the canonical postmortem from BNB Chain. {{unsourced: pin specific URL}}
- Other halts and validator-set actions: documented in the BNB Chain Twitter / blog at the time. Not always indexed in a single retrospective.

## Type libraries

There is no BSC-specific Rust types crate beyond what `alloy` provides. BSC uses standard ETH JSON-RPC, so `alloy`'s default types work without modification — the only BSC-specific data structure that needs custom handling is the `extraData` validator-set encoding.

For Go indexers: use go-ethereum's RPC types and parse `extraData` against the Parlia source. For TS indexers: viem + a small custom parser.

## Where this doc is incomplete

The following claims carry `{{unsourced}}` markers and need primary-source backing:

- Plato (BEP-126) activation block / date.
- Lorentz / Maxwell activation dates and exact block-time changes.
- Tycho fork: whether it includes EIP-4788 / beacon-root precompile, and how BSC adapts (no L1-style beacon chain).
- BEP-131 active validator cap (claimed expanded from 21 to ~41 candidates).
- Beacon chain sunset completion block.
- Whether `bsc` exposes `parlia_*` namespace methods consistently.
- Cube DAO incident exact reorg / state changes during the recovery.
- Hertz fork activation (when EIP-1559 went live on BSC).

Verify against the BEPs repo, the bsc client source, and the BNB Chain docs before promoting this doc to `deep`.
