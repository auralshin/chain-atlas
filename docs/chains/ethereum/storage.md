# Storage

Per-day volume baseline (early 2026, post-Fusaka):

| Quantity | Magnitude | Source |
|---|---|---|
| Blocks | ~7,200 | 12 s slots, ~100% slot fill |
| Transactions | ~1.2M | [verify with current dune/etherscan figures] |
| Logs | ~5M | [verify] |
| Withdrawals | ~32k | ~16 per block × 2k blocks; [verify] |

PG handles this comfortably for 1–2 years. CH starts to win at multi-year horizons or analytical workloads.

## Postgres schema

Minimum tables every Ethereum indexer needs:

```sql
CREATE TABLE blocks (
    number              BIGINT PRIMARY KEY,
    hash                BYTEA NOT NULL UNIQUE,
    parent_hash         BYTEA NOT NULL,
    timestamp           TIMESTAMPTZ NOT NULL,
    miner               BYTEA NOT NULL,
    gas_used            BIGINT NOT NULL,
    gas_limit           BIGINT NOT NULL,
    base_fee            NUMERIC(78),               -- nullable pre-London
    blob_gas_used       BIGINT,                    -- nullable pre-Cancun
    excess_blob_gas     BIGINT,                    -- nullable pre-Cancun
    parent_beacon_root  BYTEA,                     -- nullable pre-Cancun
    withdrawals_root    BYTEA,                     -- nullable pre-Shapella
    is_canonical        BOOLEAN NOT NULL DEFAULT TRUE
);
CREATE INDEX ix_blocks_hash      ON blocks (hash);
CREATE INDEX ix_blocks_canonical ON blocks (is_canonical) WHERE is_canonical;

CREATE TABLE transactions (
    hash                  BYTEA PRIMARY KEY,
    block_number          BIGINT NOT NULL REFERENCES blocks(number),
    tx_index              INT NOT NULL,
    tx_type               SMALLINT NOT NULL,        -- 0..4
    from_addr             BYTEA NOT NULL,
    to_addr               BYTEA,                    -- NULL for contract creation
    value                 NUMERIC(78) NOT NULL,
    gas_limit             BIGINT NOT NULL,
    gas_used              BIGINT NOT NULL,
    status                SMALLINT NOT NULL,
    effective_gas_price   NUMERIC(78),
    blob_versioned_hashes BYTEA[],                  -- NULL except type-3
    authorization_count   SMALLINT,                 -- NULL except type-4
    UNIQUE (block_number, tx_index)
);
CREATE INDEX ix_tx_from ON transactions (from_addr);
CREATE INDEX ix_tx_to   ON transactions (to_addr);

CREATE TABLE logs (
    block_number BIGINT NOT NULL,
    tx_hash      BYTEA NOT NULL,
    log_index    INT NOT NULL,
    address      BYTEA NOT NULL,
    topic0       BYTEA,
    topic1       BYTEA,
    topic2       BYTEA,
    topic3       BYTEA,
    data         BYTEA,
    PRIMARY KEY (block_number, log_index)
) PARTITION BY RANGE (block_number);

CREATE INDEX ix_logs_addr_topic0 ON logs (address, topic0);

CREATE TABLE withdrawals (
    index           BIGINT PRIMARY KEY,
    block_number    BIGINT NOT NULL REFERENCES blocks(number),
    validator_index BIGINT NOT NULL,
    address         BYTEA NOT NULL,
    amount_gwei     BIGINT NOT NULL                 -- gwei, not wei
);

CREATE TABLE authorizations (                       -- EIP-7702 (Pectra)
    tx_hash       BYTEA NOT NULL,
    auth_index    SMALLINT NOT NULL,
    chain_id      BIGINT NOT NULL,
    authority     BYTEA NOT NULL,                   -- recovered from signature
    delegated_to  BYTEA NOT NULL,
    nonce         BIGINT NOT NULL,
    PRIMARY KEY (tx_hash, auth_index)
);
```

Partitioning: monthly range partitions on `logs(block_number)`. Ethereum produces ~210k blocks/month; partitions of 1M blocks per file (~5 months of data) are a reasonable sweet spot.

When PG is the right choice:

- Single-chain or few-chain workloads
- Mixed read patterns including point lookups and address-keyed queries
- Up to ~2 years of historical data on commodity hardware
- Strong relational integrity and ACID matter

When PG breaks:

- Full-history full-text logs scans
- Wide analytical queries (`GROUP BY topic0 ... ORDER BY count DESC`) over hundreds of millions of rows
- Multi-chain stores at terabyte scale

## ClickHouse schema

```sql
CREATE TABLE blocks (
    number              UInt64,
    hash                FixedString(32),
    parent_hash         FixedString(32),
    timestamp           DateTime,
    miner               FixedString(20),
    gas_used            UInt64,
    gas_limit           UInt64,
    base_fee            Nullable(UInt256),
    blob_gas_used       Nullable(UInt64),
    excess_blob_gas     Nullable(UInt64),
    parent_beacon_root  Nullable(FixedString(32)),
    withdrawals_root    Nullable(FixedString(32))
) ENGINE = ReplacingMergeTree()                     -- collapse on rollback by hash
ORDER BY (number, hash)
PARTITION BY toYYYYMM(timestamp);

CREATE TABLE logs (
    block_number    UInt64,
    block_timestamp DateTime,
    tx_hash         FixedString(32),
    log_index       UInt32,
    address         FixedString(20),
    topic0          Nullable(FixedString(32)),
    topic1          Nullable(FixedString(32)),
    topic2          Nullable(FixedString(32)),
    topic3          Nullable(FixedString(32)),
    data            String
) ENGINE = MergeTree()
ORDER BY (address, topic0, block_number, log_index)
PARTITION BY toYYYYMM(block_timestamp);

ALTER TABLE logs ADD PROJECTION by_block (
    SELECT * ORDER BY block_number, log_index
);
```

The two `ORDER BY` strategies — `(address, topic0, ...)` for the main table and `(block_number, ...)` via a projection — let address-filtered queries and chronological scans both be fast.

When CH wins:

- Multi-chain stores
- Analytical queries (top emitters, daily-log-frequency aggregations)
- 5+ years of history on a single instance
- Read-heavy with no need for in-place updates

CH operational cost is real — replication, backups, and schema evolution are harder than PG. Most production indexers run **both**: PG for state and operational queries, CH for events and analytics.

## Reorg-aware writes

Two patterns:

**Soft-delete (PG):** mark `is_canonical = false` on rollback. Re-insert canonical chain. Queries filter `WHERE is_canonical`. Periodic cleanup deletes orphan rows older than the finality watermark.

**Replacing engine (CH):** use `ReplacingMergeTree(version)` keyed by `(number, hash)` or by tx hash. Writes always insert; the engine collapses duplicates by keeping the highest version on merge. Readers may briefly see duplicates until merges catch up — for analytical queries that's usually fine; for point lookups, use `FINAL` or query a materialized view.

For Ethereum specifically: reorgs rarely exceed 2 slots post-Merge, so soft-delete tables with periodic cleanup work well. Beyond 2 slots — the reorg has already finalized; you'd be re-reading the canonical chain anyway.
