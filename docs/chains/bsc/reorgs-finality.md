# Reorgs and Finality

BSC has **two finality regimes** depending on whether you're indexing pre- or post-Plato history. Indexers that span the activation must handle both.

## Heads

| Head | Source | Lag | Reorg likelihood (post-Plato) |
|---|---|---|---|
| `head` | `eth_blockNumber` | ~0 s | Real but small |
| `safe` (justified) | `eth_getBlockByNumber("safe")` | ~2 blocks | Very low |
| `finalized` | `eth_getBlockByNumber("finalized")` | ~2 finalization rounds | Effectively never |

Pre-Plato, only `head` is meaningful and reorg risk is depth-based.

## Pre-Plato regime (~2020 → 2023)

No protocol finality. Indexer practice:
- **3–6 block confirmations** for fast UX, accepting that occasional reorgs at this depth would corrupt analytics.
- **15-block confirmations** for value-bearing accounting.
- **Long-tail outages produced reorgs of 10–20+ blocks** during validator misbehavior or downtime.

### The 2022 Cube DAO incident

In October 2022, the BNB Chain cross-chain bridge (BSC ↔ Beacon Chain Token Hub) was exploited for ~$570M of BNB. The validator set responded by **halting the chain** to prevent further fund movement. When the chain restarted, several blocks were effectively reorganized as part of the recovery.

For indexers:
- Document a state-of-chain snapshot before and after. Specific blocks were affected.
- The `bsc` codebase patched the bridge contracts via consensus-layer changes — indexers caching contract code must invalidate.
- This event is the strongest argument for **not** trusting BSC validator set against catastrophic-depth reorgs.

## Post-Plato regime (2023+)

BLS-aggregated voting → fast finality. Mechanism:

1. Active validators sign blocks with BLS keys.
2. Aggregated signatures attach to subsequent blocks.
3. Block N is **justified** when 2/3+ have signed.
4. Two consecutive justified blocks **finalize** the older one.

Reorgs past `finalized` require slashing 1/3+ of the validator set's stake — equivalent to the FFG security model.

| State | Time to reach (post-Plato) | Reorg risk |
|---|---|---|
| Justified (`safe`) | ~2 blocks (~6 s at 3 s blocks; faster post-Lorentz) | 1/3+ slashed |
| Finalized | ~4 blocks (~12 s) | 1/3+ slashed across two rounds |

Indexers post-Plato can use `eth_getBlockByNumber("finalized")` as a strong-finality signal. Non-finality halts (validator outages > 1/3) trigger an inactivity-leak-like behavior — finality stops advancing while head continues {{unsourced: confirm BSC's exact non-finalizing behavior}}.

## Reorg detection

Standard ETH-shape: subscribe to `newHeads`, compare parent hashes, walk back on mismatch.

Provisioning:
- **Pre-Plato historical** indexing: provision 20-block walk-back for safety.
- **Post-Plato live** indexing: provision 4-block walk-back; rely on `finalized` for stronger guarantees.
- **Around chain-halt incidents**: expect special operator instructions; may need to skip blocks or replay from a snapshot.

## Recommended commit policy

| Use case | Commit on (post-Plato) | Commit on (pre-Plato) |
|---|---|---|
| Real-time analytics | `head` | `head` |
| Token transfers, contract state | `safe` (justified) | head + 6 blocks |
| Bridge / financial accounting | `finalized` | head + 15 blocks (with caveats) |

For pre-Plato historical data, treat anything below 15 blocks as soft-confirmed only. For post-Plato data, the `finalized` block tag is the strong signal.

## Validator-collusion threat model

The BSC active validator set is small (21 active per epoch). 2/3 collusion is **plausible** in an adversarial scenario — both technically (small set) and historically (the 2022 incident showed coordinated validator action).

For an indexer:
- **Most workloads** can ignore this and trust `finalized`.
- **Audit-grade workloads** (regulator-facing, exchange compliance) should layer additional confirmation requirements (long depth + cross-chain proof of inclusion if available).
- **Bridge accounting** specifically: be prepared to handle "bridge has been paused / contract has been upgraded by validator consensus" as a real operational state.

## Inactivity / chain halts

BSC has halted the chain in operator-coordinated incidents (Cube DAO above, plus a brief halt during a different incident {{unsourced: confirm}}). Halts manifest as:

- `head` stops advancing.
- `safe` and `finalized` follow head (they cannot advance past head).
- WebSocket subscriptions go quiet.
- Public RPCs return the same block height for minutes to hours.

Indexers should not crash or silently drop data on halt — alert and wait. When the chain restarts, **read operator communications** (BNB Chain official channels) for any reorg / state changes during the halt; reconcile storage if necessary.
