# Reorgs and Finality

**Same model as Optimism.** Three heads (`unsafe`/`safe`/`finalized`), same reorg sources, same recommended commit policy.

See [optimism/reorgs-finality.md](../optimism/reorgs-finality.md) for the full model.

## What's different on Base

### Sequencer profile

Base's sequencer is operated by Coinbase Cloud. In practice this has meant:

- **Higher uptime** than typical OP Stack chains historically.
- **Sequencer reorgs of 1–3 unsafe blocks are still routine** — the same factors that cause unsafe rewinds on OP Mainnet apply.
- A few public outage incidents have caused longer pauses {{unsourced: cite specific incidents — Base outage 2023-09 is the well-known one}}. During an outage, `safe_l2` continues to advance but `unsafe_l2` does not.

### L1 reorg cascade depth

Identical math to Optimism: an L1 reorg of N blocks can re-derive up to ~6N L2 blocks of the safe layer (12s L1 / 2s L2). L1 reorg distributions are an L1 property, not a Base property — so the depth distribution matches what an Ethereum indexer already plans for.

### Fault proofs on Base

Base's fault-proof rollout is on its own timeline. Until permissionless fault proofs are live, the `finalized_l2` head is **trust-dependent** — Coinbase's `op-proposer` posts output roots, and the L1 finality of those output roots is the indexer's only finality signal.

Verify current fault-proof status at deep-dive time. The treatment in code is the same: respect `finalized_l2` from `optimism_syncStatus` and don't override it.

## Recommended commit policy

Identical to OP Mainnet:

| Use case | Commit on |
|---|---|
| Real-time analytics dashboards | `unsafe_l2`, marked |
| Token transfers, contract state | `safe_l2` |
| Bridge accounting, audit ledgers | `finalized_l2` |
