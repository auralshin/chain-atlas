# Forks and Changelog

Polygon's upgrade history runs across **two** software stacks (`bor` and `heimdall`), which fork independently. Indexers care primarily about `bor` forks — they change EVM behavior — but heimdall forks change consensus and checkpoint logic.

## Eras

| Era | Span | Indexer impact |
|---|---|---|
| **Plasma** (Matic Network) | 2018–2020 | Different stack entirely. **Out of scope for this guide.** |
| **PoS Chain** | 2020-05-30 → present | This guide. EVM-equivalent (post-major-forks). |

Plasma-era data is generally not indexed alongside PoS data — they're separate chains for practical purposes.

## Bor (EVM) forks

| Upgrade | Bor fork height | Approx date | Indexer impact |
|---|---|---|---|
| **Genesis** | 0 | 2020-05-30 | — |
| **Berlin** | {{unsourced}} | {{unsourced}} | EIP-2929 gas costs, EIP-2930 access lists |
| **London** | 23850000 {{unsourced}} | 2022-01-17 {{unsourced}} | **EIP-1559** — base fee, type-2 txs |
| **Delhi** | {{unsourced}} | {{unsourced}} | Mumbai testnet feature parity |
| **Indore** | {{unsourced: 2023}} | {{unsourced}} | Production fixes |
| **Bhilai** | {{unsourced: 2023}} | {{unsourced}} | Shanghai-equivalent: PUSH0 etc. Adds `safe`/`finalized` block tags. |
| **Napoli** | {{unsourced: 2024}} | {{unsourced}} | Cancun-equivalent: transient storage, MCOPY, beacon root precompile (with caveats — Polygon adapts EIP-4788 differently since there is no L1-style beacon chain). |
| **Pragati** {{unsourced: name?}} | {{unsourced: 2024-2025}} | {{unsourced}} | Pectra-equivalent EIPs ported. |

Always pull current fork heights from the bor `genesis.json` / chain config or from the [bor releases](https://github.com/maticnetwork/bor/releases).

## Heimdall forks

Heimdall's consensus and checkpoint logic also evolves. Notable:

| Upgrade | Approx date | Impact |
|---|---|---|
| **Heimdall v0.2** | {{unsourced}} | Introduced span election improvements |
| **Heimdall v0.3** | {{unsourced}} | Cosmos SDK upgrade, improved validator slashing |
| **Heimdall v2** ("Heimdall-v2") | {{unsourced: 2024-2025}} | Substantial rewrite, Cosmos SDK v0.50+ — affects `clerk/*`, `staking/*` REST shapes. |

**Heimdall-v2 is a breaking change for indexers querying heimdall's REST API** {{unsourced: confirm scope}}. Field names, endpoint paths, and pagination semantics changed. Indexers should detect the heimdall version via `/status` at startup and route accordingly.

## Polygon 2.0 / AggLayer evolution

Polygon's roadmap toward "Polygon 2.0" involves:
- POL token migration (done, 2024)
- AggLayer (cross-chain settlement) — relevant to CDK chains, not PoS itself
- ZK validation of PoS state (proposed; status TBD)

These are **roadmap items**, not yet activated on PoS chain itself. An indexer doc anchored on 2026-05-08 should treat ZK validation of PoS as upcoming, not present.

## Token migration: MATIC → POL

| Date | Event |
|---|---|
| **2024-09-04** {{unsourced: confirm}} | Native gas token on bor relabeled MATIC → POL via the L1 migration contract |
| 2024-09 — present | Display tooling shows POL; old contracts still functional |

The L2 token contract address at `0x...1010` did not change; only the symbol/name and the L1-side token contract did. Indexers that hardcoded `"MATIC"` as the gas token symbol must update display logic.

## What an indexer must do per fork

1. **Pin bor fork heights** from the chain config at startup. EIP-1559 vs legacy gas pricing changes at the London fork height.
2. **Detect heimdall version** from `/status` at startup; route REST queries accordingly.
3. **Reprocess affected blocks** if your indexer recomputes anything affected by the fork (gas costs, opcode availability).
4. **Update token symbol** for MATIC → POL at the migration block (display only).

## Historical incidents (not forks, but indexer-relevant)

| Date | Incident | Indexer implication |
|---|---|---|
| 2022-03 | "Disturbing Reorg" — observed reorg of {{unsourced: confirmed depth}} blocks | Validates the case for 256+ confirmation depth |
| 2022-12 | Heimdall halt for {{unsourced: duration}} | Checkpoints stopped advancing; bor continued |
| 2023-{{unsourced}} | Bhilai upgrade rolled out | `safe`/`finalized` block tags became available |
| 2024-09 | POL migration | Token symbol change |

Verify against Polygon's incident retrospectives and governance forum.
