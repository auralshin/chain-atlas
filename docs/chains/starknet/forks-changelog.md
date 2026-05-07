# Forks and Changelog

StarkNet versions two things independently: the **protocol** (`starknet_version` field on each block) and the **RPC spec** (`starknet_specVersion`). The RPC spec evolves on its own cadence in [`starkware-libs/starknet-specs`](https://github.com/starkware-libs/starknet-specs).

## Protocol versions

Each protocol version corresponds to a sequencer + full-node software upgrade. Indexer-relevant ones:

| Protocol version | Mainnet activation | Indexer impact |
|---|---|---|
| **v0.10.x** | 2022 {{unsourced: pin exact mainnet date}} | DEPLOY tx type deprecated (use DEPLOY_ACCOUNT); account abstraction matured. |
| **v0.11.0** | 2023-Q2 {{unsourced: pin exact mainnet activation}} | **Cairo 1 transition begins.** DECLARE v2 for Sierra classes is enabled on mainnet **after** governance vote and a Testnet stabilization period. **Indexers must handle both Cairo 0 and Cairo 1 classes**. ([source: Starknet blog "v0.11.0: Transition to Cairo 1.0"](https://www.starknet.io/blog/starknet-alpha-v0-11-0-the-transition-to-cairo-1-0-begins/)) |
| **v0.12.x** | 2023-Q3 {{unsourced: pin}} | Sequencer + RPC performance work. |
| **v0.13.0** | 2024 (Sepolia testnet 2023-12-13; mainnet shortly after) | **Tx v3 (the dominant version going forward).** **STRK fee token support** alongside ETH. Receipts gain `actual_fee = { amount, unit }` with `unit ∈ {"WEI", "FRI"}`. **Indexers must update fee aggregation to track both units.** ([source: CoinJournal](https://coinjournal.net/news/starknet-delegates-vote-to-move-v0-13-0-from-testnet-to-mainnet/)) |
| **v0.13.1** | 2024 | Block-time reductions; protocol-internal tweaks. |
| **v0.13.2** | 2024 {{unsourced}} | Continued performance work. |
| **v0.13.3** | 2024-late {{unsourced}} | Tuning. |
| **v0.13.4 / v0.13.5** | 2025 {{unsourced}} | Continued. |
| **v0.13.6** | **Mainnet 2025-07-08** (Testnet 2025-06-30) | Pre-Grinta tuning. |
| **v0.14.0** ("Grinta") | **Mainnet 2025-09-01** (originally 2025-08-18, delayed) | Decentralization work; protocol updates. ([source: CoinJournal](https://crypto.news/starknet-v0-14-0-upgrade-live-on-mainnet-2025/)) |
| **v0.14.1** | **Mainnet 2025-11-25** | Follow-up to Grinta. |
| **v0.15+** | 2026+ | {{unsourced: track via `starknet_version` on latest block}} |

Always pull current protocol version from `starknet_version` on the latest block at runtime.

## RPC spec versions

The `starknet-specs` repo versions independently from the protocol. Releases pulled from [`starkware-libs/starknet-specs/releases`](https://github.com/starkware-libs/starknet-specs/releases):

| Spec | Released | Roughly aligns with protocol |
|---|---|---|
| **v0.5** | pre-2023-12 | Pre-v0.13 era |
| **v0.6.0** | 2023-12-07 | v0.13.0 era |
| **v0.7.0** | 2024-03-14 | v0.13.x era |
| **v0.7.1** | 2024-03-24 | v0.13.x era |
| **v0.8.0** | 2025-03-17 | v0.13.x era; **adds WebSocket subscription methods** (`starknet_subscribeNewHeads`, `starknet_subscribeEvents`, etc.) |
| **v0.8.1** | 2025-04-02 | v0.13.x era |
| **v0.9.0** | 2025-08-11 | Around v0.13.6 / v0.14.0 |
| **v0.10.0** | **2025-11-25** | **Same day as v0.14.1 mainnet** — aligned release |
| **v0.10.1** | 2026-03-13 | Post-Grinta |
| **v0.10.2** | 2026-03-23 | Latest as of 2026-05-08 |

Different node clients support different spec versions. Some support multiple via routing. **Pin compatibility via `starknet_specVersion` at startup** before depending on subscription methods or other version-gated features.

## Cairo 0 → Cairo 1 transition

| Era | Behavior |
|---|---|
| **Pre-v0.11.0** | Only Cairo 0 contracts. DECLARE v0/v1 only. |
| **v0.11.0+** | **Cairo 1 transition begins** on mainnet (after governance vote). DECLARE v2 (Sierra classes) enabled. Cairo 0 contracts continue to work. |
| **Post-Regenesis** ({{unsourced: never executed as a single event}}) | Originally planned to fully deprecate Cairo 0. In practice, Cairo 0 contracts have continued to work — full deprecation has been gradual rather than a single Regenesis event. |

For indexers — both formats coexist on mainnet today (2026-05-08):
- `starknet_getClass(class_hash)` returns either Cairo 0 or Cairo 1 shape depending on the class.
- ABI formats differ (Cairo 0 list-of-functions vs Cairo 1 enum-style with discriminators).
- Function selector hashing has edge-case differences.

**A complete indexer's class decoder must support both Cairo 0 and Cairo 1 — even on a 2026-anchored deployment.**

## STRK fee token (v0.13.0+)

Pre-v0.13.0: only ETH could pay fees. Receipts had `actual_fee` denominated in WEI implicitly.
Post-v0.13.0: tx v3 introduces dual-token fee support.

| Tx version | Fee token |
|---|---|
| v0, v1, v2 | ETH (WEI) |
| **v3** | **STRK (FRI)** by default; ETH via paymaster fallback (e.g. AVNU, Cartridge paymasters) |

Receipts at and after v0.13.0:

```
"actual_fee": { "amount": "0x...", "unit": "WEI" | "FRI" }
```

A naive "sum of `actual_fee.amount`" gives nonsense — different units, different price feeds. **Always partition by `unit`** and convert separately for any aggregated revenue dashboard.

Mainnet trajectory: post-v0.14.0 (2025-09), the network increasingly uses STRK as the primary fee token; ETH-paid txs are minority traffic.

## L1 → L2 message format

| Era | Format |
|---|---|
| Pre-v0.10 | Earlier message format with different fields |
| **v0.10+** | Stable format with `nonce` field consistently included |

Most indexers anchored on 2026-05-08 don't see pre-v0.10 messages in active use. For backfill of full chain history (genesis = 2021-11), decode against the fork-appropriate format.

## What an indexer must do per upgrade

1. **Pin `starknet_version`** at startup; route decoders by block's `starknet_version` (each protocol version may add fields to receipts, blocks, or txs).
2. **Pin `starknet_specVersion`** at startup; verify your client library matches. WebSocket subscriptions require ≥ v0.8.0.
3. **Update fee-tracking at v0.13.0** to handle both WEI and FRI.
4. **Add Cairo 1 class decoder** for any indexing of contracts deployed at or after v0.11.0.
5. **Refresh tx v3 schema** at each v0.13.x sub-version (resource bounds shape evolves: l1_gas + l2_gas in early v0.13, plus l1_data_gas in later).
6. **Update RPC client library** at major spec bumps (v0.6 → v0.7 → v0.8 → v0.9 → v0.10) — field renames and new methods land at these.

## Where this doc is incomplete

These are flagged for verification against the canonical sources:

- Exact mainnet activation dates for v0.10.x, v0.11.0, v0.12.x, v0.13.1/.2/.3/.4/.5.
- Status of Regenesis (originally planned, gradual in practice).
- Per-protocol-version receipt / block schema additions (e.g. when `l1_data_gas_price` appeared on blocks).
- Whether v0.14.0 ("Grinta") includes any indexer-affecting RPC changes beyond version alignment.

Canonical sources for closing these:
- [`starkware-libs/starknet-specs/releases`](https://github.com/starkware-libs/starknet-specs/releases) — RPC spec release dates.
- [Starknet version notes](https://docs.starknet.io/learn/cheatsheets/version-notes) — protocol version dates.
- [Starknet developer version releases page](https://www.starknet.io/developers/version-releases/).
- StarkWare blog announcements per version.
