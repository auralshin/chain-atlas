# Skeleton

Illustrative Rust. StarkNet has its own JSON-RPC; `alloy` does not work. Use a StarkNet-aware client.

## Crates

```toml
# illustrative
starknet-rs = "0.13"           # Top-level Rust SDK; tracks the JSON-RPC spec.
starknet-providers = "0.13"    # Sub-crate for RPC clients
starknet-core = "0.13"         # Core types: Felt, FieldElement, etc.
serde = "1"
anyhow = "1"
tokio = { version = "1", features = ["full"] }
sqlx = { version = "0.8", features = ["postgres", "runtime-tokio"] }
```

`starknet-rs` is the canonical Rust SDK. It tracks the JSON-RPC spec versions and exposes typed clients. Use this rather than rolling your own.

## Decoded data shape

```rust
use starknet::core::types::{Felt, BlockId, BlockTag};
use starknet::providers::jsonrpc::{HttpTransport, JsonRpcClient};

pub struct StarknetBlock {
    pub block_number: u64,
    pub block_hash: Felt,
    pub parent_hash: Felt,
    pub timestamp: u64,
    pub starknet_version: String,
    pub status: BlockStatus,                       // PENDING / ACCEPTED_ON_L2 / ACCEPTED_ON_L1
    pub transactions: Vec<Transaction>,            // typed enum from starknet-core
    pub l1_gas_price: ResourcePrice,
    pub l1_data_gas_price: Option<ResourcePrice>,
}

pub struct ResourcePrice {
    pub price_in_wei: u128,
    pub price_in_strk: u128,                       // FRI, smallest unit of STRK
}
```

## Minimal handler

```rust
async fn run(http_url: &str) -> anyhow::Result<()> {
    let client = JsonRpcClient::new(HttpTransport::new(http_url.parse()?));
    let mut last_seen: u64 = db.last_block_number().await?;

    loop {
        let head = client.block_number().await?;
        for n in (last_seen + 1)..=head {
            let block = client.get_block_with_receipts(BlockId::Number(n)).await?;
            ingest_block(&block).await?;
            last_seen = n;
        }
        tokio::time::sleep(std::time::Duration::from_secs(2)).await;
    }
}
```

`get_block_with_receipts` returns the block + all txs + their receipts in one round-trip — much friendlier than the EVM `eth_getBlockByNumber + eth_getBlockReceipts` two-call pattern.

## Tx-type-aware persistence

```rust
use starknet::core::types::{Transaction, TransactionReceipt};

async fn ingest_tx(block_n: u64, tx: &Transaction, receipt: &TransactionReceipt)
    -> anyhow::Result<()>
{
    match tx {
        Transaction::Invoke(inv) => persist_invoke(block_n, inv, receipt).await,
        Transaction::Declare(decl) => persist_declare(block_n, decl, receipt).await,
        Transaction::DeployAccount(dep) => persist_deploy_account(block_n, dep, receipt).await,
        Transaction::L1Handler(h) => persist_l1_handler(block_n, h, receipt).await,
        Transaction::Deploy(_) => {
            // Deprecated; only historical txs match this
            Ok(())
        }
    }
}
```

Each variant has different fields. The schema in [storage.md](storage.md) stores them in a wide row with NULL for non-applicable columns.

## State diff ingestion

```rust
async fn ingest_state_update(client: &JsonRpcClient<HttpTransport>, block_n: u64)
    -> anyhow::Result<()>
{
    let update = client.get_state_update(BlockId::Number(block_n)).await?;

    for storage_diff in update.state_diff.storage_diffs {
        for entry in storage_diff.storage_entries {
            db.insert_storage_diff(block_n, storage_diff.address, entry.key, entry.value).await?;
        }
    }

    for declared in update.state_diff.declared_classes {
        // For Cairo 1: Sierra class. Fetch via starknet_getClass to get full code.
        let class = client.get_class(BlockId::Number(block_n), declared.class_hash).await?;
        db.insert_class(declared.class_hash, &class, block_n).await?;
    }

    for deployed in update.state_diff.deployed_contracts {
        db.insert_contract(deployed.address, deployed.class_hash, block_n).await?;
    }

    for replaced in update.state_diff.replaced_classes {
        db.insert_class_replacement(replaced.contract_address, replaced.class_hash, block_n).await?;
    }

    for nonce_update in update.state_diff.nonces {
        db.update_nonce(nonce_update.contract_address, nonce_update.nonce, block_n).await?;
    }

    Ok(())
}
```

This is the **canonical state ingestion pattern** for StarkNet. Walk through state diffs per block; everything you need is there.

## Event ingestion (paginated)

```rust
async fn run_events(client: &JsonRpcClient<HttpTransport>, contract: Felt) -> anyhow::Result<()> {
    let mut continuation_token: Option<String> = None;
    loop {
        let filter = EventFilter {
            from_block: Some(BlockId::Number(/* last seen */)),
            to_block: Some(BlockId::Tag(BlockTag::Latest)),
            address: Some(contract),
            keys: None,
        };
        let page = client.get_events(filter, continuation_token.clone(), 1000).await?;

        for event in page.events {
            db.insert_event(EventRow {
                block_number: event.block_number.unwrap(),
                tx_index: 0,                       // resolve via receipt cross-ref
                from_address: event.from_address,
                keys: event.keys,
                data: event.data,
                ..Default::default()
            }).await?;
        }

        match page.continuation_token {
            Some(t) => continuation_token = Some(t),
            None => break,
        }
    }
    Ok(())
}
```

`continuation_token`-based pagination is friendlier than EVM's block-range chunking.

## L1 ↔ L2 message tracking

L1 side (`Starknet Core Contract`): subscribe to `LogMessageToL2` and `ConsumedMessageToL2`.

```rust
const STARKNET_CORE: alloy::primitives::Address =
    address!("0xc662c410C0ECf747543f5bA90660f6ABeBD9C8c4");  // {{unsourced: confirm}}

async fn run_l1(l1: &alloy::providers::Provider) -> anyhow::Result<()> {
    let filter = alloy::rpc::types::Filter::new().address(vec![STARKNET_CORE]);
    let mut sub = l1.subscribe_logs(&filter).await?.into_stream();
    while let Some(log) = sub.next().await {
        // Match on event signature, decode, persist
        // ...
    }
    Ok(())
}
```

L2 side: when receipts include `messages_sent`, persist. Match L1 → L2 on the message hash.

## What's not shown

- **Reorg walk-back** — same shape as EVM; rare in practice on StarkNet.
- **Cairo 0 vs Cairo 1 class decoding** — branch on the class shape; the `starknet-rs` types differ.
- **Trace ingestion via `starknet_traceTransaction`** — useful for internal-tx-equivalent analytics; format is StarkNet-specific.
- **Fee unit conversion** (WEI vs FRI) — for "USD fee" totals, convert per tx using the block's prices.
- **Mempool / pending tx subscription** — for real-time UIs that want sub-finality data.
