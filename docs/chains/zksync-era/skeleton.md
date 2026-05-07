# Skeleton

Illustrative Rust. Era's distinctive needs: handle type 0x71 / 0xFF txs, ingest paymaster + AA fields via `zks_*`, and track batch lifecycle on L1.

## Crates

```toml
# illustrative
alloy = { version = "0.8", features = ["full"] }
serde = "1"
serde_json = "1"
anyhow = "1"
tokio = { version = "1", features = ["full"] }
# zksync-types: the canonical Rust types from Matter Labs.
zksync-types = "26"  # {{unsourced: confirm latest version}}
```

`zksync-types` provides `Transaction` (with EIP-712 fields), `L1BatchDetails`, `BlockDetails`, etc. Without it, hand-rolling the type-0x71 deserializer is bug-prone.

## Decoded data shape

```rust
pub struct EraBlock {
    pub header: alloy::Header,
    pub l1_batch_number: u64,
    pub transactions: Vec<EraTx>,
}

pub struct EraTx {
    pub hash: B256,
    pub kind: EraTxKind,
    pub from: Address,             // account contract
    pub to: Option<Address>,
    pub value: U256,
    pub gas: u64,
    pub nonce: u64,
    pub paymaster: Option<Address>,
    pub paymaster_input: Bytes,
    pub factory_deps: Vec<B256>,
    pub custom_signature: Bytes,
    pub priority_request_id: Option<B256>, // for priority op (0xFF)
    pub l1_from: Option<Address>,           // for priority op
}

pub enum EraTxKind {
    Legacy,
    Eip2930,
    Eip1559,
    Eip712,             // 0x71
    PriorityOp,         // 0xFF
}
```

## Minimal handler: stream + ingest with `zks_*` enrichment

```rust
async fn run(ws_url: &str, http_url: &str) -> anyhow::Result<()> {
    let provider = ProviderBuilder::new().on_ws(ws_url.parse()?).await?;
    let zks = ProviderBuilder::new().on_http(http_url.parse()?);  // for zks_* methods

    let mut heads = provider.subscribe_blocks().await?.into_stream();
    while let Some(header) = heads.next().await {
        // Standard ETH-shape block
        let block = provider
            .get_block_by_hash(header.hash, true.into())
            .await?
            .expect("block");

        // zks-specific batch details
        let block_details: serde_json::Value = zks
            .raw_request("zks_getBlockDetails", (header.number,))
            .await?;
        let l1_batch_number = block_details["l1BatchNumber"].as_u64().unwrap_or(0);

        for tx in block.transactions.txns() {
            // For each tx, fetch zks-enriched details to get paymaster info
            let details: serde_json::Value = zks
                .raw_request("zks_getTransactionDetails", (tx.hash,))
                .await?;
            let era_tx = parse_era_tx(tx, &details)?;
            ingest_tx(&era_tx, l1_batch_number).await?;
        }
    }
    Ok(())
}
```

Note: fetching `zks_getTransactionDetails` per tx is expensive at scale. For high-throughput ingestion, batch via `zks_*` plural variants if available, or accept that initial ingestion uses standard `eth_getBlockReceipts` and a follow-up sweep enriches paymaster info.

## Tracking batch lifecycle on L1

Subscribe to L1 events from the `ExecutorFacet`:

```rust
const EXECUTOR_FACET: Address = address!("0x...");  // chain-specific

const TOPIC_BLOCK_COMMIT:        B256 = /* keccak256("BlockCommit(uint256,bytes32,bytes32)") */ ;
const TOPIC_BLOCKS_VERIFICATION: B256 = /* keccak256("BlocksVerification(uint256,uint256)") */;
const TOPIC_BLOCK_EXECUTION:     B256 = /* keccak256("BlockExecution(uint256,bytes32,bytes32)") */;

async fn run_l1_batches(l1: &Provider) -> anyhow::Result<()> {
    let filter = alloy::rpc::types::Filter::new()
        .address(vec![EXECUTOR_FACET])
        .event_signature(vec![
            TOPIC_BLOCK_COMMIT,
            TOPIC_BLOCKS_VERIFICATION,
            TOPIC_BLOCK_EXECUTION,
        ]);

    let mut sub = l1.subscribe_logs(&filter).await?.into_stream();
    while let Some(log) = sub.next().await {
        match log.topics[0] {
            t if t == TOPIC_BLOCK_COMMIT => {
                let batch = u64::from_be_bytes(log.topics[1][24..].try_into()?);
                db.update_batch_committed(batch, log.block_number, log.transaction_hash).await?;
                db.update_blocks_status(batch, BatchStatus::Committed).await?;
            }
            t if t == TOPIC_BLOCKS_VERIFICATION => {
                let (from, to) = decode_verification_range(&log.data)?;
                for batch in from..=to {
                    db.update_batch_proven(batch, log.block_number, log.transaction_hash).await?;
                    db.update_blocks_status(batch, BatchStatus::Proven).await?;
                }
            }
            t if t == TOPIC_BLOCK_EXECUTION => {
                let batch = u64::from_be_bytes(log.topics[1][24..].try_into()?);
                db.update_batch_executed(batch, log.block_number, log.transaction_hash).await?;
                db.update_blocks_status(batch, BatchStatus::Executed).await?;
            }
            _ => {}
        }
    }
    Ok(())
}
```

This is the canonical way to advance the three finality stages.

## Withdrawals: linking L2 burn to L1 finalization

```rust
async fn on_l1_messenger_event(log: &Log, block: u64) -> anyhow::Result<()> {
    let withdrawal = parse_l2_to_l1_message(&log.data)?;
    db.insert_withdrawal(WithdrawalRow {
        l2_tx_hash: log.transaction_hash,
        l2_log_index: log.log_index,
        l2_block_number: block,
        l1_batch_number: get_l1_batch_for_block(block).await?,
        from: withdrawal.sender,
        to: withdrawal.receiver,
        value: withdrawal.value,
        ..Default::default()
    }).await
}

// Later, when the user calls L1Bridge.finalizeWithdrawal:
async fn on_l1_finalize(log: &L1FinalizeLog) -> anyhow::Result<()> {
    db.update_withdrawal_finalized(
        log.l2_tx_hash, log.l2_log_index,
        log.l1_block, log.l1_tx
    ).await
}
```

## What's not shown

- **Paymaster-specific decoding** — depends on the paymaster contract; extract its events at indexing time.
- **AA signer recovery** — non-trivial; depends on the account contract; capture signer address from the validation context if you care.
- **EraVM trace decoding** — `debug_traceTransaction` produces frames including system contracts; filter by `from != Bootloader` for user-facing call trees.
- **Reorg walk-back** — same shape as ETH but small in practice (Era sequencer rewinds are rare and shallow).
- **Bridgehub-routed deposits** for non-Era ZK Stack chains — query Bridgehub for the deposit's actual destination chain.
