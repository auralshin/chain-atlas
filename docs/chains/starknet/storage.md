# Storage

StarkNet schemas are fundamentally different from EVM-baseline schemas because the data shapes are different (felt252, classes vs contracts, state diffs as first-class).

## Postgres sketch

### Block

```sql
CREATE TABLE l2_blocks (
    number              BIGINT PRIMARY KEY,
    hash                BYTEA NOT NULL UNIQUE,    -- felt252 → 32-byte buffer
    parent_hash         BYTEA NOT NULL,
    timestamp           BIGINT NOT NULL,
    new_root            BYTEA NOT NULL,
    sequencer_address   BYTEA NOT NULL,
    starknet_version    TEXT NOT NULL,
    -- Fee parameters
    l1_gas_price_wei    NUMERIC(78,0) NOT NULL,
    l1_gas_price_fri    NUMERIC(78,0) NOT NULL,
    l1_data_gas_price_wei NUMERIC(78,0),
    l1_data_gas_price_fri NUMERIC(78,0),
    l1_da_mode          TEXT NOT NULL,             -- 'BLOB' or 'CALLDATA'
    -- Finality
    status              TEXT NOT NULL              -- 'PENDING' / 'ACCEPTED_ON_L2' / 'ACCEPTED_ON_L1'
);
CREATE INDEX ON l2_blocks (status, number);
```

### Transactions

```sql
CREATE TABLE l2_transactions (
    block_number      BIGINT,                       -- NULL for pending
    tx_index          INTEGER,                      -- NULL for pending
    hash              BYTEA NOT NULL UNIQUE,
    type              TEXT NOT NULL,                -- 'INVOKE' / 'DECLARE' / 'DEPLOY_ACCOUNT' / 'L1_HANDLER' / 'DEPLOY'
    version           SMALLINT NOT NULL,            -- 0/1/2/3
    sender_address    BYTEA,                        -- For INVOKE / DECLARE / DEPLOY_ACCOUNT
    nonce             BYTEA,                        -- felt
    -- INVOKE
    calldata          BYTEA[],                      -- Array of felts
    signature         BYTEA[],                      -- Array of felts
    -- DECLARE
    class_hash        BYTEA,
    compiled_class_hash BYTEA,
    -- DEPLOY_ACCOUNT
    contract_address_salt BYTEA,
    constructor_calldata  BYTEA[],
    -- L1_HANDLER
    entry_point_selector BYTEA,
    -- Fees (v3)
    max_l1_gas_amount    NUMERIC(78,0),
    max_l1_gas_price     NUMERIC(78,0),
    max_l2_gas_amount    NUMERIC(78,0),
    max_l2_gas_price     NUMERIC(78,0),
    max_l1_data_gas_amount NUMERIC(78,0),
    max_l1_data_gas_price NUMERIC(78,0),
    tip                  NUMERIC(78,0),
    paymaster_data       BYTEA[],
    -- Status
    execution_status     TEXT,                      -- 'SUCCEEDED' / 'REVERTED'
    revert_reason        TEXT
);
CREATE INDEX ON l2_transactions (sender_address);
CREATE INDEX ON l2_transactions (type);
CREATE INDEX ON l2_transactions (class_hash) WHERE class_hash IS NOT NULL;
```

The schema is wide because StarkNet has multiple tx types with disjoint field sets. Many columns will be NULL for any given row; that's fine.

### Receipts (with messages and execution resources)

```sql
CREATE TABLE l2_receipts (
    block_number       BIGINT NOT NULL,
    tx_index           INTEGER NOT NULL,
    actual_fee_amount  NUMERIC(78,0) NOT NULL,
    actual_fee_unit    TEXT NOT NULL,              -- 'WEI' or 'FRI'
    -- Execution resources (Cairo VM steps and builtins)
    steps              BIGINT NOT NULL,
    memory_holes       BIGINT,
    range_check        BIGINT,
    pedersen           BIGINT,
    poseidon           BIGINT,
    ecdsa              BIGINT,
    bitwise            BIGINT,
    keccak             BIGINT,
    ec_op              BIGINT,
    -- L2 → L1 messages emitted by this tx (denormalized count; full data in separate table)
    l2_to_l1_messages_count INTEGER NOT NULL DEFAULT 0,
    PRIMARY KEY (block_number, tx_index)
);
```

### Events

```sql
CREATE TABLE l2_events (
    block_number       BIGINT NOT NULL,
    tx_index           INTEGER NOT NULL,
    event_index        INTEGER NOT NULL,
    from_address       BYTEA NOT NULL,
    keys               BYTEA[] NOT NULL,
    data               BYTEA[] NOT NULL,
    PRIMARY KEY (block_number, tx_index, event_index)
);
CREATE INDEX ON l2_events (from_address);
CREATE INDEX ON l2_events ((keys[1])) WHERE array_length(keys, 1) >= 1;  -- selector, if present
```

Fields are felt arrays — `BYTEA[]` is a natural fit in Postgres.

### Classes (immutable; cache forever)

```sql
CREATE TABLE l2_classes (
    class_hash        BYTEA PRIMARY KEY,
    declared_at_block BIGINT NOT NULL,
    declared_in_tx    BYTEA NOT NULL,
    is_cairo_1        BOOLEAN NOT NULL,           -- false = Cairo 0
    -- Code: large; consider separate table or external storage
    abi_json          JSONB NOT NULL,
    sierra_program    BYTEA,                       -- Cairo 1 only; can be very large
    casm_bytecode     BYTEA,                       -- Cairo 0 program or compiled CASM
    compiled_class_hash BYTEA                       -- Cairo 1 only
);
```

Classes can be hundreds of KB to multiple MB each. Most indexers store classes in object storage (S3) and keep the metadata in Postgres.

### Contracts (address → class_hash)

```sql
CREATE TABLE l2_contracts (
    address            BYTEA PRIMARY KEY,
    class_hash         BYTEA NOT NULL,             -- current class
    deployed_at_block  BIGINT NOT NULL,
    deployed_in_tx     BYTEA NOT NULL
);

CREATE TABLE l2_class_replacements (
    address            BYTEA NOT NULL,
    block_number       BIGINT NOT NULL,
    old_class_hash     BYTEA NOT NULL,
    new_class_hash     BYTEA NOT NULL,
    PRIMARY KEY (address, block_number)
);
```

`replace_class` syscall is rare but real — track changes if your application cares.

### State diffs

```sql
CREATE TABLE l2_storage_diffs (
    block_number       BIGINT NOT NULL,
    contract_address   BYTEA NOT NULL,
    storage_key        BYTEA NOT NULL,
    value              BYTEA NOT NULL,
    PRIMARY KEY (block_number, contract_address, storage_key)
);
```

Useful for state-tracking applications. Volume can be very large; consider whether you need this table at all.

### L1 ↔ L2 messaging

```sql
CREATE TABLE l1_to_l2_messages (
    nonce             BYTEA PRIMARY KEY,           -- felt; from L1 LogMessageToL2 event
    l1_block          BIGINT NOT NULL,
    l1_tx             BYTEA NOT NULL,
    from_l1_address   BYTEA NOT NULL,
    to_address        BYTEA NOT NULL,
    selector          BYTEA NOT NULL,
    payload           BYTEA[] NOT NULL,
    -- L2 execution
    l2_block_number   BIGINT,                      -- NULL until L1_HANDLER tx executes
    l2_tx_hash        BYTEA,
    status            TEXT NOT NULL                -- 'queued' / 'executed'
);

CREATE TABLE l2_to_l1_messages (
    block_number       BIGINT NOT NULL,
    tx_hash            BYTEA NOT NULL,
    message_index      INTEGER NOT NULL,
    from_address       BYTEA NOT NULL,
    to_l1_address      BYTEA NOT NULL,
    payload            BYTEA[] NOT NULL,
    -- L1 consumption
    l1_consumed_block  BIGINT,
    l1_consumed_tx     BYTEA,
    PRIMARY KEY (block_number, tx_hash, message_index)
);
```

## ClickHouse sketch

For event volume:

```sql
CREATE TABLE l2_events
(
    block_number       UInt64,
    tx_index           UInt32,
    event_index        UInt32,
    block_hash         FixedString(32),
    from_address       FixedString(32),             -- felt252 → 32 bytes
    selector           Nullable(FixedString(32)),   -- keys[0] if present
    keys_count         UInt8,
    data_count         UInt32,
    keys               Array(FixedString(32)),
    data               Array(FixedString(32)),
    timestamp          DateTime,
    finality           LowCardinality(String)       -- 'PENDING' / 'ACCEPTED_ON_L2' / 'ACCEPTED_ON_L1'
)
ENGINE = ReplacingMergeTree(block_number)
PARTITION BY toMonday(timestamp)
ORDER BY (from_address, selector, block_number, tx_index, event_index)
SETTINGS index_granularity = 8192;
```

`Array(FixedString(32))` is a natural fit for felt arrays.

## Reorg handling

Same shape as EVM rollups: subscribe to new heads, walk back on parent-hash mismatch, advance `status` field as L1 events arrive.

`status` advances `PENDING → ACCEPTED_ON_L2 → ACCEPTED_ON_L1`. PENDING data is volatile; commit it to `*_pending` shadow tables and only promote on seal.

## Choosing PG vs CH

| Workload | Pick |
|---|---|
| Class registry, contract → class_hash mapping | **Postgres** |
| L1↔L2 message lifecycle | **Postgres** |
| Event-based analytics (DEX volume etc.) | **ClickHouse** |
| State diff history | **ClickHouse** (very high volume) |
| Account-level indexing (txs per account) | **Postgres** for canonical, **ClickHouse** for aggregates |

## Sizing

StarkNet's data volume is moderate. State diffs alone can be substantial (every storage write is recorded); decide whether you need the diffs or just the latest state.
