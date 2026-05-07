# Storage

Polygon-specific storage adds **state-sync tracking**, **checkpoint mapping**, and a **deeper reorg buffer** on top of the standard ETH-baseline schemas.

## Postgres sketch

### Block

```sql
CREATE TABLE bor_blocks (
    number          BIGINT PRIMARY KEY,
    hash            BYTEA NOT NULL UNIQUE,
    parent_hash     BYTEA NOT NULL,
    timestamp       BIGINT NOT NULL,
    miner           BYTEA NOT NULL,                -- sprint producer
    state_root      BYTEA NOT NULL,
    base_fee        NUMERIC(78,0),                 -- post-London only
    -- Polygon-specific
    sprint_index    INTEGER NOT NULL,              -- which sprint within the span
    span_id         BIGINT NOT NULL,
    checkpoint_id   BIGINT,                        -- NULL until the block is checkpointed on L1
    -- Finality
    finality        SMALLINT NOT NULL              -- 0 unconfirmed, 1 deep-confirmed (>256), 2 checkpointed
);
CREATE INDEX ON bor_blocks (miner);
CREATE INDEX ON bor_blocks (checkpoint_id) WHERE checkpoint_id IS NOT NULL;
CREATE INDEX ON bor_blocks (finality, number) WHERE finality < 2;
```

### Checkpoints

```sql
CREATE TABLE checkpoints (
    checkpoint_id        BIGINT PRIMARY KEY,
    start_block          BIGINT NOT NULL,
    end_block            BIGINT NOT NULL,
    root_hash            BYTEA NOT NULL,
    proposer             BYTEA NOT NULL,
    -- L1 anchoring
    l1_block_number      BIGINT NOT NULL,
    l1_tx_hash           BYTEA NOT NULL,
    l1_finalized         BOOLEAN NOT NULL DEFAULT FALSE
);
CREATE INDEX ON checkpoints (start_block, end_block);
```

### State syncs

```sql
CREATE TABLE state_syncs (
    state_id          BIGINT PRIMARY KEY,          -- heimdall-issued
    -- L1 origination
    l1_block_number   BIGINT NOT NULL,
    l1_tx_hash        BYTEA NOT NULL,
    l1_log_index      INTEGER NOT NULL,
    contract_address  BYTEA NOT NULL,              -- destination L2 contract
    data              BYTEA NOT NULL,
    -- L2 execution
    l2_block_number   BIGINT,                      -- NULL until executed on bor
    l2_executed_at    TIMESTAMPTZ,
    status            SMALLINT NOT NULL            -- 0 pending, 1 executed, 2 failed (rare)
);
CREATE INDEX ON state_syncs (status, state_id) WHERE status = 0;
CREATE INDEX ON state_syncs (l2_block_number) WHERE l2_block_number IS NOT NULL;
CREATE INDEX ON state_syncs (contract_address);
```

State-syncs are keyed on `state_id` (heimdall-issued, monotonic) — **not** on bor block + log index — because their bor injection point can shift on reorgs.

### Validator state

Optional but recommended for bridge-aware indexers:

```sql
CREATE TABLE heimdall_validators (
    validator_id     BIGINT PRIMARY KEY,
    bor_address      BYTEA NOT NULL,
    heimdall_address BYTEA NOT NULL,
    signer_pubkey    BYTEA NOT NULL,
    voting_power     BIGINT NOT NULL,
    activated_at     TIMESTAMPTZ NOT NULL,
    deactivated_at   TIMESTAMPTZ
);
CREATE INDEX ON heimdall_validators (bor_address);

CREATE TABLE spans (
    span_id          BIGINT PRIMARY KEY,
    start_block      BIGINT NOT NULL,
    end_block        BIGINT NOT NULL,
    selected_producers BIGINT[] NOT NULL           -- validator_ids elected for this span
);
```

### Withdrawals (L2 → L1)

```sql
CREATE TABLE bor_withdrawals (
    burn_tx_hash      BYTEA PRIMARY KEY,
    burn_block_number BIGINT NOT NULL,
    "from"            BYTEA NOT NULL,
    "to"              BYTEA NOT NULL,
    token_address     BYTEA NOT NULL,
    amount            NUMERIC(78,0) NOT NULL,
    -- Lifecycle
    checkpoint_id     BIGINT,                      -- the checkpoint that included this burn block
    proven_at_l1_tx   BYTEA,
    exited_at_l1_tx   BYTEA,
    exited_at         TIMESTAMPTZ
);
CREATE INDEX ON bor_withdrawals ("from");
CREATE INDEX ON bor_withdrawals (exited_at) WHERE exited_at IS NULL;
```

## ClickHouse sketch

For event volume, partitioned by week given Polygon's tx volume.

```sql
CREATE TABLE bor_logs
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
    finality           UInt8,                       -- 0/1/2
    is_state_sync_log  UInt8                        -- 1 if log was emitted by state-sync, 0 otherwise
)
ENGINE = ReplacingMergeTree(block_number)
PARTITION BY toMonday(timestamp)
ORDER BY (address, topic0, block_number, log_index)
SETTINGS index_granularity = 8192;
```

The `is_state_sync_log` column lets you filter "natural" L2 contract logs from those triggered by L1→L2 syncs — useful for correctly attributing activity to user actions vs bridge-injected actions.

## Reorg handling — extra-deep buffer

The reorg buffer must be sized for Polygon's worst-case observed depths:

```sql
-- Mark blocks as "settled" only when number < latest - REORG_DEPTH
-- where REORG_DEPTH = 256 minimum (ETH uses ~12)
-- Or, better: only when checkpoint_id IS NOT NULL (full finality)
```

Cascade-delete on bor_blocks is fine but expensive when reorg depth is large. Consider:
- Keeping a **shadow table** for unconfirmed data and only promoting after `latest - 256`.
- Or maintaining `is_canonical` flags and flipping on reorg without delete.

For state-syncs, **never delete on reorg** — the `state_id` is canonical. On reorg, just update `(l2_block_number, l2_executed_at)` to point to the new injection block.

## Multi-stream consistency

A Polygon indexer reads three streams: bor, heimdall, L1. Reorgs in any of them affect storage consistency:

- **L1 reorg**: orphan checkpoint posting txs and state-sync `StateSynced` events. Update affected `state_syncs.l1_block_number` rows. Update `checkpoints.l1_block_number` rows.
- **Heimdall reorg**: rare (Tendermint BFT), but if it happens: re-fetch checkpoints and state-sync queue from heimdall.
- **Bor reorg**: re-derive bor_blocks rows; state-sync execution rows update `l2_block_number`.

The streams are not synchronized — design for each independently with idempotent updates.

## Choosing PG vs CH

| Workload | Pick |
|---|---|
| State-sync lifecycle, bridge accounting, checkpoint joins | **Postgres** |
| Token transfer history per address | Either |
| `eth_getLogs`-shaped queries at scale | **ClickHouse** |
| Validator analytics, span / sprint statistics | **Postgres** |

Polygon's high tx volume + deep reorgs make ClickHouse the practical choice for log/event analytics; PG is the practical choice for any workload that joins to checkpoints, state-syncs, or validator state.
