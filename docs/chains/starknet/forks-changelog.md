# Forks and Changelog

StarkNet versions its **protocol** (`starknet_version` field on each block) and its **RPC spec** (`starknet_specVersion`).

## Protocol versions

Each protocol version corresponds to a sequencer + node software upgrade. Indexer-relevant ones:

| Protocol version | Approx date | Indexer impact |
|---|---|---|
| **V0.10** | {{unsourced: 2022}} | Removed deprecated DEPLOY tx type; account abstraction matured. |
| **V0.11** | {{unsourced: 2023-Q1}} | **Cairo 1 / Sierra introduced.** DECLARE v2 for Sierra-class declarations. **Indexers must handle both Cairo 0 and Cairo 1 classes.** |
| **V0.12** | {{unsourced: 2023-Q3}} | Sequencer / RPC improvements; performance work. |
| **V0.13.0** | {{unsourced: 2024-Q1}} | **STRK fee token introduced.** Tx v3 — supports paying in either STRK or ETH. **Indexers must update fee-tracking logic.** |
| **V0.13.1** | {{unsourced: 2024}} | Block time reductions; protocol tweaks. |
| **V0.13.2** | {{unsourced: 2024}} | Continued performance work. |
| **V0.13.3** | {{unsourced: 2024-2025}} | Further tuning. |
| **V0.14** | {{unsourced: 2025}} | Decentralization work; continued upgrades. |

Always pull current protocol version from `starknet_version` on the latest block.

## Cairo 0 → Cairo 1/2 transition

Pre-V0.11: only Cairo 0 contracts. DECLARE v0 / v1.
Post-V0.11: Cairo 1 (Sierra-based) contracts can be declared via DECLARE v2. Cairo 0 contracts can still be **deployed** if their class was previously declared, but **new Cairo 0 declarations are disabled** post some later version {{unsourced: confirm exact deprecation}}.

For indexers:
- `starknet_getClass(class_hash)` returns either Cairo 0 or Cairo 1 format depending on the class. The shape is different.
- **ABI conventions differ**: Cairo 0 uses one ABI format; Cairo 1 uses another (more like Ethereum ABI but for felts).
- Function selector hashing differs across some edge cases.

A complete indexer's class decoder must support **both** formats.

## STRK fee token (V0.13)

Pre-V0.13: only ETH could pay fees. Receipts had `actual_fee` in WEI.
Post-V0.13: STRK fee payments are supported. Receipts have `actual_fee = { amount, unit }` where `unit ∈ {"WEI", "FRI"}` (FRI is STRK's smallest unit, like wei for ETH).

Indexer fee-tracking logic must handle both. A naive "sum of `actual_fee` in WEI" misses STRK-paid txs entirely.

## RPC spec versions

The `starknet-specs` repo versions independently from the protocol. Indexer-relevant:

- **v0.5** — pre-V0.13 spec.
- **v0.6** — V0.13 era.
- **v0.7** — current as of writing {{unsourced: confirm latest}}; adds subscription methods.
- **v0.8+** — continued evolution.

Different node clients support different spec versions; some support multiple. Pin compatibility via `starknet_specVersion` at startup.

## L1 → L2 message format changes

Pre-V0.10: certain message fields were structured differently. Post-V0.10: stable format with the `nonce` field consistently included.

Most indexers don't need to handle pre-V0.10 messages (rare in active use). For backfill of full chain history, decode against the fork-appropriate format.

## What an indexer must do per upgrade

1. **Pin `starknet_version`** at startup; route decoders by block's `starknet_version`.
2. **Pin `starknet_specVersion`** at startup; ensure your client library version is compatible.
3. **Update fee-tracking** at V0.13 (STRK alongside ETH).
4. **Update class decoder** at V0.11 (Cairo 1).
5. **Refresh tx-type version** at each V0.13.x sub-version (resource bounds, paymaster fields).

## Where this doc is incomplete

The exact dates for V0.10 / V0.11 / V0.12 / V0.13.x / V0.14, RPC spec progression, exact deprecation of Cairo 0 declarations, and the subscription-method support matrix are flagged `{{unsourced}}`. Verify against:

- [`starkware-libs/starknet-specs`](https://github.com/starkware-libs/starknet-specs) release tags.
- StarkWare blog posts for each V0.x announcement.
- [`pathfinder`](https://github.com/eqlabs/pathfinder) / [`juno`](https://github.com/NethermindEth/juno) release notes.
