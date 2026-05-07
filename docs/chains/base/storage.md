# Storage

**Same schema as Optimism.** PG and CH sketches in [optimism/storage.md](../optimism/storage.md) apply unchanged.

## Base-specific notes

### No legacy era to handle

Optimism's storage advice includes "don't try to stream OVM 1.0/2.0 with the same code as Bedrock+." That doesn't apply on Base — there's nothing before Bedrock. Tables can omit the `era` discriminator that some multi-OP-chain indexers carry.

### Higher row volume

Base typically produces more transactions and logs per block than OP Mainnet, sometimes by a substantial multiple. Sizing implications:

- **Postgres**: a `l2_logs` table can cross the ~1B-row threshold faster than on OP Mainnet. Plan partitioning (by `block_number / 1_000_000` or by `timestamp` month) from the start, not as a retrofit.
- **ClickHouse**: monthly partitions can become unwieldy quickly. Consider weekly partitions for `l2_logs` on Base.

### Multi-chain indexers (OP + Base + others)

If you're building a single indexer that covers multiple OP Stack chains, **partition by `chain_id` first** in every table. Some teams skip this and find themselves rewriting indexes when adding a new chain.

```sql
CREATE TABLE l2_blocks (
    chain_id        INTEGER NOT NULL,
    number          BIGINT NOT NULL,
    hash            BYTEA NOT NULL,
    -- … rest as in optimism/storage.md
    PRIMARY KEY (chain_id, number)
);
CREATE INDEX ON l2_blocks (chain_id, hash);
```

In ClickHouse, `chain_id` becomes the leading column of the `ORDER BY`:

```sql
ORDER BY (chain_id, address, topic0, block_number, log_index)
```

### Bridge accounting per chain

L1 bridge events (deposits, withdrawals) on Ethereum are written by **multiple** OP Stack chains. The `OptimismPortal` for OP Mainnet, Base, and other chains each emit `TransactionDeposited` from a different L1 address. An indexer's L1 ingestion pass must classify by emitter address, then route to the per-chain L2 indexer.

```sql
CREATE TABLE l1_to_l2_deposits (
    chain_id            INTEGER NOT NULL,        -- 10 = OP, 8453 = Base, …
    l1_block_number     BIGINT NOT NULL,
    l1_tx_hash          BYTEA NOT NULL,
    source_hash         BYTEA NOT NULL,          -- matches L2 deposit tx source_hash
    "from"              BYTEA NOT NULL,
    "to"                BYTEA,
    mint                NUMERIC(78,0) NOT NULL,
    value               NUMERIC(78,0) NOT NULL,
    PRIMARY KEY (chain_id, source_hash)
);
```

## Choosing PG vs CH

Same tradeoffs as Optimism. Same recommendation: PG for source-of-truth state, CH for analytics, derived from the same raw stream.
