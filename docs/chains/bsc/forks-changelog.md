# Forks and Changelog

BSC's upgrade history is documented as a series of BEPs (BNB Evolution Proposals). Most BEPs are consensus-relevant; some are operational. Indexers care primarily about consensus BEPs and Ethereum-equivalence BEPs.

## Major upgrades

Activation block heights and timestamps below are taken verbatim from the canonical `BSCChainConfig` in [`bnb-chain/bsc/params/config.go`](https://github.com/bnb-chain/bsc/blob/master/params/config.go). The earlier (height-based) forks pinned to a block height; later (time-based) forks pinned to a Unix timestamp.

### Height-activated forks (pre-Shanghai)

| Upgrade | Block height (mainnet) | BEP | Indexer impact |
|---|---|---|---|
| **Genesis** | 0 | — | 2020-09 launch. |
| **MirrorSync** | 5,184,000 | — | Cross-chain bridge sync. |
| **Bruno** | 13,082,000 | [BEP-95](https://github.com/bnb-chain/BEPs/blob/master/BEPs/BEP95.md) | Real-time burning of a portion of gas fees. |
| **Euler** | 18,907,621 | — | EIP-1108, EIP-1884 — gas cost adjustments. |
| **Nano** | 21,962,149 | — | Bug fixes. |
| **Moran** | 22,107,423 | — | Post-Cube-DAO validator behavior tightening. |
| **Gibbs** | 23,846,001 | — | Shanghai-equivalent EVM changes (PUSH0, etc.). |
| **Planck** | 27,281,024 | — | Operational fixes. |
| **Luban** | 29,020,050 | — | BLS-related staking infrastructure. |
| **Plato** | **30,720,096** | **[BEP-126](https://github.com/bnb-chain/BEPs/blob/master/BEPs/BEP126.md)** | **Fast finality.** BLS-aggregated voting. `safe`/`finalized` block tags become meaningful. **Most important upgrade for indexers** since genesis. |
| **Berlin / London / Hertz** | 31,302,048 | — | Ethereum Berlin + London-equivalent at the same height: EIP-1559, EIP-2930, etc. (BSC adopted London-equivalent late.) |
| **HertzFix** | 34,140,700 | — | Hertz follow-up patch. |

### Time-activated forks (post-Shanghai)

| Upgrade | Activation (UTC) | Equivalent / notes |
|---|---|---|
| **Shanghai** / **Kepler** | 2024-01-23 08:00:00 | Ethereum Shanghai EVM equivalence. |
| **Feynman** (+ FeynmanFix) | 2024-04-18 05:49:00 | Validator-set / staking changes related to BC fusion. |
| **Cancun** / **Haber** | 2024-06-20 06:05:00 | Ethereum Cancun-equivalent: transient storage, MCOPY, beacon-root precompile. (Blob DA path used by opBNB; BSC L1 itself remains calldata.) |
| **HaberFix** | 2024-09-26 02:02:00 | Haber follow-up patch. |
| **Bohr** | 2024-09-26 02:20:00 | Consensus-layer adjustments. |
| **Pascal** / **Prague** | 2025-03-20 02:10:00 | Pectra-equivalent. |
| **Lorentz** | **2025-04-29 05:05:00** | **Block time reduction** toward 1.5 s. Affects "blocks per hour" assumptions. |
| **Maxwell** | 2025-06-30 02:30:00 | Further cadence / consensus tuning. |
| **Fermi** | 2026-01-14 02:30:00 | Continued evolution. |
| **Osaka** / **Mendel** | 2026-04-28 02:30:00 | Ethereum Osaka-equivalent. |

Always pull current fork heights/times from the `BSCChainConfig` in `params/config.go` at runtime — the values above are pinned to the canonical source as of the anchor date 2026-05-08.

## Beacon Chain → BSC fusion

Separate from BEP fork timeline:

| Phase | Date | Status |
|---|---|---|
| [BEP-333](https://github.com/bnb-chain/BEPs/blob/master/BEPs/BEP333.md) (BC sunset proposal) | Created 2023-11-29 | Status: Enabled (per BEP file header) |
| Feynman fork (validator-set / staking changes for BC fusion) | 2024-04-18 05:49:00 UTC ([BSCChainConfig](https://github.com/bnb-chain/bsc/blob/master/params/config.go)) | StakeHub becomes the native-staking destination |
| BC sunset complete | {{unsourced: needs governance forum / blog confirmation}} | BC deprecated |

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
