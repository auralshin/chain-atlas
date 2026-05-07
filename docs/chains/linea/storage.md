# Storage

**Same schema as Scroll.** PG and CH sketches in [scroll/storage.md](../scroll/storage.md) apply unchanged.

## Linea-specific notes

### L1 contract addresses

The L1 batch lifecycle table watches Linea's L1 contracts (`ZkEvm.sol`, `MessageService`, `TokenBridge`) instead of Scroll's. Schema is identical; the addresses to monitor differ.

### Excluded-tx tracking (early Linea)

For historical Linea data, you may want to capture which txs the sequencer excluded:

```sql
CREATE TABLE l2_excluded_txs (
    block_number      BIGINT,                  -- inferred from when the user submitted
    tx_hash           BYTEA,
    submitted_by      BYTEA,
    excluded_reason   TEXT,                    -- 'opcode_not_covered', 'sequencer_rejected', etc.
    PRIMARY KEY (tx_hash)
);
```

This is unique to Linea; Scroll and other ZK-EVMs don't have an analogous "submitted but excluded" pseudo-state.

### Multi-chain ZK-EVM indexers

If you're indexing both Scroll and Linea (and Ethereum), partition by `chain_id` everywhere — same advice as for OP Stack chains.

```sql
CREATE TABLE l2_blocks (
    chain_id        INTEGER NOT NULL,
    number          BIGINT NOT NULL,
    -- … rest as in scroll/storage.md
    PRIMARY KEY (chain_id, number)
);
```

### Sizing

Linea's volume is moderate — within typical Postgres sizing for several years of history. ClickHouse becomes worthwhile for log/event analytics workloads at scale.

## Choosing PG vs CH

Same as Scroll. Same recommendation: PG for source-of-truth + bridge state, CH for analytics.
