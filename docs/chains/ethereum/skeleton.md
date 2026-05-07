# Indexer skeleton

Illustrative Rust using [alloy](https://github.com/alloy-rs/alloy) (1.7+ as of 2026). Trims setup, error handling, and dependency wiring. Cross-link to [storage.md](storage.md) and [reorgs-finality.md](reorgs-finality.md).

## Provider setup

```rust
use alloy::providers::{Provider, ProviderBuilder, WsConnect};

let provider = ProviderBuilder::new()
    .on_ws(WsConnect::new("ws://localhost:8546"))
    .await?;
```

For backfill, prefer HTTP and parallelize. For tip-following, use WS to subscribe.

## Streaming new heads

```rust
use futures::StreamExt;

let sub = provider.subscribe_blocks().await?;
let mut stream = sub.into_stream();
while let Some(header) = stream.next().await {
    handle_block(&provider, header.number).await?;
}
```

## Decoding the block

Pull the full block (with transactions) and the receipts in parallel:

```rust
use alloy::eips::BlockNumberOrTag;

let (block, receipts) = tokio::try_join!(
    provider.get_block_by_number(BlockNumberOrTag::Number(n), true),
    provider.get_block_receipts(BlockNumberOrTag::Number(n)),
)?;

let block = block.expect("block exists");
let receipts = receipts.unwrap_or_default();
```

`block.transactions` is `BlockTransactions::Full(Vec<Transaction>)`. Each carries an envelope discriminant; the alloy `TxEnvelope` enum covers all five live tx types.

## Extracting tx-type-specific fields

```rust
use alloy::consensus::TxEnvelope;

match tx.into_inner() {
    TxEnvelope::Legacy(t)   => /* type 0x00 */,
    TxEnvelope::Eip2930(t)  => /* type 0x01 — t.access_list */,
    TxEnvelope::Eip1559(t)  => /* type 0x02 — t.max_fee_per_gas, t.max_priority_fee_per_gas */,
    TxEnvelope::Eip4844(t)  => /* type 0x03 — t.blob_versioned_hashes */,
    TxEnvelope::Eip7702(t)  => /* type 0x04 — t.authorization_list */,
}
```

[verify: alloy 1.7 enum names match exactly — `TxEnvelope::Eip4844` vs `TxEnvelope::Eip4844WithSidecar` etc.]

## Writing to Postgres

```rust
sqlx::query!(
    "INSERT INTO blocks (number, hash, parent_hash, timestamp, miner,
                         gas_used, gas_limit, base_fee, blob_gas_used,
                         excess_blob_gas, parent_beacon_root, withdrawals_root)
     VALUES ($1, $2, $3, to_timestamp($4), $5, $6, $7, $8, $9, $10, $11, $12)",
    block.header.number as i64,
    block.header.hash.as_slice(),
    block.header.parent_hash.as_slice(),
    block.header.timestamp as f64,
    block.header.miner.as_slice(),
    block.header.gas_used as i64,
    block.header.gas_limit as i64,
    block.header.base_fee_per_gas.map(|v| v as i64),  // u64 fits BIGINT
    block.header.blob_gas_used.map(|v| v as i64),
    block.header.excess_blob_gas.map(|v| v as i64),
    block.header.parent_beacon_block_root.as_ref().map(|h| h.as_slice()),
    block.header.withdrawals_root.as_ref().map(|h| h.as_slice()),
)
.execute(&pg)
.await?;
```

For 256-bit integers (`value`, `effective_gas_price`), use `NUMERIC(78)` in PG and convert from `alloy::primitives::U256`:

```rust
fn u256_to_decimal(v: U256) -> rust_decimal::Decimal {
    // alloy U256 → string → decimal; or use a u256-numeric adapter crate
    v.to_string().parse().expect("valid u256")
}
```

[verify: pick a stable adapter — sqlx doesn't natively map U256. Some indexers store as `BYTEA(32)` and decode at read time.]

## Reorg detection

```rust
let stored_parent = pg_get_block_hash(block.header.number - 1).await?;
if stored_parent != Some(block.header.parent_hash) {
    rollback_to_common_ancestor(&provider, &pg, block.header.parent_hash).await?;
}
insert_block(&pg, &block).await?;
```

Full ancestor-walking loop: [reorgs-finality.md#recovery-pattern](reorgs-finality.md#recovery-pattern).

## Beacon API for blobs and finality

Run a CL alongside. Pull blob bodies via Beacon API:

```rust
#[derive(Deserialize)]
struct BlobSidecar {
    index: String,
    blob: String,            // 0x-prefixed hex, ~125 KB
    kzg_commitment: String,
    kzg_proof: String,
    // ...
}

let sidecars: Vec<BlobSidecar> = beacon_client
    .get(&format!("{}/eth/v1/beacon/blob_sidecars/{}", beacon_url, slot))
    .send().await?
    .json::<BeaconResponse<Vec<BlobSidecar>>>().await?
    .data;
```

Track finality:

```rust
let cp: FinalityCheckpoints = beacon_client
    .get(&format!("{}/eth/v2/beacon/states/head/finality_checkpoints", beacon_url))
    .send().await?
    .json::<BeaconResponse<FinalityCheckpoints>>().await?
    .data;

let finalized_slot = cp.finalized.epoch * 32;
```

[verify: BeaconResponse + FinalityCheckpoints shape against current Beacon API spec or the `ethereum-consensus` Rust crate]

## Notes

- **alloy is canonical.** ethers-rs is in maintenance mode and should not be used for new work as of 2026.
- For high throughput, look at **reth ExEx** (in-process, no JSON serialization) or **Substreams** (out-of-band streaming) — JSON-RPC has fundamental latency and parallelism limits.
- Backfilling 23M+ blocks at default settings is multi-day. For cold starts, use Erigon snapshots or geth `dump` exports, then go live via RPC.
