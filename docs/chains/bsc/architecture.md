# Architecture

PoSA (Proof of Staked Authority) — a hybrid of PoA (Proof of Authority, like Clique) and stake-weighted validator election. The validator set is small enough to enable fast block production but large enough to be more decentralized than pure PoA.

## Consensus

| Property | Value |
|---|---|
| Consensus algorithm | Parlia (PoSA) |
| Active validator set | 21 active per epoch + 20 candidate (backup) validators = 41 total ([BEP-131](https://github.com/bnb-chain/BEPs/blob/master/BEPs/BEP131.md)) |
| Epoch length | 200 blocks (Parlia default) |
| Block time | 3 s (genesis–Lorentz); reduced toward 1.5 s post-Lorentz (Lorentz activation: 2025-04-29 05:05:00 UTC, [BSCChainConfig](https://github.com/bnb-chain/bsc/blob/master/params/config.go)) |
| Block proposer | Round-robin among active validators within an epoch |
| Out-of-turn proposer | Allowed with reduced "difficulty" — provides liveness during proposer absence |

Validators stake BNB to qualify. The active set rotates each epoch via stake-weighted election (post-Beacon-chain-fusion: staking happens directly on BSC).

## Finality model — pre and post BEP-126 (Plato fork)

### Pre-Plato (~2020–2023)

No protocol finality. Reorg risk reduces with depth, but no formal guarantee. Empirically:
- **Reorgs of 2–6 blocks** were observed regularly during normal operation.
- **Reorgs of 10–15+ blocks** happened during validator outages or network issues.
- The conventional indexer wisdom was "wait 15 blocks for safety."

### Post-Plato ([BEP-126](https://github.com/bnb-chain/BEPs/blob/master/BEPs/BEP126.md), block 30,720,096 — see [BSCChainConfig](https://github.com/bnb-chain/bsc/blob/master/params/config.go))

**Fast finality.** Validators sign blocks with BLS keys and aggregate. Block N is **justified** when 2/3+ of the active validator set have signed it. Two consecutive justified blocks **finalize** the older one.

| State | Reorg requirement | Typical age |
|---|---|---|
| `head` | 1 honest validator misses | 0–3 s |
| `safe` (justified) | 1/3+ validators slashed | ~2 blocks |
| `finalized` | 1/3+ validators slashed (across two voting rounds) | ~2 finalization rounds |

This is **structurally similar to Ethereum's Casper FFG**, with a much smaller validator set and faster cadence.

`bsc` exposes `eth_getBlockByNumber("safe")` and `eth_getBlockByNumber("finalized")` from the Plato fork onward (per [BEP-126 §4 Specification](https://github.com/bnb-chain/BEPs/blob/master/BEPs/BEP126.md)).

## Validator rotation

At each epoch boundary (every 200 blocks):

1. The current proposer fetches the candidate validator set (from on-chain `Validators` system contract).
2. Stake-weighted election produces the next epoch's active set.
3. The transition is encoded in the **header extra-data** of the last block of the previous epoch.

Indexers that need validator-state per epoch parse extra-data at epoch boundaries:

```
header.extraData = | extra_vanity (32B) | validators (N × 20B + N × 48B BLS)... | extra_seal (65B) |
```

(Format varies pre- vs post-Plato; verify against Parlia source.)

## System contracts

BSC has a set of "system contracts" deployed at low addresses by genesis. They are upgradeable via cross-validator consensus.

| Address | Contract | Role |
|---|---|---|
| `0x0000…1000` | `ValidatorContract` | Active validator set, slashing |
| `0x0000…1001` | `SlashContract` | Records misbehavior, applies slashing |
| `0x0000…1002` | `SystemRewardContract` | Distributes block rewards |
| `0x0000…1003` | `LightClientContract` | Light client of BNB Beacon Chain (deprecated post-fusion) |
| `0x0000…1004` | `TokenHubContract` | Cross-chain token transfers (deprecated post-fusion) |
| `0x0000…1005` | `RelayerHubContract` | Cross-chain relayers (deprecated post-fusion) |
| `0x0000…1006` | `GovHubContract` | Governance proposals (deprecated post-fusion) |
| `0x0000…1007` | `TokenManagerContract` | Token registration (deprecated post-fusion) |
| `0x0000…2000` | `StakeHubContract` (post-fusion) | Native BSC staking, replaces beacon-chain staking |

A complete BSC indexer maintains decoders for these — most contract activity is mundane, but validator-set changes, slashing events, and post-fusion staking flow through them.

## Beacon Chain → BSC fusion (2024)

Pre-2024: BNB Chain had two L1s — **BNB Beacon Chain (BC)** for staking + governance, and **BSC** for EVM execution. Cross-chain communication via Token Hub / Relayer Hub.

In 2024, BC was deprecated. Staking moved natively onto BSC via `StakeHub`. Cross-chain contracts (TokenHub, etc.) were sunset.

For indexers anchored on **2026-05-08**: BC is historical. New activity is BSC-only. Historical data on BC exists but is rarely indexed by application teams.

## Where the data lives

| Data | Where to fetch | Notes |
|---|---|---|
| Blocks, txs, receipts, logs | `bsc` JSON-RPC | Standard ETH RPC |
| Validator set | `bsc` JSON-RPC, `eth_call` against `0x0000…1000` | Or parse header extra-data at epoch boundaries |
| Finality status | `eth_getBlockByNumber("finalized")` post-Plato | Authoritative |
| BLS attestations (post-Plato) | `bsc` JSON-RPC `eth_getBlockByHash` votes field {{unsourced: confirm field}} | For verifying finality |
| Slashing events | `0x0000…1001` events | |
| Stake / staking events (post-fusion) | `0x0000…2000` events | |
| Beacon chain (historical) | Separate BC node — practically nobody indexes this anymore | Deprecated |
