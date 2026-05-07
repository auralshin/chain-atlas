# RPC Surface

Polygon indexer needs **three** RPC endpoints: `bor` (EVM), `heimdall` (Tendermint+Cosmos), and Ethereum L1.

## `bor` JSON-RPC

Standard Ethereum JSON-RPC. All `eth_*`, `web3_*`, `net_*` methods work. Notable:

| Method | Notes |
|---|---|
| `eth_getBlockByNumber` | Standard ETH block; `miner` is the sprint producer. |
| `eth_getTransactionReceipt` | Standard ETH receipt. **No state-sync info here** — see heimdall. |
| `eth_getBlockByNumber("safe")` | Post-Bhilai: heimdall-derived safe head. |
| `eth_getBlockByNumber("finalized")` | Post-Bhilai: heimdall-derived finalized head. |
| `eth_subscribe("newHeads")` | Standard. Bor reorgs surface here as expected. |
| `bor_getCurrentValidators` | Active validator set for the current span. |
| `bor_getCurrentProposer` | Sprint producer for the current sprint. |
| `bor_getRootHash` | Merkle root of a block range, used by heimdall checkpoint construction. |
| `bor_getSignersAtHash` | Validator signatures attesting to a given block hash. |

The `bor_*` namespace is small and mostly diagnostic.

## Trace APIs (`bor`)

`bor` is a `geth` fork and supports `debug_traceBlockByNumber`, `debug_traceTransaction`, `debug_traceCall` — same shape as Ethereum.

Some providers (Alchemy, QuickNode) expose Erigon-style `trace_*` flat traces against Polygon. Provider-dependent.

## `heimdall` RPC

Two interfaces, both useful for indexers:

### Tendermint RPC (port 26657 by default)

```
GET  /status
GET  /block?height=N
GET  /tx?hash=0x...
GET  /tx_search?query=...
WS   /websocket  (subscribe to NewBlock, Tx events)
```

For tracking heimdall block heights and validator votes.

### Cosmos REST (port 1317 by default)

```
GET  /staking/validators
GET  /staking/validator/<addr>
GET  /bor/span/<span_id>
GET  /bor/spans/list
GET  /clerk/event-record/list?from-id=...
GET  /clerk/event-record/<state_id>
GET  /checkpoints/list?page=...
GET  /checkpoints/<id>
GET  /checkpoints/last-no-ack
```

The `clerk/event-record/*` endpoints are how you index state syncs without scraping bor for ghost activity. The `checkpoints/*` endpoints give you finality state.

## Ethereum L1 endpoints needed

For complete indexing:

- `RootChain` / `RootChainManager` — checkpoint posting + bridge entry
- `StateSender` — `StateSynced` events
- `StakeManager` — validator stake changes
- `DepositManager` (legacy / Plasma bridge) — for Plasma-era data

Standard ETH JSON-RPC + `eth_getLogs` against these contracts is enough.

## Archive node requirements

`bor` archive: standard ETH archive disk profile, multi-TB. State at any historical block.

`heimdall` archive: Tendermint full node with full block history + Cosmos state. **Substantially smaller** than bor archive.

Most production indexers run:
- `bor` archive (self-hosted or paid provider)
- `heimdall` full node (self-hosted; relatively cheap)
- Ethereum archive (provider, since L1 archive is expensive)

## Public provider quirks

- **`eth_getLogs` block range limits** are tight on Polygon (often 1k–3k blocks) because of high tx volume per block. Plan for short windows.
- **WebSocket reorg behavior**: `eth_subscribe("newHeads")` emits new heads as they arrive on `bor`'s view. **Reorgs surface implicitly**: a parent-hash mismatch means a reorg occurred. Some providers buffer / dedupe, some don't. Test your provider's behavior on a known reorg.
- **Heimdall RPC is rarely exposed by public providers** — it's an internal coordination layer. Self-host if you need it. Some providers expose a curated subset (e.g. `/checkpoints/list`).

## Picking transports

| Transport | Use when |
|---|---|
| HTTP `bor` | Bulk historical scans |
| WebSocket `bor` | Real-time `newHeads` + `logs` |
| HTTP `heimdall` | Periodic polling for checkpoints, state syncs, validator changes |
| WebSocket `heimdall` (Tendermint) | Real-time checkpoint / validator events |
| HTTP L1 | Periodic polling for `StateSender`, `RootChain`, `StakeManager` events |

A typical poll cadence: bor at WS for newHeads; heimdall every 30 seconds for checkpoints + state syncs; L1 every 30 seconds for L1 events.
