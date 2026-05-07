# RPC Surface

Two interfaces matter: **execution-layer JSON-RPC** (geth/reth/erigon/...) and **beacon API** (lighthouse/prysm/...). Many indexers also pull from a **trace/debug** namespace, which is **not portable** across EL clients.

## Execution-layer JSON-RPC (standard)

Methods every Ethereum indexer uses:

| Method | Purpose | Archive required |
|---|---|---|
| `eth_blockNumber` | Latest block | No |
| `eth_getBlockByNumber` | Block header + tx hashes (or full txs with `true` arg) | No |
| `eth_getBlockReceipts` | All receipts for a block in one call | No |
| `eth_getTransactionByHash` | Full tx | No |
| `eth_getTransactionReceipt` | Single receipt | No |
| `eth_getLogs` | Filtered log query | No (recent) / Yes (historical, depending on client retention) |
| `eth_getProof` | Account/storage proof at block | Yes (hash-based archive) |
| `eth_getCode` | Code at address at block | Yes for historical blocks |
| `eth_call` (with `blockTag`) | State read at block | Yes for historical blocks |
| `eth_getStorageAt` | Slot read at block | Yes for historical |

Prefer `eth_getBlockReceipts` over per-tx `eth_getTransactionReceipt` — one round trip vs N. Available on geth (1.13+), reth, erigon, besu, nethermind.

## Trace / debug methods

This is where clients diverge. A method that works on geth may not exist on reth, may have different output on erigon, etc.

| Method | geth | reth | erigon | nethermind | besu |
|---|---|---|---|---|---|
| `debug_traceTransaction` | ✅ | ✅ | ✅ | ✅ | ✅ |
| `debug_traceBlockByNumber` | ✅ | ✅ | ✅ | ✅ | ✅ |
| `debug_traceCall` | ✅ | ✅ | ✅ | ✅ | ✅ |
| `trace_transaction` (parity) | ❌ | partial | ✅ | ✅ | ✅ |
| `trace_block` (parity) | ❌ | partial | ✅ | ✅ | ✅ |
| `trace_replayBlockTransactions` | ❌ | partial | ✅ | ✅ | partial |
| `ots_*` (Otterscan namespace) | ❌ | ✅ | ✅ | ❌ | ❌ |

[verify: matrix against current client docs — drift expected, especially on reth which has been actively adding methods]

**Tracer modes (passed as the second arg to `debug_traceTransaction`):**

| Tracer | Output |
|---|---|
| `callTracer` | Call frames — most indexers want this |
| `prestateTracer` | Pre-tx state read by the tx |
| `4byteTracer` | Function selector counts |
| `noopTracer` | Nothing — useful for benchmarking |
| Custom JS / Lua / Rust | Per-client custom tracers |

`callTracer` with `{ withLog: true }` produces nested call frames including emitted logs, which is what most "internal tx" indexers consume.

## Beacon API

Run a CL alongside the EL. Methods an indexer needs:

| Endpoint | Purpose |
|---|---|
| `/eth/v1/beacon/headers/{block_id}` | CL block header |
| `/eth/v2/beacon/blocks/{block_id}` | Full CL block |
| `/eth/v1/beacon/blob_sidecars/{block_id}` | Blob bodies (EIP-4844) |
| `/eth/v2/beacon/states/{state_id}/finality_checkpoints` | Justified / finalized checkpoints |
| `/eth/v1/events?topics=head,finalized_checkpoint` | SSE stream |
| `/eth/v1/beacon/states/{state_id}/validators` | Validator state (for staking indexers) |

[verify: paths against current Beacon API spec — `eth/v1` vs `eth/v2` versioning shifts per upgrade]

## Subscription / streaming

JSON-RPC over WebSocket:

| Subscription | Stream |
|---|---|
| `newHeads` | New block headers |
| `logs` (with filter) | Filtered logs as they're mined |
| `newPendingTransactions` | Mempool txs (full or hash) |

For high-throughput indexers, JSON-RPC has fundamental latency and parallelism limits. Alternatives:

- **reth ExEx** (Execution Extensions) — in-process plugin running inside the client; lowest-latency block stream, no JSON serialization cost
- **erigon snapshots** — historical bulk download via snapshot files, then live via RPC
- **Substreams** (StreamingFast / Firehose) — out-of-band streaming layer if you don't want to operate a node yourself

## Archive requirements

Definition: an archive node retains historical state and execution history, not just recent state.

| Client | Archive footprint (early 2026) | Notes |
|---|---|---|
| geth (path-based) | ~1.9–2.0 TB | New default. **Does not yet support `eth_getProof` over historical state.** |
| geth (hash-based) | 12–20+ TB | Legacy. Required for historical Merkle proofs. |
| erigon | ~1.8–2.2 TB | Designed for compact archive from start |
| reth | ~2.8 TB | |

[verify: sizes drift with chain growth — re-check before quoting in production planning]

Methods that require archive: anything with `blockTag` ≠ `latest`/`pending` against historical state — `eth_call`, `eth_getStorageAt`, `eth_getProof`, `eth_getCode`, all `debug_trace*` and `trace_*` against old blocks.

## Rate limit reality

Public RPC endpoints rate-limit aggressively. A backfill at 1k req/sec from a public endpoint will get IP-banned. Self-host or use paid providers (Alchemy, Infura, QuickNode, BlockPi, etc.) for backfill workloads.

Per-block request budget for a thorough indexer: 1 block call + 1 `eth_getBlockReceipts` + 1 traces call (if used) ≈ 3 RPCs per block. At 12 s/block that's ≤ 1 RPS steady state — but backfill of 23M+ blocks at 100–500 RPS still takes days.
