# Skeleton

Illustrative Rust. BSC is the closest of any non-Ethereum EVM chain to "just use the Ethereum skeleton" — but two BSC-specific concerns matter: era-aware finality and validator-set parsing.

## Crates

```toml
# illustrative — same as Ethereum skeleton
alloy = { version = "0.8", features = ["full"] }
serde = "1"
anyhow = "1"
tokio = { version = "1", features = ["full"] }
```

No first-party Rust SDK for BSC; `alloy` works for everything because BSC is EVM-equivalent.

## Decoded data shape

Standard `alloy` types — no BSC-specific structs needed for transactions or receipts. The deltas are at the block level (validator info in `extraData`) and the optional finality / attestation layer.

```rust
pub struct BscBlock {
    pub header: alloy::Header,
    pub transactions: Vec<alloy::TxEnvelope>,
    pub validator_set: Option<ValidatorSetUpdate>,   // Some(_) only at epoch boundaries
}

pub struct ValidatorSetUpdate {
    pub addresses: Vec<Address>,
    pub bls_pubkeys: Vec<Vec<u8>>,                   // post-Plato only
}
```

## Minimal handler

```rust
use alloy::providers::{Provider, ProviderBuilder};

const PLATO_FORK_BLOCK: u64 = /* from chain config */ 0;

async fn run(ws_url: &str) -> anyhow::Result<()> {
    let provider = ProviderBuilder::new().on_ws(ws_url.parse()?).await?;
    let mut heads = provider.subscribe_blocks().await?.into_stream();

    while let Some(header) = heads.next().await {
        let block = provider
            .get_block_by_hash(header.hash, true.into())
            .await?
            .expect("block");

        ingest_block(&block).await?;

        // Validator set rotates at epoch boundary
        if header.number % 200 == 0 {
            let validators = parse_validator_set(&header.extra_data, header.number)?;
            db.upsert_epoch_validators(header.number / 200, &validators).await?;
        }
    }
    Ok(())
}
```

The validator-set parser branches on era:

```rust
fn parse_validator_set(extra: &[u8], block_number: u64) -> anyhow::Result<ValidatorSetUpdate> {
    if block_number >= PLATO_FORK_BLOCK {
        parse_post_plato_extra_data(extra)
    } else {
        parse_pre_plato_extra_data(extra)
    }
}
```

## Era-aware finality

```rust
async fn poll_finality(p: &Provider, head: u64) -> anyhow::Result<FinalityHeads> {
    if head >= PLATO_FORK_BLOCK {
        // Post-Plato: use block tags
        let safe = p.get_block_by_number("safe".into(), false.into()).await?;
        let finalized = p.get_block_by_number("finalized".into(), false.into()).await?;
        Ok(FinalityHeads {
            justified: safe.map(|b| b.header.number),
            finalized: finalized.map(|b| b.header.number),
        })
    } else {
        // Pre-Plato: depth-based heuristic
        Ok(FinalityHeads {
            justified: Some(head.saturating_sub(6)),    // soft conf
            finalized: Some(head.saturating_sub(15)),   // hard conf
        })
    }
}
```

## System contract event ingestion

```rust
const VALIDATOR_CONTRACT: Address = address!("0x0000000000000000000000000000000000001000");
const SLASH_CONTRACT:     Address = address!("0x0000000000000000000000000000000000001001");
const STAKE_HUB:          Address = address!("0x0000000000000000000000000000000000002000");

async fn run_system_events(p: &Provider) -> anyhow::Result<()> {
    let filter = alloy::rpc::types::Filter::new()
        .address(vec![VALIDATOR_CONTRACT, SLASH_CONTRACT, STAKE_HUB]);

    let mut sub = p.subscribe_logs(&filter).await?.into_stream();
    while let Some(log) = sub.next().await {
        match (log.address, log.topics.first()) {
            (SLASH_CONTRACT, _) => persist_slash_event(&log).await?,
            (STAKE_HUB, _) => persist_stake_event(&log).await?,
            (VALIDATOR_CONTRACT, _) => persist_validator_event(&log).await?,
            _ => {}
        }
    }
    Ok(())
}
```

## What's not shown

- Pre-Plato vs post-Plato `extraData` decoder — the canonical source is the `bsc` codebase's `parlia/parlia.go`.
- Trace ingestion (`debug_traceBlockByNumber`) — same pattern as Ethereum.
- Reorg walk-back — same pattern as Ethereum, with era-conditional depth.
- Cross-chain (pre-fusion) event ingestion — only relevant for backfill of historical bridge activity.
- BLS attestation verification — needed only if you're verifying finality independently rather than trusting the node's `finalized` tag.

## What can be reused from the Ethereum skeleton

Almost everything. The Ethereum skeleton's tx ingestion, log ingestion, receipt ingestion, reorg handling, and trace ingestion all transfer to BSC unchanged. The chain-specific code is just:

1. The era-aware finality switch above.
2. Validator-set parsing at epoch boundaries.
3. System contract event subscription.

If you have a working ETH indexer, adapting it to BSC is a 1-day job, not a 1-week job — by far the simplest non-ETH EVM chain to add.
