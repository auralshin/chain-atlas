# RPC Surface

**Identical to Optimism.** All `eth_*`, `debug_*`, and `optimism_*` methods exposed by `op-geth` + `op-node` apply.

See [optimism/rpc-surface.md](../optimism/rpc-surface.md).

## Provider quirks specific to Base

Base has higher real-world traffic than most OP Stack chains, which has operational consequences for indexers:

- **`eth_getLogs` block-range limits are tighter on most public providers** for Base than for other OP chains, because the per-block log volume is higher. Plan for smaller windows (often 500–2000 blocks per call on free tiers).
- **Coinbase-operated public RPC** at [`mainnet.base.org`](https://mainnet.base.org/) — has rate limits; not suitable for heavy indexing.
- **Self-hosted is recommended** for production indexers due to throughput. Run `op-geth` + `op-node` against a synced Ethereum L1 archive.

## Trace API availability

Same as Optimism. `debug_traceBlockByNumber` works on archive `op-geth` / `op-reth` / `op-erigon` running against Base.

## Archive node disk size

Base's archive footprint grows faster than OP Mainnet's because of higher tx volume. As of writing: {{unsourced: actual GB figure}}. Plan accordingly — most providers' "archive" tier is more expensive for Base than for OP Mainnet.
