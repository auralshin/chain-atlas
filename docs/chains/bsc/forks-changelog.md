# Forks and Changelog

BSC's upgrade history is documented as a series of BEPs (BNB Evolution Proposals). Most BEPs are consensus-relevant; some are operational. Indexers care primarily about consensus BEPs and Ethereum-equivalence BEPs.

## Major upgrades

| Upgrade | Date | BEP | Indexer impact |
|---|---|---|---|
| **Genesis** | 2020-09-01 | — | Block 0. |
| **Bruno** | {{unsourced: 2021}} | BEP-95 | Burns a portion of gas fees. Affects fee accounting. |
| **Euler** | {{unsourced: 2021/2022}} | — | EIP-1108, EIP-1884 — gas cost adjustments. |
| **Gibbs** | {{unsourced}} | — | EIP-3855 (PUSH0) and other Shanghai-equivalent. |
| **Moran** | {{unsourced: post-Cube-DAO 2022}} | — | Stricter validator behavior, anti-collusion guards. Indexer impact: validator-set semantics tightened. |
| **Nano** | {{unsourced}} | — | Bug fixes. |
| **Plato** | {{unsourced: 2023}} | **BEP-126** | **Fast finality.** BLS-aggregated voting. `safe`/`finalized` block tags become meaningful. **Most important upgrade for indexers** since genesis. |
| **Hertz** | {{unsourced: 2023}} | — | Ethereum Berlin/London-equivalent: EIP-1559, EIP-2930. (BSC adopted London-equivalent late.) |
| **Tycho** | {{unsourced: 2024}} | — | Cancun-equivalent: blob support (used by opBNB), transient storage, MCOPY, beacon-root. {{unsourced: confirm — BSC may have adopted differently}} |
| **Pascal** | {{unsourced: 2024}} | — | Continued Ethereum-equivalence work. |
| **Lorentz** | {{unsourced: 2024-2025}} | — | **Block time reduction** toward 1.5s. Affects "blocks per hour" assumptions. |
| **Maxwell** | {{unsourced: 2025}} | — | Further cadence / consensus tuning. |

Always pull current fork heights from the `bsc` chain config (`genesis.json` or `BscChainConfig`) at runtime.

## Beacon Chain → BSC fusion

Separate from BEP fork timeline:

| Phase | Approximate date | Status |
|---|---|---|
| BEP-333 (BC sunset proposal) | 2023-2024 {{unsourced}} | Approved |
| StakeHub deployed on BSC | 2024 {{unsourced}} | Native staking begins |
| BC sunset complete | 2024 {{unsourced: confirm exact date}} | BC deprecated |

Indexer impact:
- **Pre-fusion**: cross-chain bridge contracts (`TokenHub`, `RelayerHub`, etc.) are active. Substantial event volume.
- **Post-fusion**: those contracts become inactive. Native staking moves to `StakeHub`. Indexers should detect the cutover and stop processing cross-chain events after that point.

## Pre-Plato vs Post-Plato — the indexer's "biggest fork"

For practical indexing, **Plato is the dividing line**. Two different finality models, two different recommended commit policies, two different RPC behaviors for `safe`/`finalized`.

A complete BSC indexer covering pre- and post-Plato history needs:
- A configurable **confirmation depth per era**: 15 blocks pre-Plato, `finalized` block tag post-Plato.
- A **finality-detection mode switch** at the Plato activation block.
- A **header parser** that handles both pre- and post-Plato extraData formats.

## What an indexer must do per upgrade

1. **Pin BEP activation heights** at startup from the chain config.
2. **Switch finality detection at Plato.** Pre-Plato uses depth heuristics; post-Plato uses `finalized` block tag.
3. **Reprocess affected blocks** if a fork changes EVM semantics that affect your derived data (gas, specific opcodes).
4. **Stop processing cross-chain events** at the BC fusion completion block.
5. **Recompute "blocks per hour" estimates** if your indexer caches them — Lorentz changes block time.

## Operational incidents that aren't forks

| Date | Event | Indexer implication |
|---|---|---|
| 2022-10-06 | Cube DAO / Token Hub exploit, chain halted briefly | Validator-set response → reorg / state freeze; reconcile per operator advisory |
| {{unsourced}} | Other halts | Same pattern |

Subscribe to BNB Chain official communications channels for incident notifications. Validator-set actions can affect data after the fact.
