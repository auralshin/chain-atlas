# RPC Surface

StarkNet has its own JSON-RPC standard. **No `eth_*` methods.** All methods are `starknet_*`.

The spec is open and versioned: [`starkware-libs/starknet-specs`](https://github.com/starkware-libs/starknet-specs).

## Core methods

### Block / chain info

| Method | Returns | Why |
|---|---|---|
| `starknet_blockNumber` | u64 | Latest sealed L2 block. |
| `starknet_blockHashAndNumber` | { hash, number } | Latest sealed L2 block + hash. |
| `starknet_getBlockWithTxs(block_id)` | Block + full txs | Standard ingestion call. |
| `starknet_getBlockWithTxHashes(block_id)` | Block + tx hashes only | Lighter; for skim |
| `starknet_getBlockWithReceipts(block_id)` | Block + txs + receipts in one call | **Preferred for indexers** — saves N round-trips |
| `starknet_chainId` | felt252 | Chain ID. Mainnet: `SN_MAIN`; testnet: `SN_GOERLI`/`SN_SEPOLIA`. |
| `starknet_specVersion` | string | RPC spec version (e.g. "0.7.1"). Pin compatibility. |

`block_id` is one of: `"latest"`, `"pending"`, `{ "block_number": N }`, `{ "block_hash": "0x..." }`.

### Tx and receipt

| Method | Returns | Why |
|---|---|---|
| `starknet_getTransactionByHash(hash)` | Tx | |
| `starknet_getTransactionReceipt(hash)` | Receipt | Includes `events`, `messages_sent`, `execution_resources`. |
| `starknet_getTransactionStatus(hash)` | { execution_status, finality_status } | Quick status check. |
| `starknet_getTransactionByBlockIdAndIndex(block_id, index)` | Tx by position | |

### State queries

| Method | Returns | Why |
|---|---|---|
| `starknet_getStateUpdate(block_id)` | State diff for the block | **Most important call for indexers ingesting state.** |
| `starknet_getStorageAt(address, key, block_id)` | Storage value | Random access. |
| `starknet_getNonce(address, block_id)` | Account nonce | |
| `starknet_getClassHashAt(address, block_id)` | class_hash at this address | |
| `starknet_getClass(class_hash, block_id)` | The Sierra/CASM bytecode | Cache forever — classes are immutable. |
| `starknet_getClassAt(address, block_id)` | Convenience: class at this address | |

### Events

| Method | Returns | Why |
|---|---|---|
| `starknet_getEvents(filter)` | Page of events matching filter | Roughly equivalent to `eth_getLogs`. |

`filter` accepts: `from_block`, `to_block`, `address`, `keys` (array of arrays — outer is OR, inner is AND), `chunk_size`, `continuation_token`.

The events query is **paginated** with continuation tokens (similar to AWS-style pagination), not block-range-bounded like `eth_getLogs`. This is friendlier for indexers — fewer rate-limit shaped queries.

### Estimation / simulation

| Method | Returns | Why |
|---|---|---|
| `starknet_estimateFee(tx, block_id)` | Fee estimate | Standard. |
| `starknet_estimateMessageFee(message, block_id)` | L1→L2 message fee | Specific to bridge UIs. |
| `starknet_call(call, block_id)` | Read-only call | Read storage / view functions without a tx. |
| `starknet_simulateTransactions(...)` | Simulated execution | For preview / dry-run. |

### Subscription methods (post-V0.7+ spec)

WebSocket subscriptions:
- `starknet_subscribeNewHeads`
- `starknet_subscribeEvents`
- `starknet_subscribePendingTransactions`
- `starknet_subscribeTransactionStatus`

{{unsourced: confirm subscription support varies by client — pathfinder vs juno vs papyrus}}

## Trace methods

| Method | Returns |
|---|---|
| `starknet_traceTransaction(hash)` | Full execution trace |
| `starknet_traceBlockTransactions(block_id)` | All txs in a block |

Traces include:
- `validate_invocation` — the `__validate__` phase.
- `execute_invocation` — the `__execute__` phase.
- `fee_transfer_invocation` — fee charging.
- Nested calls within each phase.

For an indexer building "internal txs"–style analytics, traces are essential. They are the StarkNet equivalent of geth's `debug_traceTransaction` output.

## Picking a client

| Client | Language | Notable for |
|---|---|---|
| `pathfinder` | Rust | Performance; widely deployed. |
| `juno` | Go | Nethermind-built; modern; growing. |
| `papyrus` | Rust | StarkWare-built; reference implementation. |

All three serve the same JSON-RPC spec. Indexers can use any, with provider-specific caveats around what optional methods are exposed.

## Public provider quirks

- **Public RPC**: many options ([blastapi.io](https://blastapi.io/), [infura.io](https://www.infura.io/) for StarkNet, [alchemy.com](https://www.alchemy.com/) for StarkNet, dRPC, etc.).
- **Block range / event pagination**: usually 1k events per page, paginated via continuation tokens.
- **`starknet_subscribe*` support varies** — not all providers expose WebSocket subscriptions.
- **Trace endpoint cost**: traces are heavy; archive-tier on most providers.

## Picking a transport

| Transport | Use when |
|---|---|
| HTTP | Standard ingestion, event scans, state queries |
| WebSocket | Real-time `subscribeNewHeads` if supported by your provider |
| Custom binary protocols | None — JSON-RPC only |

For self-hosted indexing, run `pathfinder` or `juno` locally and connect via HTTP/WebSocket.

## Archive node requirements

StarkNet archive: full historical state. As of writing, multi-hundred-GB to low-TB depending on client (pathfinder vs juno vs papyrus have different storage profiles) {{unsourced: confirm current sizes}}.
