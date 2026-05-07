# Storage

Arbitrum-specific schema deltas on top of the Ethereum baseline. Same overall shape as the OP Stack schemas, but with Arbitrum's distinct tx types, receipt fields, and the retryable + outbox state machines.

## Postgres sketch

### Block

```sql
CREATE TABLE l2_blocks (
    chain_id        INTEGER NOT NULL,            -- 42161, 42170, …
    number          BIGINT NOT NULL,
    hash            BYTEA NOT NULL,
    parent_hash     BYTEA NOT NULL,
    timestamp       BIGINT NOT NULL,
    state_root      BYTEA NOT NULL,
    receipts_root   BYTEA NOT NULL,
    base_fee        NUMERIC(78,0) NOT NULL,
    -- Arbitrum-specific
    l1_block_number BIGINT NOT NULL,             -- sequencer's L1 view at production
    send_root       BYTEA NOT NULL,
    send_count      BIGINT NOT NULL,
    -- Batch correlation (filled in when SequencerBatchDelivered is observed)
    batch_index     BIGINT,                      -- L1 batch index containing this block
    batch_l1_block  BIGINT,
    batch_l1_tx     BYTEA,
    -- Finality
    finality        SMALLINT NOT NULL,           -- 0 sequencer, 1 L1-confirmed, 2 finalized
    PRIMARY KEY (chain_id, number)
);
CREATE INDEX ON l2_blocks (chain_id, hash);
CREATE INDEX ON l2_blocks (chain_id, batch_index) WHERE batch_index IS NOT NULL;
CREATE INDEX ON l2_blocks (chain_id, finality, number) WHERE finality < 2;
```

`batch_index` and friends are NULL when a block is at sequencer-feed level only; populated when the L1 batch is observed.

### Transactions

```sql
CREATE TABLE l2_transactions (
    chain_id        INTEGER NOT NULL,
    block_number    BIGINT NOT NULL,
    tx_index        INTEGER NOT NULL,
    hash            BYTEA NOT NULL UNIQUE,
    type            SMALLINT NOT NULL,
    "from"          BYTEA NOT NULL,
    "to"            BYTEA,
    value           NUMERIC(78,0) NOT NULL,
    gas             BIGINT NOT NULL,
    nonce           BIGINT,
    -- Arbitrum system-tx fields (NULL for user txs)
    request_id      BYTEA,                       -- For 0x64-0x6A: identifies the inbox message
    l1_from         BYTEA,                       -- Unaliased L1 origin address
    retryable_ticket_id BYTEA,                   -- For 0x68 redeems: which ticket
    PRIMARY KEY (chain_id, block_number, tx_index),
    FOREIGN KEY (chain_id, block_number) REFERENCES l2_blocks(chain_id, number) ON DELETE CASCADE
);
CREATE INDEX ON l2_transactions (chain_id, "from");
CREATE INDEX ON l2_transactions (chain_id, "to");
CREATE INDEX ON l2_transactions (chain_id, type) WHERE type >= 100;  -- 0x64+ Arbitrum system
CREATE INDEX ON l2_transactions (retryable_ticket_id) WHERE retryable_ticket_id IS NOT NULL;
```

### Receipts

```sql
CREATE TABLE l2_receipts (
    chain_id            INTEGER NOT NULL,
    block_number        BIGINT NOT NULL,
    tx_index            INTEGER NOT NULL,
    cumulative_gas      BIGINT NOT NULL,
    gas_used            BIGINT NOT NULL,
    gas_used_for_l1     BIGINT NOT NULL,
    effective_gas_price NUMERIC(78,0) NOT NULL,
    status              SMALLINT NOT NULL,
    contract_address    BYTEA,
    PRIMARY KEY (chain_id, block_number, tx_index)
);
```

The L1-fee accounting is `gas_used_for_l1 * effective_gas_price` — store the inputs, recompute fees in queries.

### Retryable ticket lifecycle

```sql
CREATE TABLE retryable_tickets (
    chain_id              INTEGER NOT NULL,
    ticket_id             BYTEA NOT NULL,
    -- Submission
    submit_l2_block       BIGINT NOT NULL,
    submit_l2_tx          BYTEA NOT NULL,         -- the 0x69 tx
    l1_request_id         BYTEA NOT NULL,
    l1_from               BYTEA NOT NULL,
    "to"                  BYTEA NOT NULL,
    value                 NUMERIC(78,0) NOT NULL,
    callvalue             NUMERIC(78,0) NOT NULL,
    submission_fee        NUMERIC(78,0) NOT NULL,
    -- Lifecycle
    state                 SMALLINT NOT NULL,      -- 0 pending, 1 redeemed_auto, 2 redeemed_manual, 3 expired, 4 cancelled
    auto_redeem_l2_tx     BYTEA,
    manual_redeem_l2_tx   BYTEA,
    expires_at            BIGINT NOT NULL,        -- L2 timestamp when ticket expires
    PRIMARY KEY (chain_id, ticket_id)
);
CREATE INDEX ON retryable_tickets (chain_id, state) WHERE state = 0;  -- pending tickets
CREATE INDEX ON retryable_tickets (l1_from);
```

### Outbox / withdrawal lifecycle

```sql
CREATE TABLE outbox_messages (
    chain_id              INTEGER NOT NULL,
    message_index         BIGINT NOT NULL,        -- from L2ToL1Tx event
    -- L2 side
    l2_block_number       BIGINT NOT NULL,
    l2_tx_hash            BYTEA NOT NULL,
    "from"                BYTEA NOT NULL,
    "to"                  BYTEA NOT NULL,
    value                 NUMERIC(78,0) NOT NULL,
    data                  BYTEA NOT NULL,
    -- L1 confirmation
    contained_in_send_root  BYTEA,
    confirmed_at_l1_block   BIGINT,               -- assertion confirmed
    confirmed_via_assertion BYTEA,                -- legacy nodeId or BoLD edgeId
    -- L1 execution
    executed_at_l1_block    BIGINT,
    executed_at_l1_tx       BYTEA,
    PRIMARY KEY (chain_id, message_index)
);
CREATE INDEX ON outbox_messages ("from");
CREATE INDEX ON outbox_messages (chain_id, executed_at_l1_block) WHERE executed_at_l1_block IS NULL;
```

## ClickHouse sketch

For event/trace volume on Arbitrum, similar shape to OP Stack with chain-id-prefixed ordering.

```sql
CREATE TABLE l2_logs
(
    chain_id           UInt32,
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
    l1_block_number    UInt64,
    finality           UInt8
)
ENGINE = ReplacingMergeTree(block_number)
PARTITION BY (chain_id, toMonday(timestamp))
ORDER BY (chain_id, address, topic0, block_number, log_index)
SETTINGS index_granularity = 8192;
```

For traces (the high-volume table on Arbitrum thanks to `arbtrace_*` adoption):

```sql
CREATE TABLE l2_internal_txs
(
    chain_id           UInt32,
    block_number       UInt64,
    tx_index           UInt32,
    trace_address      Array(UInt32),
    type               LowCardinality(String),    -- call, create, suicide
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
PARTITION BY (chain_id, toMonday(timestamp))
ORDER BY (chain_id, "from", block_number, tx_index, trace_address)
SETTINGS index_granularity = 8192;
```

## Choosing PG vs CH

| Workload | Pick |
|---|---|
| Retryable lifecycle, outbox lifecycle, bridge accounting | **Postgres** |
| Token transfer history per address | Either |
| `eth_getLogs`-shaped queries at scale | **ClickHouse** |
| Trace-based analytics ("how much DEX volume routed through this aggregator") | **ClickHouse** with internal-tx table |
| Joins between L2 logs and L1 events | **Postgres** |

## Reorg handling

Same two patterns as OP Stack: cascade-delete via FK, or versioned rows. **Cascade-delete is generally fine** because Arbitrum L2 reorgs are typically shallow (sequencer rewinds of 1–3 blocks). Larger reorgs from L1 reorganizations are rare but possible — design for it.

For the sequencer feed: shadow into `_unsafe` tables and only promote to canonical when the L1-confirmed head advances past them.

## Multi-chain considerations

If you index multiple Arbitrum chains (One + Nova + Orbit chains), partition by `chain_id` everywhere — same advice as OP Stack. The Arbitrum ecosystem is moving toward many-chains-one-stack, so multi-chain is the default expectation.
