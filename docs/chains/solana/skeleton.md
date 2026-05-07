# Skeleton

Illustrative Rust. Solana indexer fundamentals: Geyser gRPC consumer for live, RPC for historical, and per-program instruction decoders.

## Crates

```toml
# illustrative
solana-sdk = "1.18"                            # Core types: Pubkey, Slot, Signature, etc.
solana-client = "1.18"                         # JSON-RPC client
solana-transaction-status = "1.18"             # EncodedTransactionWithStatusMeta etc.
yellowstone-grpc-proto = "1"                   # Geyser gRPC schema (Yellowstone)
yellowstone-grpc-client = "1"                  # Client for the same
spl-token = "4"                                # SPL Token decoders
spl-token-2022 = "1"                           # Token-2022 decoders
borsh = "1"                                    # For Borsh-deserialization of program data
serde = "1"
anyhow = "1"
tokio = { version = "1", features = ["full"] }
```

`solana-sdk` is the canonical Rust SDK. For typed program decoding, dApp-specific crates (often Anchor IDL-derived) cover the program you're indexing.

## Geyser stream

```rust
use yellowstone_grpc_client::{GeyserGrpcClient, GeyserGrpcClientResult};
use yellowstone_grpc_proto::prelude::*;

async fn run_geyser(endpoint: &str, x_token: Option<&str>) -> anyhow::Result<()> {
    let mut client = GeyserGrpcClient::build_from_shared(endpoint)?
        .x_token(x_token)?
        .connect().await?;

    let mut request = SubscribeRequest::default();

    // Filter: all confirmed slots
    request.slots.insert("all".to_string(), SubscribeRequestFilterSlots {
        filter_by_commitment: Some(true),
    });

    // Filter: all txs except votes
    request.transactions.insert("all".to_string(), SubscribeRequestFilterTransactions {
        vote: Some(false),                                  // exclude vote txs
        failed: Some(true),                                 // include failed txs
        signature: None,
        account_include: vec![],
        account_exclude: vec![],
        account_required: vec![],
    });

    let (mut subscribe_tx, mut stream) = client.subscribe().await?;
    subscribe_tx.send(request).await?;

    while let Some(msg) = stream.next().await {
        match msg? {
            SubscribeUpdate { update_oneof: Some(update_oneof::UpdateOneof::Transaction(tx)), .. } => {
                handle_tx(&tx).await?;
            }
            SubscribeUpdate { update_oneof: Some(update_oneof::UpdateOneof::Slot(slot)), .. } => {
                handle_slot(slot.slot, slot.status).await?;
            }
            _ => {}
        }
    }
    Ok(())
}
```

The Yellowstone gRPC schema is the de-facto standard. Most Geyser providers (Triton, Helius, etc.) speak it.

## Per-tx ingestion

```rust
async fn handle_tx(tx: &SubscribeUpdateTransaction) -> anyhow::Result<()> {
    let inner = &tx.transaction.unwrap();
    let meta = inner.meta.as_ref();

    let signature = bs58::encode(&inner.signature).into_string();
    let fee_payer = inner.transaction.as_ref().unwrap().message.as_ref().unwrap()
        .account_keys[0].clone();

    db.insert_tx(TxRow {
        slot: tx.slot,
        signature,
        fee_payer,
        fee_lamports: meta.map(|m| m.fee).unwrap_or(0),
        compute_units: meta.map(|m| m.compute_units_consumed.unwrap_or(0)).unwrap_or(0) as u32,
        err: meta.and_then(|m| m.err.as_ref().map(|e| format!("{:?}", e))),
        is_vote_tx: false,                                  // already filtered by Geyser
        version: detect_version(inner),
        ..Default::default()
    }).await?;

    // Walk top-level + inner instructions
    for (top_idx, ix) in inner.transaction.as_ref().unwrap().message.as_ref().unwrap()
        .instructions.iter().enumerate()
    {
        persist_instruction(tx.slot, vec![top_idx as u32], ix).await?;
    }

    if let Some(meta) = meta {
        for inner_set in &meta.inner_instructions {
            for (inner_idx, inner_ix) in inner_set.instructions.iter().enumerate() {
                let path = vec![inner_set.index as u32, inner_idx as u32];
                persist_instruction(tx.slot, path, inner_ix).await?;
            }
        }
    }

    // Token balance changes via pre/post diffs
    if let Some(meta) = meta {
        for (pre, post) in meta.pre_token_balances.iter().zip(meta.post_token_balances.iter()) {
            if pre.ui_token_amount != post.ui_token_amount {
                persist_token_balance_change(tx.slot, pre, post).await?;
            }
        }
    }

    Ok(())
}
```

## SPL Token transfer reconstruction

```rust
fn extract_token_transfers(
    tx: &EncodedTransactionWithStatusMeta,
) -> Vec<TokenTransfer> {
    let mut transfers = vec![];

    let meta = tx.meta.as_ref().expect("meta");
    let pre = &meta.pre_token_balances;
    let post = &meta.post_token_balances;

    // Per-account: detect mint + balance change
    for p in pre {
        let matching = post.iter().find(|q| q.account_index == p.account_index && q.mint == p.mint);
        if let Some(m) = matching {
            let delta = m.ui_token_amount.amount.parse::<i128>().unwrap_or(0)
                      - p.ui_token_amount.amount.parse::<i128>().unwrap_or(0);
            if delta != 0 {
                // Decoded balance change. Direction (from/to) requires CPI inspection.
                transfers.push(TokenTransfer {
                    account_index: p.account_index,
                    mint: p.mint.clone(),
                    delta,
                });
            }
        }
    }
    transfers
}
```

For exact "from -> to" attribution, walk inner instructions to find `Transfer` calls into SPL Token / Token-2022 and decode their args.

## Program-specific decoder example (Anchor)

```rust
// Suppose program X uses Anchor-style program logs.
async fn decode_program_x_logs(logs: &[String]) -> anyhow::Result<Vec<XEvent>> {
    let mut events = vec![];
    for log in logs {
        if let Some(b64) = log.strip_prefix("Program data: ") {
            let bytes = base64::decode(b64)?;
            // Anchor events: 8-byte discriminator + Borsh body
            let discriminator = &bytes[0..8];
            let body = &bytes[8..];

            match discriminator {
                X_SWAP_EVENT_DISCRIMINATOR => {
                    let event: SwapEvent = borsh::from_slice(body)?;
                    events.push(XEvent::Swap(event));
                }
                _ => {}
            }
        }
    }
    Ok(events)
}
```

`borsh::from_slice` is the canonical Anchor event decoding. The discriminator is the first 8 bytes of `sha256("event:EventName")`.

## ALT resolution

```rust
fn resolve_account_keys(tx: &VersionedTransaction, meta: &TransactionStatusMeta) -> Vec<Pubkey> {
    let mut keys = tx.message.static_account_keys().to_vec();
    if let Some(loaded) = &meta.loaded_addresses {
        keys.extend(&loaded.writable);
        keys.extend(&loaded.readonly);
    }
    keys
}
```

`meta.loaded_addresses` is populated by the validator at execution time. For backfill, this saves you from manually fetching ALT state.

## Historical stake account reconstruction

For reward attribution at a past epoch, `getAccountInfo` is wrong (see [gotchas.md](gotchas.md#historical-stake-state-cannot-be-queried--getaccountinfo-returns-current-state-only) for the three failure cases). Reconstruct by **backward-replay** through every Stake program instruction that touched the account.

```rust
use solana_sdk::pubkey::Pubkey;
use solana_client::nonblocking::rpc_client::RpcClient;
use solana_sdk::stake::instruction::StakeInstruction;

#[derive(Default, Debug, Clone)]
pub struct StakeStateView {
    pub staker:               Option<Pubkey>,
    pub withdrawer:           Option<Pubkey>,
    pub delegated_vote:       Option<Pubkey>,
    pub activation_epoch:     Option<u64>,
    pub deactivation_epoch:   Option<u64>,
    pub lamports:             u64,
    pub closed:               bool,
}

/// Replay all Stake program instructions touching `stake_account` up to
/// (but not including) `epoch_first_slot`. Returns the reconstructed state
/// at the target epoch boundary.
pub async fn reconstruct_stake_at_epoch(
    rpc:              &RpcClient,
    stake_account:    &Pubkey,
    epoch_first_slot: u64,
) -> anyhow::Result<StakeStateView> {
    // getSignaturesForAddress returns descending; reverse to ascending.
    let sigs = collect_all_signatures_for_address(rpc, stake_account).await?
        .into_iter()
        .filter(|s| s.slot < epoch_first_slot)
        .rev()
        .collect::<Vec<_>>();

    let mut state = StakeStateView::default();
    for sig in sigs {
        let tx = rpc.get_transaction(&sig.signature, /* config */).await?;
        for ix in iter_stake_instructions_touching(&tx, stake_account) {
            apply_stake_ix(&mut state, &ix, &tx)?;
        }
    }
    Ok(state)
}

fn apply_stake_ix(
    state: &mut StakeStateView,
    ix:    &StakeInstruction,
    tx:    &EncodedTransactionWithStatusMeta,
) -> anyhow::Result<()> {
    use solana_sdk::stake::state::StakeAuthorize;
    match ix {
        StakeInstruction::Initialize(authorized, _lockup) => {
            state.staker     = Some(authorized.staker);
            state.withdrawer = Some(authorized.withdrawer);
        }
        StakeInstruction::Authorize(new_authority, role) => match role {
            StakeAuthorize::Staker     => state.staker     = Some(*new_authority),
            StakeAuthorize::Withdrawer => state.withdrawer = Some(*new_authority),
        },
        StakeInstruction::DelegateStake => {
            // Vote account is the 2nd account in the ix's account list (per program ABI).
            // Resolve from the surrounding tx's account_keys.
            // state.delegated_vote   = Some(resolve_account(tx, ix, 1));
            // state.activation_epoch = Some(epoch_at_slot(tx.slot));
            // state.deactivation_epoch = None;
        }
        StakeInstruction::Deactivate => {
            // state.deactivation_epoch = Some(epoch_at_slot(tx.slot));
        }
        StakeInstruction::Withdraw(lamports) => {
            state.lamports = state.lamports.saturating_sub(*lamports);
            if state.lamports == 0 { state.closed = true; }
        }
        StakeInstruction::Split(_lamports) => {
            // Creates a sibling account; track in a separate per-account replay.
        }
        StakeInstruction::Merge => {
            // Source account folded in; combine balances at replay time.
        }
        // SetLockup / SetLockupChecked / AuthorizeChecked / AuthorizeWithSeed /
        // AuthorizeCheckedWithSeed / InitializeChecked / GetMinimumDelegation /
        // MoveStake / MoveLamports — apply per program semantics.
        _ => {}
    }
    Ok(())
}

/// Was the stake account's delegation active during `epoch`?
pub fn was_active_during(state: &StakeStateView, epoch: u64) -> bool {
    let activated = state.activation_epoch.map_or(false, |a| a <= epoch);
    let still_active = state.deactivation_epoch.map_or(true, |d| d > epoch);
    !state.closed && activated && still_active
}
```

For high-volume historical attribution, **don't replay per query** — materialize per-account state snapshots at each epoch boundary into Postgres once, stream forward thereafter. The replay path above is for cold-start backfill or one-off reconciliation; ongoing reward attribution should use cached snapshots indexed by `(stake_account, epoch)`.

## Partitioned epoch rewards: scanning the distribution window

Post-SIMD-0118, staking rewards spread across a window of slots at the start of each new epoch — not a single block. The empirically observed window can extend to ~300 slots (see [forks-changelog.md](forks-changelog.md#partitioned-epoch-rewards-simd-0118) for the epoch-937 measurement). Scan with margin:

```rust
const REWARD_SCAN_WINDOW: u64 = 500;  // slots; safety margin over observed ~300

pub async fn collect_staking_rewards_for_epoch(
    rpc:              &RpcClient,
    epoch_first_slot: u64,
) -> anyhow::Result<Vec<RewardEntry>> {
    let mut all = Vec::new();
    let mut max_offset_seen: u64 = 0;

    for offset in 1..=REWARD_SCAN_WINDOW {
        let slot = epoch_first_slot + offset;
        let block = match rpc.get_block(slot).await {
            Ok(b)  => b,
            Err(_) => continue,                    // skipped slot
        };
        for r in &block.rewards {
            if r.reward_type == Some(RewardType::Staking) {
                all.push(RewardEntry {
                    slot,
                    pubkey: r.pubkey.clone(),
                    lamports: r.lamports,
                    post_balance: r.post_balance,
                    commission: r.commission,
                });
                max_offset_seen = max_offset_seen.max(offset);
            }
        }
    }

    // Operational metric: alert if window approaches the scan limit.
    metrics::gauge!("solana.partitioned_rewards.max_offset", max_offset_seen as f64);
    if max_offset_seen > REWARD_SCAN_WINDOW * 4 / 5 {
        tracing::warn!(epoch_first_slot, max_offset_seen,
            "rewards distribution approaching scan window — consider widening");
    }

    Ok(all)
}
```

The metric in the warning branch is the early-warning system: if your `max_offset_seen` per epoch is creeping toward the window limit over time, widen `REWARD_SCAN_WINDOW` before you start silently dropping reward entries.

## What's not shown

- **Reorg handling at `processed`** — ingest at `confirmed` instead and avoid the problem.
- **ClickHouse batch writers** — buffer rows, flush every 1–5 s or every 10k rows, whichever comes first.
- **Backfill from JSON-RPC** — use `getBlocks(start, end)` to enumerate slots, then `getBlock(slot)` per slot, parallelized.
- **Halt detection / liveness alerts** — monitor slot advancement rate; alert below threshold.
- **Provider failover** — Geyser providers occasionally hiccup; have a secondary endpoint.
