# Storage

BSC's data is Ethereum-shaped — schemas borrow heavily from the [ethereum/storage.md](../ethereum/storage.md) baseline. The deltas are sizing (BSC's volume is high) and a few BSC-specific tables (validator state, slashing).

## Postgres sketch

### Block

Identical shape to ETH:

```sql
CREATE TABLE bsc_blocks (
    number          BIGINT PRIMARY KEY,
    hash            BYTEA NOT NULL UNIQUE,
    parent_hash     BYTEA NOT NULL,
    timestamp       BIGINT NOT NULL,
    miner           BYTEA NOT NULL,                -- the assigned validator for this block
    state_root      BYTEA NOT NULL,
    receipts_root   BYTEA NOT NULL,
    base_fee        NUMERIC(78,0),                 -- post-Hertz only
    difficulty      SMALLINT NOT NULL,             -- 1 = in-turn, 2 = out-of-turn
    -- Era / finality
    is_post_plato   BOOLEAN NOT NULL,              -- determines finality logic
    finality        SMALLINT NOT NULL              -- 0 unconfirmed, 1 justified, 2 finalized
);
CREATE INDEX ON bsc_blocks (miner);
CREATE INDEX ON bsc_blocks (finality, number) WHERE finality < 2;
```

`is_post_plato` is derived from the block number against the Plato fork height. Storing it makes era-conditional queries fast.

### Transactions and receipts

Same as Ethereum — no BSC-specific tx fields. Reuse:

```sql
-- See ../ethereum/storage.md
CREATE TABLE bsc_transactions (...);
CREATE TABLE bsc_receipts (...);
CREATE TABLE bsc_logs (...);
```

No `l1Fee` or other rollup-specific columns; BSC is L1.

### Validator state

```sql
CREATE TABLE bsc_validators (
    consensus_address  BYTEA PRIMARY KEY,
    operator_address   BYTEA NOT NULL,
    bls_pubkey         BYTEA,                      -- post-Plato only
    voting_power       NUMERIC(78,0) NOT NULL,
    activated_epoch    BIGINT NOT NULL,
    deactivated_epoch  BIGINT,
    is_slashed         BOOLEAN NOT NULL DEFAULT FALSE
);

CREATE TABLE bsc_epoch_validators (
    epoch_number       BIGINT NOT NULL,
    epoch_start_block  BIGINT NOT NULL,
    validators         BYTEA[] NOT NULL,           -- ordered list of consensus_addresses
    PRIMARY KEY (epoch_number)
);

CREATE TABLE bsc_slashing_events (
    block_number       BIGINT NOT NULL,
    tx_hash            BYTEA NOT NULL,
    log_index          INTEGER NOT NULL,
    validator          BYTEA NOT NULL,
    slash_type         TEXT NOT NULL,              -- 'double_sign', 'inactive', etc.
    PRIMARY KEY (block_number, log_index)
);
```

Per-epoch validator history lets you answer "who was active at block N" in one query.

### Post-Plato finality (BLS attestations)

Optional but useful for trust-minimized indexers:

```sql
CREATE TABLE bsc_attestations (
    block_number         BIGINT NOT NULL,
    target_block         BIGINT NOT NULL,         -- the block being justified
    aggregated_signature BYTEA NOT NULL,
    voter_bitmap         BYTEA NOT NULL,          -- which validators signed
    PRIMARY KEY (block_number)
);
```

Most indexers don't need this — `eth_getBlockByNumber("finalized")` is sufficient. Store it only if you're verifying finality independently.

### Cross-chain (legacy, optional)

If your indexer covers pre-fusion history:

```sql
CREATE TABLE bsc_cross_chain_packages (
    block_number       BIGINT NOT NULL,
    package_id         BIGINT NOT NULL,
    channel_id         INTEGER NOT NULL,           -- which cross-chain channel
    payload            BYTEA NOT NULL,
    direction          SMALLINT NOT NULL,          -- 0 from BC, 1 to BC
    PRIMARY KEY (block_number, package_id, channel_id)
);
```

Stop ingesting this after the BC fusion completion block.

### Native staking (post-fusion)

```sql
CREATE TABLE bsc_stake_events (
    block_number       BIGINT NOT NULL,
    tx_hash            BYTEA NOT NULL,
    log_index          INTEGER NOT NULL,
    delegator          BYTEA NOT NULL,
    validator          BYTEA NOT NULL,
    event_type         TEXT NOT NULL,              -- 'delegate', 'undelegate', 'redelegate', 'claim'
    amount             NUMERIC(78,0) NOT NULL,
    PRIMARY KEY (block_number, log_index)
);
CREATE INDEX ON bsc_stake_events (delegator);
CREATE INDEX ON bsc_stake_events (validator);
```

## ClickHouse sketch

For BSC's high event volume. Weekly partitions are reasonable; daily can also work for the largest contracts.

```sql
CREATE TABLE bsc_logs
(
    block_number       UInt64,
    tx_index           UInt32,
    log_index          UInt32,
    block_hash         FixedString(32),
    address            FixedString(20),
    topic0             FixedString(32),
    topic1             Nullable(FixedString(32)),
    topic2             Nullable(FixedString(32)),
    topic3             Nullable(FixedString(32)),
    data               String,
    timestamp          DateTime,
    finality           UInt8                       -- 0/1/2
)
ENGINE = ReplacingMergeTree(block_number)
PARTITION BY toMonday(timestamp)
ORDER BY (address, topic0, block_number, log_index)
SETTINGS index_granularity = 8192;
```

For internal txs (traces) — BSC has very high call-volume contracts, so this table can dwarf logs:

```sql
CREATE TABLE bsc_internal_txs
(
    block_number       UInt64,
    tx_index           UInt32,
    trace_address      Array(UInt32),
    type               LowCardinality(String),
    "from"             FixedString(20),
    "to"               Nullable(FixedString(20)),
    value              UInt256,
    gas                UInt64,
    gas_used           UInt64,
    input              String,
    output             String,
    error              Nullable(String),
    timestamp          DateTime
)
ENGINE = MergeTree
PARTITION BY toMonday(timestamp)
ORDER BY ("from", block_number, tx_index, trace_address)
SETTINGS index_granularity = 8192;
```

## Reorg handling

Era-aware:

- **Pre-Plato blocks**: maintain a buffer of 16 blocks before treating data as confirmed for analytics. Audit-grade requires longer.
- **Post-Plato blocks**: rely on `finalized` block tag from `eth_getBlockByNumber`. Reorgs past finality are catastrophic events that require operator response.

Cascade-delete on `bsc_blocks` is fine because reorg depths post-Plato are typically 1–3 blocks. For pre-Plato historical data, the depth distribution allows the same approach but with deeper retention.

## Choosing PG vs CH

| Workload | Pick |
|---|---|
| Validator-state queries, slashing analytics | **Postgres** |
| Cross-chain bridge accounting (legacy) | **Postgres** |
| Token transfer history per address | Either |
| `eth_getLogs`-shaped queries at scale (very common on BSC) | **ClickHouse** |
| Trace-based analytics (DEX volume, MEV, etc.) | **ClickHouse** |
| Native staking analytics | **Postgres** |

BSC's high volume + tight RPC limits make **self-hosted ClickHouse against your own archive** the practical default for any analytics indexer.

## Sizing

BSC archive nodes are large (multi-TB) and growing fast. ClickHouse storage for 1 year of BSC logs is in the hundreds-of-GB range; for 1 year of internal txs, multi-TB. Plan storage tiers (hot SSD for last 90 days, S3-tiered for older).
