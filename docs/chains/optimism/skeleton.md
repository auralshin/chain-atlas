# Skeleton

Illustrative Rust. Not a runnable indexer. Shows the shape of an OP Stack subscriber: connect, stream heads, decode the OP-specific bits, sketch what an indexer would persist.

The Ethereum equivalent is in [`../ethereum/skeleton.md`](../ethereum/skeleton.md). This file documents the **deltas**.

## Crates

```toml
# Cargo.toml dependencies, illustrative
alloy = { version = "0.8", features = ["full", "rpc-types-eth"] }
op-alloy = "0.8"                  # OP-specific types: deposit tx, OP receipt
serde = "1"
anyhow = "1"
tokio = { version = "1", features = ["full"] }
```

`op-alloy` provides the types `OpTransactionType`, `OpTxEnvelope`, `OpTransactionReceipt` etc. that include the deposit variant and the L1-fee fields. **Do not** roll your own OP receipt struct — every fork shifts the field set and `op-alloy` tracks it.

## Decoded data shape

```rust
// Illustrative — actual fields tracked by op-alloy
pub struct OpBlock {
    pub header: alloy::Header,
    pub transactions: Vec<OpTxEnvelope>,
}

pub enum OpTxEnvelope {
    Legacy(alloy::TxLegacy),
    Eip2930(alloy::TxEip2930),
    Eip1559(alloy::TxEip1559),
    Eip7702(alloy::TxEip7702),       // post-Isthmus
    Deposit(OpDepositTx),            // type 0x7E
}

pub struct OpDepositTx {
    pub source_hash: B256,
    pub from: Address,               // L1-aliased for contracts
    pub to: Option<Address>,
    pub mint: U256,
    pub value: U256,
    pub gas_limit: u64,
    pub is_system_tx: bool,
    pub input: Bytes,
}
```

## Minimal handler: stream blocks + decode deposits

```rust
use alloy::providers::{Provider, ProviderBuilder};
use alloy::pubsub::PubSubFrontend;
use op_alloy::network::Optimism;

async fn run(ws_url: &str) -> anyhow::Result<()> {
    let provider = ProviderBuilder::new()
        .network::<Optimism>()
        .on_ws(ws_url.parse()?)
        .await?;

    let mut heads = provider.subscribe_blocks().await?.into_stream();
    while let Some(header) = heads.next().await {
        let block = provider
            .get_block_by_hash(header.hash, true.into())
            .await?
            .expect("block");

        for tx in block.transactions.txns() {
            match tx {
                OpTxEnvelope::Deposit(d) => handle_deposit(&block, d).await?,
                _ => handle_user_tx(&block, tx).await?,
            }
        }
    }
    Ok(())
}
```

The `Optimism` network type tells `alloy` to use OP-specific block + receipt deserializers. Without it, deposit txs deserialize as malformed legacy txs.

## Tracking the three heads via `op-node`

```rust
#[derive(serde::Deserialize)]
struct SyncStatus {
    unsafe_l2: HeadRef,
    safe_l2: HeadRef,
    finalized_l2: HeadRef,
    current_l1: HeadRef,
    finalized_l1: HeadRef,
}
#[derive(serde::Deserialize)]
struct HeadRef { hash: B256, number: u64 }

async fn poll_finality(rpc: &alloy::rpc::Client) -> anyhow::Result<SyncStatus> {
    let v: serde_json::Value = rpc.request("optimism_syncStatus", ()).await?;
    Ok(serde_json::from_value(v)?)
}
```

Poll every 2–6 seconds. Compare the returned `safe_l2.hash` against your storage's safe head; if they disagree, walk back to find the common ancestor and re-ingest from there.

## Fork-versioned L1-fee decode

```rust
fn formula_version(timestamp: u64, cfg: &RollupConfig) -> FormulaVersion {
    if timestamp >= cfg.fjord_time     { return FormulaVersion::Fjord; }
    if timestamp >= cfg.ecotone_time   { return FormulaVersion::Ecotone; }
    /* Canyon does not change the L1-fee formula */
    FormulaVersion::Bedrock
}

fn l1_fee(receipt: &OpReceipt, v: FormulaVersion) -> U256 {
    match v {
        FormulaVersion::Bedrock => {
            receipt.l1_gas_used * receipt.l1_gas_price
                * receipt.l1_fee_scalar / U256::from(1_000_000)
        }
        FormulaVersion::Ecotone => {
            let bf = U256::from(receipt.l1_base_fee_scalar) * 16
                * receipt.l1_gas_price;
            let blob = U256::from(receipt.l1_blob_base_fee_scalar)
                * receipt.l1_blob_base_fee;
            receipt.l1_gas_used * (bf + blob) / U256::from(16_000_000)
        }
        FormulaVersion::Fjord => { /* FastLZ size estimation */ todo!() }
    }
}
```

This is the kind of code that gets it wrong silently when a fork lands. Pin every fork timestamp from `optimism_rollupConfig` at startup and assert on mismatch with hardcoded fallbacks.

## What an indexer would persist

For each block + tx + receipt:

```rust
async fn persist_tx(db: &Pg, b: &OpBlock, tx: &OpTxEnvelope, r: &OpReceipt)
    -> anyhow::Result<()>
{
    let row = match tx {
        OpTxEnvelope::Deposit(d) => TxRow {
            hash: tx.hash(),
            ty: 0x7E,
            from: d.from,
            l1_from: Some(unalias(d.from)),
            source_hash: Some(d.source_hash),
            mint: Some(d.mint),
            is_system_tx: d.is_system_tx,
            ..TxRow::default()
        },
        _ => TxRow::from_user_tx(tx),
    };
    db.insert_tx(b.header.number, &row).await?;

    db.insert_receipt(b.header.number, ReceiptRow {
        l1_fee: r.l1_fee,
        l1_gas_used: r.l1_gas_used,
        l1_gas_price: r.l1_gas_price,
        l1_base_fee_scalar: r.l1_base_fee_scalar,
        l1_blob_base_fee_scalar: r.l1_blob_base_fee_scalar,
        l1_blob_base_fee: r.l1_blob_base_fee,
        formula_version: formula_version(b.header.timestamp, &cfg) as i16,
        ..r.into()
    }).await?;
    Ok(())
}
```

The indexer's storage layer (PG schema in [storage.md](storage.md)) takes `formula_version` so future re-derivations of the L1 fee can honor whatever was active at write time.

## Not shown

- Reorg walk-back (depth-bounded; share with the ETH skeleton).
- Bridge state-machine transitions (read both L1 and L2 streams; correlate by `source_hash` for deposits and `withdrawal_hash` for withdrawals).
- Trace ingestion (`debug_traceBlockByNumber`).
- Backfill from a snapshot vs streaming.

These follow the same patterns as the [Ethereum skeleton](../ethereum/skeleton.md) and don't have OP-specific shape changes.
