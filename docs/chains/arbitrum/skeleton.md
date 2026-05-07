# Skeleton

Illustrative Rust. Same shape as the [Ethereum](../ethereum/skeleton.md) and [Optimism](../optimism/skeleton.md) skeletons; the deltas are Arbitrum's tx types, receipt fields, and the retryable lifecycle.

## Crates

```toml
# illustrative
alloy = { version = "0.8", features = ["full", "rpc-types-eth"] }
serde = "1"
anyhow = "1"
tokio = { version = "1", features = ["full"] }
# No "alloy-arbitrum" exists as a first-party crate as of writing.
# Most Rust indexers either reuse alloy primitives or maintain a small
# local module for Arbitrum tx-type decoding.
```

There is no canonical `op-alloy`-equivalent for Arbitrum in the alloy ecosystem at the time of writing {{unsourced: confirm}}. Rust indexers commonly:
- Use `alloy` for everything but tx envelope decoding.
- Roll a small local enum for Arbitrum-system tx types (`0x64`–`0x6A`).
- Read Arbitrum-specific receipt fields by deserializing the JSON-RPC response with extra `Option<_>` fields.

## Decoded data shape

```rust
pub struct ArbBlock {
    pub header: alloy::Header,
    pub transactions: Vec<ArbTxEnvelope>,
    pub l1_block_number: u64,
    pub send_root: B256,
    pub send_count: u64,
}

pub enum ArbTxEnvelope {
    Standard(alloy::TxEnvelope),                // type 0x00 / 0x01 / 0x02 / 0x04
    Deposit(ArbDepositTx),                      // 0x64
    Unsigned(ArbUnsignedTx),                    // 0x65
    Contract(ArbContractTx),                    // 0x66
    Retry(ArbRetryTx),                          // 0x68
    SubmitRetryable(ArbSubmitRetryableTx),      // 0x69
    Internal(ArbInternalTx),                    // 0x6A
}

pub struct ArbReceipt {
    pub cumulative_gas: u64,
    pub gas_used: u64,
    pub gas_used_for_l1: u64,
    pub effective_gas_price: U256,
    pub status: u8,
    pub contract_address: Option<Address>,
    pub l1_block_number: u64,
    pub logs: Vec<alloy::Log>,
}
```

## Minimal handler: stream blocks + decode Arbitrum-specific txs

```rust
use alloy::providers::{Provider, ProviderBuilder};

async fn run(ws_url: &str) -> anyhow::Result<()> {
    let provider = ProviderBuilder::new().on_ws(ws_url.parse()?).await?;
    let mut heads = provider.subscribe_blocks().await?.into_stream();

    while let Some(header) = heads.next().await {
        let block: serde_json::Value = provider
            .raw_request("eth_getBlockByHash", (header.hash, true))
            .await?;
        let arb_block = parse_arb_block(&block)?;

        for tx in &arb_block.transactions {
            match tx {
                ArbTxEnvelope::SubmitRetryable(t) => on_retryable_create(t).await?,
                ArbTxEnvelope::Retry(t) => on_retryable_redeem(t).await?,
                ArbTxEnvelope::Deposit(t) => on_eth_deposit(t).await?,
                ArbTxEnvelope::Internal(_) => { /* ArbOS internal — usually ignore */ }
                _ => on_user_tx(tx).await?,
            }
        }
    }
    Ok(())
}
```

`raw_request` is used because the standard `alloy` block deserializer doesn't know about Arbitrum's extra fields. The custom `parse_arb_block` reads them off the JSON.

## Tracking retryable lifecycle

```rust
async fn on_retryable_create(t: &ArbSubmitRetryableTx) -> anyhow::Result<()> {
    db.insert_retryable(RetryableRow {
        ticket_id: t.ticket_id,
        submit_l2_tx: t.tx_hash,
        l1_from: t.l1_origin_unaliased,
        to: t.to,
        callvalue: t.callvalue,
        state: TicketState::Pending,
        expires_at: t.submit_timestamp + 7 * 86400,
        ..Default::default()
    }).await
}

async fn on_retryable_redeem(t: &ArbRetryTx) -> anyhow::Result<()> {
    let new_state = if t.is_auto_redeem {
        TicketState::RedeemedAuto
    } else {
        TicketState::RedeemedManual
    };
    db.update_retryable_state(t.ticket_id, new_state, t.tx_hash).await
}

// Periodic sweep for expired tickets
async fn sweep_expired(now: u64) -> anyhow::Result<()> {
    db.execute(
        "UPDATE retryable_tickets
         SET state = $1
         WHERE state = $2 AND expires_at < $3",
        &[&(TicketState::Expired as i16), &(TicketState::Pending as i16), &(now as i64)]
    ).await
}
```

The expiry sweep is the only pattern in this whole skeleton that doesn't reduce to "react to chain events" — it's a wall-clock-driven state transition.

## Outbox lifecycle: L2 emission → L1 execution

```rust
// On L2 ArbSys.L2ToL1Tx events:
async fn on_outbox_create(ev: &L2ToL1TxEvent, block: &ArbBlock) -> anyhow::Result<()> {
    db.insert_outbox(OutboxRow {
        message_index: ev.position,
        l2_block_number: block.header.number,
        from: ev.caller,
        to: ev.destination,
        value: ev.value,
        data: ev.data.clone(),
        ..Default::default()
    }).await
}

// On L1 RollupProxy.NodeConfirmed (legacy) or EdgeChallengeManager.EdgeConfirmed (BoLD):
async fn on_assertion_confirmed(send_root: B256, l1_block: u64) -> anyhow::Result<()> {
    db.execute(
        "UPDATE outbox_messages
         SET confirmed_at_l1_block = $1
         WHERE l2_block_number IN (
             SELECT number FROM l2_blocks WHERE send_root = $2
         )",
        &[&(l1_block as i64), &send_root.as_bytes()]
    ).await
}

// On L1 Outbox.OutBoxTransactionExecuted:
async fn on_outbox_executed(idx: u64, l1_block: u64, l1_tx: B256) -> anyhow::Result<()> {
    db.update_outbox_executed(idx, l1_block, l1_tx).await
}
```

## What's not shown

- L1 stream (`SequencerInbox`, `Bridge`, `Outbox`, `RollupProxy` / `EdgeChallengeManager`) — patterns same as OP Stack: subscribe to L1 logs, route by emitter address.
- Reorg walk-back — same shape as OP Stack.
- Backfill from a snapshot — Nitro's `arbnode_getMessage` lets you pull individual L1 inbox messages for replay.
- BoLD vs legacy assertion routing during the migration window — read both contract sets.
- Trace ingestion via `arbtrace_block` — same shape as `debug_traceBlockByNumber` adapters but with the Parity-style flat-trace JSON.
