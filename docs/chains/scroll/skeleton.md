# Skeleton

Illustrative Rust. Scroll is the closest of any non-ETH chain to "use the Ethereum skeleton unchanged" — the deltas are the L1-fee receipt fields and the L1 batch lifecycle.

## Crates

```toml
# illustrative — same as Ethereum skeleton
alloy = { version = "0.8", features = ["full"] }
serde = "1"
anyhow = "1"
tokio = { version = "1", features = ["full"] }
```

No first-party Rust SDK for Scroll; standard `alloy` works because Scroll uses standard ETH RPC + bytecode-equivalent EVM.

## Decoded data shape

```rust
pub struct ScrollReceipt {
    // Standard ETH receipt
    pub status: u8,
    pub cumulative_gas_used: u64,
    pub gas_used: u64,
    pub effective_gas_price: U256,
    pub contract_address: Option<Address>,
    pub logs: Vec<alloy::Log>,
    // L1 fee inputs
    pub l1_fee: U256,
    pub l1_gas_used: U256,
    pub l1_gas_price: U256,
    pub l1_blob_base_fee: Option<U256>,           // post-Bernoulli
    pub l1_fee_scalar: Option<U256>,              // pre-Bernoulli
    pub l1_base_fee_scalar: Option<u64>,          // post-Bernoulli
    pub l1_blob_base_fee_scalar: Option<u64>,     // post-Bernoulli
}
```

`alloy`'s default receipt type doesn't include the L1-fee fields; deserialize via `serde_json::Value` and extract the extras, or use a custom struct with `#[serde(rename_all = "camelCase")]`.

## Minimal handler

```rust
async fn run(ws_url: &str) -> anyhow::Result<()> {
    let provider = ProviderBuilder::new().on_ws(ws_url.parse()?).await?;
    let mut heads = provider.subscribe_blocks().await?.into_stream();

    while let Some(header) = heads.next().await {
        let block = provider
            .get_block_by_hash(header.hash, true.into())
            .await?
            .expect("block");

        for tx in block.transactions.txns() {
            let receipt: serde_json::Value = provider
                .raw_request("eth_getTransactionReceipt", (tx.hash,))
                .await?;
            let scroll_receipt = parse_scroll_receipt(&receipt)?;

            // Detect L1-aliased deposits
            let is_l1_originated = is_l1_aliased_address(tx.from);

            ingest_tx(&block, tx, &scroll_receipt, is_l1_originated).await?;
        }
    }
    Ok(())
}

fn is_l1_aliased_address(addr: Address) -> bool {
    // L1 alias offset = 0x1111000000000000000000000000000000001111
    // If addr's "alias mask" matches, it's L1-originated.
    let bytes = addr.as_slice();
    bytes[0] >= 0x11 && bytes[0] <= 0x21  // simplified; check actual offset semantics
}
```

## Tracking L1 batch lifecycle

```rust
const SCROLL_CHAIN: Address = address!("0x...");  // chain-specific

const TOPIC_COMMIT_BATCH:    B256 = /* keccak256("CommitBatch(uint256,bytes32)") */;
const TOPIC_FINALIZE_BATCH:  B256 = /* keccak256("FinalizeBatch(uint256,bytes32,bytes32,bytes32)") */;

async fn run_l1_batches(l1: &Provider) -> anyhow::Result<()> {
    let filter = alloy::rpc::types::Filter::new()
        .address(vec![SCROLL_CHAIN])
        .event_signature(vec![TOPIC_COMMIT_BATCH, TOPIC_FINALIZE_BATCH]);

    let mut sub = l1.subscribe_logs(&filter).await?.into_stream();
    while let Some(log) = sub.next().await {
        let batch_index = u64::from_be_bytes(log.topics[1][24..].try_into()?);
        if log.topics[0] == TOPIC_COMMIT_BATCH {
            db.update_batch_committed(batch_index, log.block_number, log.transaction_hash).await?;
            db.update_blocks_finality(batch_index, BatchFinality::Committed).await?;
        } else {
            db.update_batch_finalized(batch_index, log.block_number, log.transaction_hash).await?;
            db.update_blocks_finality(batch_index, BatchFinality::Finalized).await?;
        }
    }
    Ok(())
}
```

Note: this requires knowing the L2 block range per batch. Either:
- Decode the batch's calldata / blob to extract the L2 block range, or
- Use Scroll's RPC if it exposes a `scroll_getBatchByIndex` method {{unsourced: confirm}}, or
- Maintain a running mapping from each L2 block to its committing batch by listening to commit events and pulling the batch's L2 range.

## L1 → L2 message tracking

```rust
async fn on_l1_message_queue(log: &Log) -> anyhow::Result<()> {
    // L1MessageQueue.QueueTransaction event
    let queue_index = decode_queue_index(&log.data)?;
    db.insert_l1_to_l2_message(L1MessageRow {
        queue_index,
        l1_block: log.block_number,
        l1_tx: log.transaction_hash,
        sender_l1: decode_sender(&log.data)?,
        target: decode_target(&log.data)?,
        value: decode_value(&log.data)?,
        data: decode_data(&log.data)?.to_vec(),
        status: MessageStatus::Queued,
        ..Default::default()
    }).await
}

// On L2 side: when a tx with L1-aliased `from` executes, mark the queue entry as executed
async fn on_l2_l1_originated_tx(tx: &alloy::Transaction, queue_index: u64) -> anyhow::Result<()> {
    db.update_l1_message_executed(queue_index, tx.block_number.unwrap(), tx.hash).await
}
```

Joining the L1 queue index to the L2 tx requires either decoding the L1 message hash on L2 (the message hash is deterministic from the queue index) or using a `RelayedMessage` event on the L2 messenger.

## What's not shown

- **L1-fee formula recomputation** — same shape as OP Stack's version-by-block; pin Bernoulli/Curie/Darwin activation timestamps.
- **Reorg walk-back** — same as ETH; small in practice on Scroll.
- **Trace ingestion** — standard `debug_traceBlockByNumber`; works as on Ethereum.
- **Withdrawal proving flow** — Merkle proof construction against the L2 messenger's withdrawal trie; for users who need to claim on L1.
