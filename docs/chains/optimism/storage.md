# Storage

OP Stack indexer schemas — Postgres for relational queries and bridge accounting; ClickHouse for high-volume event/trace analytics. Most production indexers run **both** with overlapping data; choose by query workload.

## What's different from Ethereum

The Ethereum sketch in [`../ethereum/storage.md`](../ethereum/storage.md) is the foundation. OP Stack adds:

1. **L1 origin per L2 block** — every L2 block is anchored to one L1 block. Required for L1-fee attribution and for joining to L1 events.
2. **Deposit-tx fields** (`source_hash`, `mint`, `is_system_tx`).
3. **L1-fee receipt fields** (formula version-dependent).
4. **Three finality heads** instead of one (`unsafe`/`safe`/`finalized`).
5. **Cross-domain event tables** linking L1 and L2 sides of bridges.

## Postgres sketch

### Block + L1 origin

```sql
CREATE TABLE l2_blocks (
    number          BIGINT PRIMARY KEY,
    hash            BYTEA NOT NULL UNIQUE,
    parent_hash     BYTEA NOT NULL,
    timestamp       BIGINT NOT NULL,
    state_root      BYTEA NOT NULL,
    receipts_root   BYTEA NOT NULL,
    base_fee        NUMERIC(78,0) NOT NULL,
    -- L1 origin for derivation
    l1_origin_num   BIGINT NOT NULL,
    l1_origin_hash  BYTEA NOT NULL,
    sequence_number INTEGER NOT NULL,  -- index within l1_origin
    -- Finality tracking
    finality        SMALLINT NOT NULL  -- 0 unsafe, 1 safe, 2 finalized
);
CREATE INDEX ON l2_blocks (l1_origin_num);
CREATE INDEX ON l2_blocks (finality, number) WHERE finality < 2;
```

### Transactions (with deposit fields)

```sql
CREATE TABLE l2_transactions (
    block_number    BIGINT NOT NULL REFERENCES l2_blocks(number) ON DELETE CASCADE,
    tx_index        INTEGER NOT NULL,
    hash            BYTEA NOT NULL UNIQUE,
    type            SMALLINT NOT NULL,        -- 0,1,2,4 for user; 0x7E (126) for deposit
    "from"          BYTEA NOT NULL,
    "to"            BYTEA,                    -- NULL on contract creation
    value           NUMERIC(78,0) NOT NULL,
    gas             BIGINT NOT NULL,
    nonce           BIGINT,                   -- NULL for deposits
    -- Deposit-only fields
    source_hash     BYTEA,                    -- NOT NULL when type = 126
    mint            NUMERIC(78,0),            -- NOT NULL when type = 126
    is_system_tx    BOOLEAN NOT NULL DEFAULT FALSE,
    l1_from         BYTEA,                    -- unaliased L1 sender (deposits only)
    PRIMARY KEY (block_number, tx_index)
);
CREATE INDEX ON l2_transactions ("from");
CREATE INDEX ON l2_transactions ("to");
CREATE INDEX ON l2_transactions (source_hash) WHERE source_hash IS NOT NULL;
```

### Receipts (with L1-fee fields, formula-versioned)

```sql
CREATE TABLE l2_receipts (
    block_number     BIGINT NOT NULL,
    tx_index         INTEGER NOT NULL,
    cumulative_gas   BIGINT NOT NULL,
    gas_used         BIGINT NOT NULL,
    status           SMALLINT NOT NULL,
    contract_address BYTEA,
    -- L1-fee inputs (raw — recompute the formula at query time if needed)
    l1_fee           NUMERIC(78,0),
    l1_gas_used      NUMERIC(78,0),
    l1_gas_price     NUMERIC(78,0),
    l1_blob_base_fee NUMERIC(78,0),
    l1_fee_scalar         NUMERIC(78,0),  -- pre-Ecotone
    l1_base_fee_scalar    BIGINT,         -- post-Ecotone
    l1_blob_base_fee_scalar BIGINT,       -- post-Ecotone
    formula_version  SMALLINT NOT NULL,   -- 1=Bedrock, 2=Ecotone, 3=Fjord, …
    PRIMARY KEY (block_number, tx_index)
);
```

`formula_version` is the indexer's responsibility — derive it at write time from the L2 block timestamp + the rollup config's fork activations. Storing the version with the row makes recomputation idempotent.

### Bridge state machine

```sql
CREATE TABLE bridge_withdrawals (
    withdrawal_hash         BYTEA PRIMARY KEY,        -- hash from MessagePassed
    l2_block_number         BIGINT NOT NULL,
    l2_tx_hash              BYTEA NOT NULL,
    "from"                  BYTEA NOT NULL,
    "to"                    BYTEA NOT NULL,
    value                   NUMERIC(78,0) NOT NULL,
    -- Lifecycle
    posted_in_output_root   BYTEA,                    -- L1 output root that included this
    posted_at_l1_block      BIGINT,
    proven_at_l1_block      BIGINT,
    proven_at_l1_tx         BYTEA,
    finalized_at_l1_block   BIGINT,
    finalized_at_l1_tx      BYTEA
);
CREATE INDEX ON bridge_withdrawals ("from");
CREATE INDEX ON bridge_withdrawals (l2_block_number);
CREATE INDEX ON bridge_withdrawals (finalized_at_l1_block) WHERE finalized_at_l1_block IS NULL;
```

A row enters at L2 burn time; the four `*_at_l1_block` columns are filled in as the user advances the state machine.

## ClickHouse sketch

For event/trace volume. Partitioning by `toYYYYMM(toDateTime(timestamp))` is conventional but at 2-second blocks, monthly partitions get large — consider weekly.

```sql
CREATE TABLE l2_logs
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
    l1_origin_num      UInt64,
    finality           UInt8                       -- 0/1/2
)
ENGINE = ReplacingMergeTree(block_number)
PARTITION BY toMonday(timestamp)
ORDER BY (address, topic0, block_number, log_index)
SETTINGS index_granularity = 8192;
```

Ordering by `(address, topic0, …)` is the right default for event scans. For address-only scans (`SELECT * FROM l2_logs WHERE address = ?`), this is fast. For topic0-only scans across all addresses, add a projection or a secondary table.

`ReplacingMergeTree(block_number)` lets you handle reorgs by re-ingesting with the same primary key — the old row is replaced on merge. **Note** that until the merge runs, both rows are visible; queries should `FINAL` or filter on `finality >= 1` for correctness.

## Choosing PG vs CH

| Workload | Pick |
|---|---|
| Bridge state machine, account history, auditable ledger | **Postgres** |
| Token transfer history per address | Either; PG with the right indexes scales to ~10B rows |
| `eth_getLogs`-shaped queries at scale | **ClickHouse** |
| Ad-hoc analytics ("how many DEX swaps per hour") | **ClickHouse** |
| Joins to L1 contract events | **Postgres** (PG handles cross-table joins better) |

Many production indexers use PG as the **source of truth** (block stream, finality state, bridge state) and CH as a **derived analytics store** (logs, traces, decoded events). The CH side is reproducible from the PG side + raw logs.

## Reorg handling at the storage layer

Two patterns:

1. **Cascade-delete on `l2_blocks`.** Easy to reason about; works in PG. Foreign keys with `ON DELETE CASCADE` clean up txs, receipts, logs. Slow if reorg depth is large.
2. **Versioned rows.** Add `is_canonical BOOLEAN` to every table; on reorg, flip canonicality rather than delete. Faster, but every read needs `WHERE is_canonical`.

Pattern (1) is fine on L2 because reorg depths are small in practice. Pattern (2) becomes worthwhile if you also need point-in-time historical queries.

For the unsafe head: shadow into a separate set of tables (`l2_blocks_unsafe`, etc.) and only promote to canonical when `safe_l2` advances. This avoids rewriting the canonical store on every sequencer hiccup.
