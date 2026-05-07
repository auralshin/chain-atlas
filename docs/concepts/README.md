# Concepts

Cross-chain machinery. Each chain's docs reference these instead of restating them.

## Planned

| File | What it covers |
|---|---|
| `finality-models.md` | Probabilistic vs single-slot vs sequencer-confirmed vs L1-finalized vs hybrid. The most common indexer bug across chains is treating sequencer confirmation as equivalent to finality. |
| `reorgs.md` | Detection, depth, recovery. Why "N confirmations" means very different things on different chains. |
| `rpc-vs-archive.md` | What you can and cannot get from a full node vs an archive node. Disk requirements. Pruning options. |
| `trace-apis.md` | `debug_trace*`, `ots_*`, parity-style `trace_*`, custom equivalents on non-EVM. Method-by-client matrix. |
| `storage-postgres.md` | When PG works for indexer workloads. Schema patterns, partitioning, materialized views. Where it breaks. |
| `storage-clickhouse.md` | When CH wins. ORDER BY semantics for blockchain data, projections, sparse data tricks. Operational cost. |
| `proxy-resolution.md` | EIP-1967, beacon, diamond (EIP-2535), minimal proxy (EIP-1167). How an indexer keeps ABI mappings sane. |
| `multi-chain-arch.md` | Adapter pattern for an indexer that handles many chains. What's shared, what isn't. |

These will be written before the chain deep-dives, since chain docs cite them.
