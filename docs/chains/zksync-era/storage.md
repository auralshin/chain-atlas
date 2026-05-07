# Storage

zkSync Era's distinctive storage needs: paymaster + AA tracking, batch lifecycle, and three-stage L1 finality state.

## Postgres sketch

### Block + batch correlation

```sql
CREATE TABLE l2_blocks (
    number          BIGINT PRIMARY KEY,
    hash            BYTEA NOT NULL UNIQUE,
    parent_hash     BYTEA NOT NULL,
    timestamp       BIGINT NOT NULL,
    state_root      BYTEA NOT NULL,
    -- zkSync-specific
    l1_batch_number BIGINT NOT NULL,
    l1_batch_index  INTEGER NOT NULL,           -- position within batch
    -- Finality
    batch_status    SMALLINT NOT NULL           -- 0 sealed, 1 committed, 2 proven, 3 executed
);
CREATE INDEX ON l2_blocks (l1_batch_number);
CREATE INDEX ON l2_blocks (batch_status, number) WHERE batch_status < 3;
```

`batch_status` is updated as L1 events arrive; `(0 → 1 → 2 → 3)` advances over time.

### Batches

```sql
CREATE TABLE l1_batches (
    batch_number       BIGINT PRIMARY KEY,
    -- L2 range
    l2_block_start     BIGINT NOT NULL,
    l2_block_end       BIGINT NOT NULL,
    -- L1 lifecycle
    committed_at_l1    BIGINT,                  -- L1 block of BlockCommit
    committed_l1_tx    BYTEA,
    proven_at_l1       BIGINT,                  -- L1 block of BlocksVerification
    proven_l1_tx       BYTEA,
    executed_at_l1     BIGINT,                  -- L1 block of BlockExecution
    executed_l1_tx     BYTEA,
    -- Proof system
    protocol_version   INTEGER NOT NULL
);
```

### Transactions (with paymaster + AA)

```sql
CREATE TABLE l2_transactions (
    block_number      BIGINT NOT NULL,
    tx_index          INTEGER NOT NULL,
    hash              BYTEA NOT NULL UNIQUE,
    type              SMALLINT NOT NULL,        -- 0,1,2 for ETH; 0x71 (113) for EIP-712; 0xFF (255) for priority op
    "from"            BYTEA NOT NULL,           -- account contract address
    "to"              BYTEA,
    value             NUMERIC(78,0) NOT NULL,
    gas               BIGINT NOT NULL,
    nonce             BIGINT,                   -- NonceHolder-managed
    -- AA / paymaster
    paymaster         BYTEA,                    -- NULL when sender pays gas in ETH
    paymaster_input   BYTEA,                    -- NULL or paymaster-specific data
    custom_signature  BYTEA,                    -- non-ECDSA AA signatures
    factory_deps      BYTEA[],                  -- referenced bytecode hashes
    -- Priority op (type 0xFF)
    priority_request_id  BYTEA,                 -- L1 message id for priority ops
    l1_from           BYTEA,                    -- unaliased L1 origin for priority ops
    PRIMARY KEY (block_number, tx_index),
    FOREIGN KEY (block_number) REFERENCES l2_blocks(number) ON DELETE CASCADE
);
CREATE INDEX ON l2_transactions ("from");
CREATE INDEX ON l2_transactions (paymaster) WHERE paymaster IS NOT NULL;
CREATE INDEX ON l2_transactions (priority_request_id) WHERE priority_request_id IS NOT NULL;
```

### Receipts

```sql
CREATE TABLE l2_receipts (
    block_number      BIGINT NOT NULL,
    tx_index          INTEGER NOT NULL,
    cumulative_gas    BIGINT NOT NULL,
    gas_used          BIGINT NOT NULL,
    effective_gas_price NUMERIC(78,0) NOT NULL,
    status            SMALLINT NOT NULL,
    contract_address  BYTEA,                    -- For deployments tracked via ContractDeployer
    -- L2-to-L1 messages
    l2_to_l1_messages JSONB,                    -- Array of L1Messenger sends in this tx
    PRIMARY KEY (block_number, tx_index)
);
```

### L1→L2 priority ops

```sql
CREATE TABLE priority_ops (
    request_id            BYTEA PRIMARY KEY,    -- Hash from NewPriorityRequest
    l1_block_number       BIGINT NOT NULL,
    l1_tx_hash            BYTEA NOT NULL,
    l1_from               BYTEA NOT NULL,
    "to"                  BYTEA NOT NULL,
    value                 NUMERIC(78,0) NOT NULL,
    -- L2 execution
    l2_tx_hash            BYTEA,                -- The 0xFF tx that processed this on L2
    l2_block_number       BIGINT,
    status                SMALLINT NOT NULL     -- 0 pending, 1 executed, 2 failed (rare)
);
CREATE INDEX ON priority_ops (status, request_id) WHERE status = 0;
CREATE INDEX ON priority_ops (l1_from);
```

### L2→L1 withdrawals

```sql
CREATE TABLE l2_withdrawals (
    l2_tx_hash           BYTEA NOT NULL,
    l2_log_index         INTEGER NOT NULL,
    l2_block_number      BIGINT NOT NULL,
    l1_batch_number      BIGINT NOT NULL,
    "from"               BYTEA NOT NULL,
    "to"                 BYTEA NOT NULL,
    value                NUMERIC(78,0) NOT NULL,
    data                 BYTEA,
    -- L1 finalization
    finalized_at_l1_tx   BYTEA,
    finalized_at_l1_block BIGINT,
    PRIMARY KEY (l2_tx_hash, l2_log_index)
);
CREATE INDEX ON l2_withdrawals (l1_batch_number);
CREATE INDEX ON l2_withdrawals ("from");
CREATE INDEX ON l2_withdrawals (finalized_at_l1_block) WHERE finalized_at_l1_block IS NULL;
```

A withdrawal is claimable on L1 only after `l1_batches.executed_at_l1` is set for its `l1_batch_number`.

### Contract deployments via ContractDeployer

```sql
CREATE TABLE contract_deployments (
    block_number         BIGINT NOT NULL,
    tx_hash              BYTEA NOT NULL,
    log_index            INTEGER NOT NULL,
    deployer             BYTEA NOT NULL,             -- the account that called ContractDeployer
    deployed_address     BYTEA NOT NULL,
    bytecode_hash        BYTEA NOT NULL,             -- hash, not full bytecode
    PRIMARY KEY (block_number, log_index)
);
CREATE INDEX ON contract_deployments (deployer);
CREATE INDEX ON contract_deployments (deployed_address);
```

## ClickHouse sketch

For event volume:

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
    l1_batch_number    UInt64,
    finality           UInt8                       -- 0 sealed / 1 committed / 2 proven / 3 executed
)
ENGINE = ReplacingMergeTree(block_number)
PARTITION BY toMonday(timestamp)
ORDER BY (address, topic0, block_number, log_index)
SETTINGS index_granularity = 8192;
```

## Reorg handling

Era's L2 reorgs are rare and shallow. Standard cascade-delete via FK on `l2_blocks` is fine.

The more interesting state changes are L1-side: a batch can advance from `committed` to `proven` to `executed` over hours, and rarely L1 reorgs can revert these. The `l1_batches` table tracks this lifecycle; updates are idempotent.

## Choosing PG vs CH

| Workload | Pick |
|---|---|
| Withdrawal lifecycle, paymaster attribution, deployment history | **Postgres** |
| AA validation analytics, signer enumeration | **Postgres** |
| Token transfer history per address | Either |
| `eth_getLogs`-shaped queries at scale | **ClickHouse** |
| Cross-batch analytics (e.g. "user activity per batch") | **ClickHouse** |

## Sizing

Era's tx volume is meaningful but not extreme. Disk requirements similar to OP Mainnet for the core data; **paymaster activity and AA validation events** add a unique workload not seen on simpler L2s.

For multi-chain ZK Stack indexing (Era + other Hyperchains), partition by `chain_id` everywhere, same as OP Stack.
