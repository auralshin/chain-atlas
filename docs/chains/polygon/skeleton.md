# Skeleton

Illustrative Rust. Polygon's distinctive needs: a heimdall client alongside the bor client, and explicit state-sync ingestion via heimdall (not bor).

## Crates

```toml
# illustrative
alloy = { version = "0.8", features = ["full"] }
serde = "1"
anyhow = "1"
tokio = { version = "1", features = ["full"] }
reqwest = { version = "0.12", features = ["json"] }   # for heimdall REST
# No first-party Rust SDK for heimdall; use raw HTTP.
```

## Three streams

A complete Polygon indexer pulls from three sources:

```rust
pub struct Streams {
    pub bor: alloy::providers::RootProvider<...>,    // bor JSON-RPC
    pub heimdall: HeimdallClient,                    // heimdall Cosmos REST
    pub l1: alloy::providers::RootProvider<...>,     // Ethereum mainnet
}
```

## Bor block stream

```rust
async fn run_bor(streams: &Streams) -> anyhow::Result<()> {
    let mut heads = streams.bor.subscribe_blocks().await?.into_stream();
    while let Some(header) = heads.next().await {
        let block = streams.bor
            .get_block_by_hash(header.hash, true.into())
            .await?
            .expect("block");
        ingest_bor_block(&block).await?;
    }
    Ok(())
}
```

Standard ETH-shape stream. The reorg walk-back is the same shape as for ETH but with **deeper buffer** — see [reorgs-finality.md](reorgs-finality.md). Provision the walk-back to 512+ blocks.

## Heimdall checkpoint poller

```rust
pub struct HeimdallCheckpoint {
    pub id: u64,
    pub start_block: u64,
    pub end_block: u64,
    pub root_hash: alloy::primitives::B256,
    pub proposer: alloy::primitives::Address,
}

impl HeimdallClient {
    pub async fn checkpoints_since(&self, last_id: u64)
        -> anyhow::Result<Vec<HeimdallCheckpoint>>
    {
        let resp: serde_json::Value = self.client
            .get(format!("{}/checkpoints/list?from-id={}", self.base, last_id + 1))
            .send().await?.json().await?;
        // Parse Cosmos REST response shape into HeimdallCheckpoint
        parse_checkpoints(resp)
    }
}

async fn run_checkpoints(streams: &Streams) -> anyhow::Result<()> {
    let mut last = db.last_checkpoint_id().await?;
    loop {
        let new = streams.heimdall.checkpoints_since(last).await?;
        for cp in new {
            db.upsert_checkpoint(&cp).await?;
            db.mark_blocks_checkpointed(cp.start_block, cp.end_block, cp.id).await?;
            last = cp.id;
        }
        tokio::time::sleep(std::time::Duration::from_secs(30)).await;
    }
}
```

## Heimdall state-sync poller

```rust
pub struct StateSyncRecord {
    pub id: u64,
    pub contract: alloy::primitives::Address,
    pub data: Vec<u8>,
    pub tx_hash_l1: alloy::primitives::B256,
    pub log_index: u64,
}

async fn run_state_syncs(streams: &Streams) -> anyhow::Result<()> {
    let mut last = db.last_state_sync_id().await?;
    loop {
        let new: Vec<StateSyncRecord> = streams.heimdall
            .get(format!("/clerk/event-record/list?from-id={}", last + 1))
            .await?;
        for rec in new {
            db.upsert_state_sync(&rec).await?;
            last = rec.id;
        }
        tokio::time::sleep(std::time::Duration::from_secs(30)).await;
    }
}
```

This is the **canonical state-sync ingestion path**. Do not try to scrape state-syncs from bor logs — heimdall is the source of truth.

## L1 stream: state-sync origination + checkpoint anchoring

```rust
async fn run_l1(streams: &Streams) -> anyhow::Result<()> {
    let filter = alloy::rpc::types::Filter::new()
        .address(vec![STATE_SENDER, ROOT_CHAIN, STAKE_MANAGER]);

    let mut sub = streams.l1.subscribe_logs(&filter).await?.into_stream();
    while let Some(log) = sub.next().await {
        match (log.address, log.topics.first()) {
            (STATE_SENDER, Some(t)) if *t == STATE_SYNCED => {
                let (state_id, contract, data) = decode_state_synced(&log.data)?;
                db.update_state_sync_l1(state_id, log.block_number, log.transaction_hash).await?;
            }
            (ROOT_CHAIN, Some(t)) if *t == NEW_HEADER_BLOCK => {
                let cp = decode_new_header_block(&log.data)?;
                db.update_checkpoint_l1(cp.id, log.block_number, log.transaction_hash).await?;
            }
            (STAKE_MANAGER, _) => {
                // Update validator state
            }
            _ => {}
        }
    }
    Ok(())
}
```

L1 events are **complementary** to heimdall's view: heimdall tells you the conceptual state, L1 events tell you when it was anchored to Ethereum.

## Three-stream consistency

The three streams arrive independently and at different cadences. Recommended:

```rust
#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let streams = Streams::connect(&cfg).await?;
    let _ = tokio::join!(
        run_bor(&streams),
        run_checkpoints(&streams),
        run_state_syncs(&streams),
        run_l1(&streams),
    );
    Ok(())
}
```

State writes are idempotent (upserts on `state_id`, `checkpoint_id`, etc.) so the streams can interleave freely. Reads that need consistency (e.g. "is block N final") join across tables at query time.

## What's not shown

- Reorg walk-back on bor (deep buffer; same shape as ETH, larger constant).
- Validator-set indexing (heimdall `/staking/validators` poll).
- Sprint / span tracking (heimdall `/bor/spans/list`).
- Withdrawal proving flow (computing Merkle proofs against checkpointed bor blocks).
- Plasma bridge (legacy; only relevant for very specific historical assets).
