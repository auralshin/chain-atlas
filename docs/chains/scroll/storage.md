# Storage

Scroll's schema is essentially the Ethereum baseline + L1-fee receipt fields + batch lifecycle tables. Closest to the [Optimism schema](../optimism/storage.md) of any chain in this guide.

## Postgres sketch

### Block + batch correlation

```sql
CREATE TABLE l2_blocks (
    number          BIGINT PRIMARY KEY,
    hash            BYTEA NOT NULL UNIQUE,
    parent_hash     BYTEA NOT NULL,
    timestamp       BIGINT NOT NULL,
    state_root      BYTEA NOT NULL,
    base_fee        NUMERIC(78,0) NOT NULL,
    -- Batch correlation
    batch_index     BIGINT,                       -- L1 batch this block belongs to (NULL until committed)
    -- Finality
    finality        SMALLINT NOT NULL             -- 0 sequencer, 1 committed, 2 finalized
);
CREATE INDEX ON l2_blocks (batch_index) WHERE batch_index IS NOT NULL;
CREATE INDEX ON l2_blocks (finality, number) WHERE finality < 2;
```

### Batches

```sql
CREATE TABLE l1_batches (
    batch_index            BIGINT PRIMARY KEY,
    -- L2 range
    l2_block_start         BIGINT NOT NULL,
    l2_block_end           BIGINT NOT NULL,
    -- L1 lifecycle
    committed_at_l1        BIGINT,
    committed_l1_tx        BYTEA,
    finalized_at_l1        BIGINT,
    finalized_l1_tx        BYTEA,
    -- Format / version
    encoding_version       SMALLINT NOT NULL      -- bumped at Darwin compression changes
);
```

### Transactions and receipts

Transactions are standard ETH-shape. Receipts include L1-fee fields:

```sql
CREATE TABLE l2_receipts (
    block_number      BIGINT NOT NULL,
    tx_index          INTEGER NOT NULL,
    cumulative_gas    BIGINT NOT NULL,
    gas_used          BIGINT NOT NULL,
    status            SMALLINT NOT NULL,
    contract_address  BYTEA,
    -- L1-fee inputs
    l1_fee            NUMERIC(78,0),
    l1_gas_used       NUMERIC(78,0),
    l1_gas_price      NUMERIC(78,0),
    l1_blob_base_fee  NUMERIC(78,0),
    l1_fee_scalar     NUMERIC(78,0),              -- pre-Bernoulli shape
    l1_base_fee_scalar BIGINT,                    -- post-Bernoulli shape
    l1_blob_base_fee_scalar BIGINT,
    formula_version   SMALLINT NOT NULL,
    PRIMARY KEY (block_number, tx_index)
);
```

### L1 → L2 messaging

```sql
CREATE TABLE l1_to_l2_messages (
    queue_index       BIGINT PRIMARY KEY,         -- from L1MessageQueue
    l1_block          BIGINT NOT NULL,
    l1_tx             BYTEA NOT NULL,
    sender_l1         BYTEA NOT NULL,
    target            BYTEA NOT NULL,
    value             NUMERIC(78,0) NOT NULL,
    data              BYTEA NOT NULL,
    -- L2 execution
    l2_block_number   BIGINT,                     -- NULL until executed
    l2_tx_hash        BYTEA,
    status            SMALLINT NOT NULL           -- 0 queued, 1 executed
);
```

### L2 → L1 messaging (withdrawals)

```sql
CREATE TABLE l2_to_l1_messages (
    l2_tx_hash             BYTEA NOT NULL,
    l2_log_index           INTEGER NOT NULL,
    l2_block_number        BIGINT NOT NULL,
    sender                 BYTEA NOT NULL,
    target                 BYTEA NOT NULL,
    value                  NUMERIC(78,0) NOT NULL,
    data                   BYTEA NOT NULL,
    -- L1 finalization
    l1_relayed_block       BIGINT,
    l1_relayed_tx          BYTEA,
    PRIMARY KEY (l2_tx_hash, l2_log_index)
);
CREATE INDEX ON l2_to_l1_messages (sender);
CREATE INDEX ON l2_to_l1_messages (l1_relayed_block) WHERE l1_relayed_block IS NULL;
```

## ClickHouse sketch

For event volume, same shape as OP Stack:

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
    batch_index        UInt64,
    finality           UInt8
)
ENGINE = ReplacingMergeTree(block_number)
PARTITION BY toMonday(timestamp)
ORDER BY (address, topic0, block_number, log_index)
SETTINGS index_granularity = 8192;
```

## Reorg handling

Scroll L2 reorgs are rare (single-sequencer + ZK rollup). Cascade-delete on `l2_blocks` works fine. Provision shallow walk-back (~10 blocks).

L1-side: subscribe to `ScrollChain.CommitBatch` and `FinalizeBatch`; advance per-block finality state; handle L1 reorgs by reverting batch status.

## Choosing PG vs CH

Same as OP Stack:

| Workload | Pick |
|---|---|
| Withdrawal lifecycle, bridge accounting | **Postgres** |
| Token transfers per address | Either |
| `eth_getLogs`-shaped queries at scale | **ClickHouse** |
| Cross-batch analytics | **ClickHouse** |

Scroll's volume is moderate; PG can hold the entire chain comfortably for several years before sizing concerns kick in. ClickHouse becomes worthwhile for log/event analytics workloads.
