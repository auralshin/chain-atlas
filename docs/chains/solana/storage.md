# Storage

Solana storage is **ClickHouse-first**. Postgres works for canonical metadata but the per-tx, per-instruction, per-account-write volume saturates Postgres faster than any chain in this guide.

## Volume reality check

Approximate scale (mainnet, 2025):
- ~3,000+ user txs / second (excluding vote txs).
- ~50–100 instructions per second from CPIs and inner instructions.
- ~10,000+ account writes per second.

Per day: hundreds of GB of raw events. Per year: hundreds of TB raw, ~tens of TB compressed in ClickHouse.

For comparison, Ethereum is ~15 txs/sec. Solana's volume is **two orders of magnitude higher**.

## ClickHouse sketch (primary)

### Slots

```sql
CREATE TABLE slots
(
    slot               UInt64,
    block_height       Nullable(UInt64),           -- NULL if slot was skipped
    block_hash         Nullable(FixedString(32)),
    parent_slot        UInt64,
    leader             FixedString(32),
    block_time         DateTime,
    finalized_at       Nullable(DateTime),
    confirmed_at       Nullable(DateTime)
)
ENGINE = ReplacingMergeTree(slot)
PARTITION BY toMonday(block_time)
ORDER BY slot
SETTINGS index_granularity = 8192;
```

### Transactions

```sql
CREATE TABLE transactions
(
    slot               UInt64,
    block_height       UInt64,
    tx_index           UInt32,
    signature          FixedString(64),            -- ed25519 signature, hex
    fee_payer          FixedString(32),
    err                Nullable(String),
    fee_lamports       UInt64,
    compute_units      UInt32,
    is_vote_tx         UInt8,                      -- filter target
    version            LowCardinality(String),     -- 'legacy' or '0'
    timestamp          DateTime
)
ENGINE = MergeTree
PARTITION BY toMonday(timestamp)
ORDER BY (slot, tx_index)
SETTINGS index_granularity = 8192;
```

### Instructions (top-level + inner)

```sql
CREATE TABLE instructions
(
    slot               UInt64,
    tx_index           UInt32,
    instruction_path   Array(UInt32),              -- [0] for top-level idx 0; [0, 1, 2] for nested
    program_id         FixedString(32),
    accounts           Array(FixedString(32)),     -- ordered as referenced
    data               String,
    timestamp          DateTime
)
ENGINE = MergeTree
PARTITION BY toMonday(timestamp)
ORDER BY (program_id, slot, tx_index, instruction_path)
SETTINGS index_granularity = 8192;
```

`instruction_path` encodes both depth and position: `[0]` = top-level instruction 0; `[0, 1]` = first inner instruction inside top-level 0. This is the canonical way to represent the CPI tree in a flat table.

`(program_id, slot, ...)` ordering makes "all instructions to program X" queries fast.

### Account state diffs

```sql
CREATE TABLE account_writes
(
    slot               UInt64,
    pubkey             FixedString(32),
    owner              FixedString(32),
    lamports           UInt64,
    data_len           UInt32,
    data_hash          FixedString(32),            -- hash of data; full data optional
    is_writable        UInt8,
    timestamp          DateTime
)
ENGINE = MergeTree
PARTITION BY toMonday(timestamp)
ORDER BY (pubkey, slot)
SETTINGS index_granularity = 8192;
```

Storing full account data per-write is prohibitive. Most indexers store the **hash + length** by default and fetch full data on-demand for the addresses they care about.

### Token transfers (denormalized for performance)

```sql
CREATE TABLE token_transfers
(
    slot               UInt64,
    tx_index           UInt32,
    instruction_path   Array(UInt32),
    program            LowCardinality(String),     -- 'spl-token' or 'token-2022'
    mint               FixedString(32),
    "from"             FixedString(32),
    "to"               FixedString(32),
    amount             UInt256,
    decimals           UInt8,
    timestamp          DateTime
)
ENGINE = MergeTree
PARTITION BY toMonday(timestamp)
ORDER BY (mint, slot, tx_index)
SETTINGS index_granularity = 8192;
```

Derived from `pre/postTokenBalances` deltas + Token program inner instructions. Recomputable from raw data.

### Logs (text logs from `msg!()`)

```sql
CREATE TABLE logs
(
    slot               UInt64,
    tx_index           UInt32,
    log_index          UInt32,
    program_id         Nullable(FixedString(32)),  -- if parsed; many logs are program-prefixed
    log_text           String,
    is_data_log        UInt8,                      -- 1 if 'Program data: ...' (base64)
    timestamp          DateTime
)
ENGINE = MergeTree
PARTITION BY toMonday(timestamp)
ORDER BY (slot, tx_index, log_index)
SETTINGS index_granularity = 8192;
```

Useful for ad-hoc debugging; for serious dApp indexing, parse logs at ingest into structured tables.

## Postgres sketch (canonical metadata)

For things that benefit from referential integrity and joins:

```sql
CREATE TABLE programs (
    program_id         BYTEA PRIMARY KEY,
    name               TEXT,
    is_anchor          BOOLEAN,
    idl_json           JSONB,
    deployed_at_slot   BIGINT,
    last_upgrade_slot  BIGINT
);

CREATE TABLE token_mints (
    mint               BYTEA PRIMARY KEY,
    program            TEXT NOT NULL,              -- 'spl-token' or 'token-2022'
    decimals           SMALLINT NOT NULL,
    supply             NUMERIC(78,0) NOT NULL,
    name               TEXT,
    symbol             TEXT,
    is_initialized_at_slot BIGINT NOT NULL
);

CREATE TABLE address_lookup_tables (
    address            BYTEA PRIMARY KEY,
    authority          BYTEA,
    last_extended_slot BIGINT,
    deactivation_slot  BIGINT,
    addresses          BYTEA[]                     -- current set; appends only
);
```

ALT state is mutable but additive; storing the current set is sufficient if you also ingest historical `loadedAddresses` per V0 tx.

## Geyser ingestion architecture

Production indexers consume Geyser gRPC and write to ClickHouse via:

1. **Async streaming consumer** (Rust async task with `yellowstone-grpc-client`).
2. **Buffered writes** (batch inserts to ClickHouse — ClickHouse loves batch sizes of 10k–100k rows).
3. **Out-of-order tolerance** (slots arrive close-to-in-order but not always; `ReplacingMergeTree` handles re-ingestion idempotently).

Backpressure from ClickHouse is rare if batched correctly; from a cold start, ingestion lag is usually limited by Geyser's stream rate, not DB writes.

## Reorg handling

Re-ingestion on slot status transitions:
- `processed` slot data: write to `_pending` shadow tables (or skip; commit only on `confirmed`).
- `confirmed`: write to canonical tables. `ReplacingMergeTree` on slot deduplicates if the same slot is ingested again.
- `finalized`: nothing to do beyond updating the slot record's `finalized_at` timestamp.

For deep-fork-rare reorgs at `confirmed`: re-fetch the affected slots and re-ingest. `ReplacingMergeTree` resolves duplicates.

## Multi-tenant / multi-cluster

If you index Solana mainnet + devnet + a Solana-VM L2 (Eclipse, Sonic, etc.), partition all tables by `cluster_id`. Most teams keep separate clickhouse databases per cluster.

## Choosing PG vs CH (for Solana specifically)

| Workload | Pick |
|---|---|
| Canonical: program registry, mint registry, ALT registry | **Postgres** |
| Token transfers history | **ClickHouse** |
| Per-address activity scans | **ClickHouse** |
| Per-program instruction analytics | **ClickHouse** |
| Lookup of "this signature's full tx" | **ClickHouse** (or hot blob storage) |

For Solana, PG is a small island of metadata; CH is the main warehouse. **This is the opposite of EVM chains** where PG can carry most workloads.
