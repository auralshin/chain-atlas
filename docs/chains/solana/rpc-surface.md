# RPC Surface

Solana exposes JSON-RPC, WebSocket subscriptions, and the **Geyser plugin** for high-throughput streaming. Production indexers use Geyser; JSON-RPC is for ad-hoc queries.

## JSON-RPC core methods

### Block / slot

| Method | Returns | Why |
|---|---|---|
| `getSlot` | Current slot at given commitment | Liveness |
| `getBlockHeight` | Current block height | Sequential block index (≠ slot) |
| `getBlock(slot)` | Full block (txs + receipts + meta) | Standard ingestion |
| `getBlocks(start, end)` | List of slots that produced blocks | Backfill (skipped slots not in result) |
| `getBlockTime(slot)` | Unix timestamp | |
| `getEpochInfo` | Current epoch + slot index within epoch | |

`getBlock` returns everything for a slot (txs, inner instructions, logs, balances, rewards). One call is usually enough.

### Transactions

| Method | Returns | Why |
|---|---|---|
| `getTransaction(signature)` | Single tx with full meta | For per-tx queries |
| `getSignaturesForAddress(addr, opts)` | Recent signatures touching an address | Backwards walk: each call returns up to 1000 sigs |

`getSignaturesForAddress` is the primary backfill primitive for "all activity for this address." Note: it returns signatures, not full txs; follow up with `getTransaction(sig)`.

### Accounts

| Method | Returns | Why |
|---|---|---|
| `getAccountInfo(pubkey)` | Account state at commitment | Random access |
| `getMultipleAccounts(pubkeys[])` | Up to 100 accounts in one call | Batched fetch |
| `getProgramAccounts(programId, filters)` | All accounts owned by a program | **Use sparingly** — can be huge |

`getProgramAccounts` is operationally expensive. Most providers gate it heavily. For "all SPL Token accounts for owner X" use `getTokenAccountsByOwner` instead.

### Tokens

| Method | Returns | Why |
|---|---|---|
| `getTokenAccountsByOwner(owner, mint_or_program)` | Token accounts owned by `owner` | Wallet UI |
| `getTokenAccountBalance(account)` | Decoded balance | |
| `getTokenLargestAccounts(mint)` | Top holders | |
| `getTokenSupply(mint)` | Total + circulating | |

### Validators / consensus

| Method | Returns | Why |
|---|---|---|
| `getVoteAccounts` | Active vote accounts + stake | For consensus-aware indexing |
| `getEpochSchedule` | Epoch parameters | |
| `getInflationGovernor` | Staking rewards parameters | |

### Compute / fee

| Method | Returns | Why |
|---|---|---|
| `getFeeForMessage(message, commitment)` | Fee for a hypothetical tx | UI estimation |
| `getRecentPrioritizationFees(addresses)` | Recent priority fee distribution | UX hints |

## WebSocket subscriptions

| Method | Notifies on |
|---|---|
| `slotSubscribe` | Each new slot |
| `slotsUpdatesSubscribe` | Slot status changes (created, completed, optimisticConfirmed, rooted) |
| `blockSubscribe` | New block at given commitment |
| `signatureSubscribe(sig)` | When a specific signature is confirmed |
| `accountSubscribe(pubkey)` | Account state changes |
| `programSubscribe(programId, filters)` | Account changes within a program |
| `logsSubscribe(filter)` | Log messages matching filter |

WebSocket scales to a few subscriptions; not to the whole chain. For full-stream indexing, use Geyser.

## Geyser plugin (gRPC)

The production interface for high-throughput indexing. Geyser is a validator-side plugin system that streams:

- **Slot updates** (status transitions).
- **Account writes** (every account modification, with full state).
- **Transactions** (full txs as they execute, with logs and inner instructions).
- **Block notifications** (when a block is produced).

Two production options:

### 1. Run a Geyser-enabled validator

Self-hosted. Substantial operational cost (Solana validator hardware: 256 GB+ RAM, 12-core+ CPU, 2 TB+ NVMe). Stream over gRPC to your indexer.

### 2. Use a Geyser-tier provider

[Triton](https://triton.one/), [Helius](https://helius.dev/), [Jito](https://jito.network/), [Yellowstone](https://github.com/rpcpool/yellowstone-grpc), [Shyft](https://shyft.to/), etc. provide Geyser gRPC endpoints. Filter and consume.

The canonical gRPC schema is **Yellowstone gRPC** (`yellowstone-grpc-proto`) — most providers support it.

## Geyser via `yellowstone-grpc` schema

Subscribe with filters:

```protobuf
SubscribeRequest {
  accounts: { ... }       // Filter accounts: by owner, by pubkey, by data slice
  slots: { ... }
  transactions: { ... }    // Filter txs: by required account, by failed/success, etc.
  blocks: { ... }
  blocks_meta: { ... }
}
```

The server streams matching events. **Filtering is server-side** — much more efficient than `accountSubscribe` per-account.

## Archive / historical data

Solana validators do not retain full historical state by default. Three options for historical access:

1. **Archival RPC providers** (Helius, Triton, Solscan, etc.) maintain historical block + tx data.
2. **Big Table snapshots** (Solana Foundation's archive on Google BigQuery — historical txs).
3. **Self-host historical** with a long-retention validator + database — expensive.

For an indexer, ingesting historical data usually means **a one-time backfill from a provider** + ongoing real-time ingestion.

## Picking a transport

| Workload | Pick |
|---|---|
| One-off queries (lookups, balances) | JSON-RPC over HTTP |
| Real-time + light filtering | WebSocket subscriptions |
| Real-time + full-chain | **Geyser gRPC** (Yellowstone schema) |
| Backfill | RPC `getBlock` per slot, parallelized |

For production indexers, Geyser is the only sustainable path. Plan for it.

## Compute units in receipts

Each tx returns `computeUnitsConsumed` in `meta`. Per-block aggregations:

- Total CU consumed per block.
- CU price distribution (priority fee market).

These are operational metrics; useful for fee analysis dashboards.
