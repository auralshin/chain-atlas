# Reorgs and Finality

Same model as [Scroll](../scroll/reorgs-finality.md). Two-stage L1 finality: commit → finalize.

See scroll/reorgs-finality.md for the full model.

## Linea-specific notes

### Sequencer profile

Single-sequencer, operated by Consensys. In practice:
- Sequencer rewinds are rare and shallow (1–3 blocks when they happen).
- Brief sequencer outages have happened during major upgrades or operational maintenance.

### Finalization cadence

Linea's prove + finalize cadence has historically been **slower than Scroll's** {{unsourced: confirm — was reportedly hours-to-days for some upgrade periods}}. Bridges and audit-grade indexers should be aware that "finalized" can lag substantially.

For non-finality-critical workloads (analytics, dashboards), commit on the L2 head with a small reorg buffer (10 blocks). For audit-grade, wait for `BlockFinalized` events on L1.

### Force-include

Linea has a force-include / inclusion-guarantee mechanism via L1 messaging {{unsourced: confirm details}}. If the sequencer is offline, users can submit txs through L1 that the protocol guarantees inclusion.

### No challenge window

Like every ZK rollup: SNARKs are mathematically guaranteed to verify. No fault-proof challenge. Reorgs past finality are not possible barring cryptographic break of the proof system or governance-mediated emergency intervention.

## Recommended commit policy

| Use case | Commit on |
|---|---|
| Real-time analytics | L2 head |
| Token transfers, contract state | L2 head + small buffer |
| Bridge accounting, audit-grade | L1-finalized (`BlockFinalized` on `ZkEvm.sol`) |
