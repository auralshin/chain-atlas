# Skeleton

The Optimism skeleton in [optimism/skeleton.md](../optimism/skeleton.md) works for Base **unchanged**, with one substitution: the rollup config + L1 contract addresses are Base's.

## Configuration delta

```rust
// Multi-chain skeleton — same code, different config
let config = match chain {
    Chain::OpMainnet => RollupConfig::op_mainnet(),
    Chain::Base       => RollupConfig::base_mainnet(),
    // …
};

let provider = ProviderBuilder::new()
    .network::<Optimism>()                   // op-alloy network type works for ALL OP Stack chains
    .on_ws(config.l2_ws_url.parse()?)
    .await?;

let op_node = OpNodeClient::new(&config.op_node_url);
let l1 = ProviderBuilder::new().on_http(config.l1_http_url.parse()?);
```

The `Optimism` network type from `op-alloy` is **chain-agnostic**: it tells `alloy` to deserialize OP-format blocks/receipts. It does not encode a specific chain's identity.

## Multi-chain indexer pattern

A single binary can index OP Mainnet, Base, and any other OP Stack chain in parallel by spinning up one task per chain with a shared schema:

```rust
async fn run_chain(chain: Chain, db: Pg) -> anyhow::Result<()> {
    let cfg = chain.rollup_config();
    let provider = build_provider(&cfg).await?;
    let op_node = OpNodeClient::new(&cfg.op_node_url);

    let mut heads = provider.subscribe_blocks().await?.into_stream();
    while let Some(h) = heads.next().await {
        ingest_block(&db, chain, &provider, h.hash).await?;
        // Periodically:
        let ss = op_node.sync_status().await?;
        update_finality(&db, chain, &ss).await?;
    }
    Ok(())
}

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let db = Pg::connect(env("DATABASE_URL")).await?;
    let _ = tokio::join!(
        run_chain(Chain::OpMainnet, db.clone()),
        run_chain(Chain::Base, db.clone()),
    );
    Ok(())
}
```

Cross-chain consistency: store `chain_id` on every row (see [storage.md](storage.md)). Reorg handling is per-chain; each task owns its head state.

## What's identical to Optimism

- Deposit-tx decoding.
- L1-fee formula version selection (Bedrock / Ecotone / Fjord) — but use Base's fork timestamps from the rollup config, not OP Mainnet's.
- `optimism_syncStatus` polling for the three heads.
- Bridge state machine.
- Reorg walk-back.

## What's different

- **L1 contract addresses** to monitor for deposits/output-roots/withdrawals.
- **Fork activation timestamps** (see [forks-changelog.md](forks-changelog.md)).

Everything else: read the Optimism skeleton.
