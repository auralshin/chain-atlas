# Forks and Upgrades

Chronological. The "indexer impact" column flags whether a code-path change is required for blocks at or after the fork.

| Date | Name | EL block | CL epoch | Headline EIPs | Indexer impact |
|---|---|---|---|---|---|
| 2015-07-30 | Frontier | 0 | — | — | Genesis |
| 2015-08-04 | Frontier Thawing | 200,000 | — | — | Lifted genesis 5k gas cap |
| 2016-02-29 | Homestead | 1,150,000 | — | EIP-2, 7, 8 | Protocol foundation |
| 2016-07-20 | DAO Fork | 1,920,000 | — | — | Fund recovery; chain split (ETC) |
| 2016-10-18 | Tangerine Whistle | 2,463,000 | — | EIP-150 | Opcode repricing |
| 2016-11-22 | Spurious Dragon | 2,675,000 | — | EIP-155, 160, 161, 170 | **Replay protection (chain ID in sigs)** |
| 2017-10-16 | Byzantium | 4,370,000 | — | EIP-100, 140, 196, 197, 198, 211, 214, 658 | **Receipt `status` field replaces post-state root** |
| 2019-02-28 | Constantinople / Petersburg | 7,280,000 | — | EIP-145, 1014, 1052, 1234 | `CREATE2` opcode |
| 2019-12-08 | Istanbul | 9,069,000 | — | EIP-152, 1108, 1344, 1884, 2028, 2200 | `CHAINID` opcode |
| 2020-01-02 | Muir Glacier | 9,200,000 | — | EIP-2384 | Difficulty bomb delay |
| 2020-12-01 | Beacon Chain genesis | — | 0 | — | CL launch (no EL impact yet) |
| 2021-04-15 | Berlin | 12,244,000 | — | EIP-2565, 2718, 2929, 2930 | **Tx envelope (EIP-2718); type `0x01`** |
| 2021-08-05 | London | 12,965,000 | — | EIP-1559, 3198, 3529, 3541 | **Tx type `0x02`; `baseFeePerGas` header field** |
| 2021-10-27 | Altair | — | 74,240 | — | CL only (sync committees) |
| 2021-12-09 | Arrow Glacier | 13,773,000 | — | EIP-4345 | Difficulty bomb delay |
| 2022-06-30 | Gray Glacier | 15,050,000 | — | EIP-5133 | Difficulty bomb delay |
| 2022-09-06 | Bellatrix | — | 144,896 | — | CL prep for Merge |
| 2022-09-15 | Paris (The Merge) | 15,537,394 | — | EIP-3675, 4399 | **PoS. `mixHash`→`prevRandao`. `difficulty`=0. `nonce`=0. `ommers`=∅.** |
| 2023-04-12 | Shanghai-Capella (Shapella) | 17,034,870 | 194,048 | EIP-3651, 3855, 3860, 4895 | **Withdrawals in block body. `withdrawalsRoot` header field.** |
| 2024-03-13 | Cancun-Deneb (Dencun) | 19,426,587 | 269,568 | EIP-1153, 4788, 4844, 5656, 6780, 7044, 7045, 7514, 7516 | **Tx type `0x03` (blobs). `blobGasUsed`/`excessBlobGas`/`parentBeaconBlockRoot` header fields.** |
| 2025-05-07 | Prague-Electra (Pectra) | 22,431,084 | 364,032 | EIP-2537, 2935, 6110, 7002, 7251, 7549, 7623, 7691, 7702, 7840 | **Tx type `0x04` (set-code). EOAs can have code. Validator max effective balance 2048 ETH. Blob target 6 / max 9.** |
| 2025-12-03 | Fulu-Osaka (Fusaka) | 23,935,694 | 411,392 | EIP-7594, 7892, 7918, 7642, 7823, 7825, 7883, 7934, 7935, 7917, 7939, 7951, 7910 | **PeerDAS for blob DA. Per-tx gas cap 16,777,216 (2²⁴). secp256r1 precompile. `eth_config` RPC method.** |

## Scheduled (not live as of 2026-05-08)

| Target | Name | Headline EIPs |
|---|---|---|
| H1 2026 | Glamsterdam (Gloas + Amsterdam) | EIP-7732 (ePBS), EIP-7928 (Block-level Access Lists), EIP-7904 (gas repricing) |

## Notes on indexer-relevant upgrades

### Spurious Dragon (2016-11-22)

EIP-155 added chain-ID to transaction signatures (replay protection). Indexers covering pre-Spurious-Dragon blocks must handle txs without chain-ID in their signature scheme.

### Byzantium (2017-10-16)

Receipts changed format. Pre-Byzantium receipts contained a post-state root (`postStateRoot`); from Byzantium onward they use a 1/0 `status` field (EIP-658). An indexer covering pre-Byzantium history must handle both shapes.

### Berlin (2021-04-15)

Introduced **EIP-2718 typed transactions**. From here on, transactions can be encoded with a leading type byte (`0x01..`) and a type-specific payload. Indexers must check the first byte of the encoded tx and dispatch to the right decoder. Type `0x01` (EIP-2930 access list) was the first typed tx.

### London (2021-08-05)

EIP-1559: added `baseFeePerGas` to block headers and **transaction type `0x02`** (`maxFeePerGas` / `maxPriorityFeePerGas`). The `gasPrice` field on a type-2 receipt is `effectiveGasPrice = min(maxFeePerGas, baseFeePerGas + maxPriorityFeePerGas)` — not the same as `maxFeePerGas`.

### The Merge (2022-09-15)

Execution-layer effects: `difficulty=0`, `nonce=0`, `ommers=∅`, `mixHash` repurposed as `prevRandao` (RANDAO output for that slot). Pre-Merge code paths that compute or rely on `difficulty` for any analytical purpose silently return 0 from this block onward.

### Shapella (2023-04-12)

Withdrawals appear in block bodies as a separate list. **They are not transactions.** An indexer tracking ETH balance flows must add a withdrawal-credit code path. Withdrawal amounts are gwei, not wei.

### Dencun (2024-03-13)

**Tx type `0x03`** (blob-carrying). Type-3 txs reference blob bodies via `blobVersionedHashes`. Blob bodies live on the beacon node and are retained ~18 days. Indexers needing historical blob bodies must persist them on ingest.

Three new header fields: `blobGasUsed`, `excessBlobGas`, `parentBeaconBlockRoot` (EIP-4788). The last enables on-chain access to beacon chain state — relevant for restaking protocols (EigenLayer-style).

### Pectra (2025-05-07)

**Tx type `0x04`** (set-code, EIP-7702). An EOA can attach contract code for the duration of a transaction via an `authorizationList`. Indexer consequence: an account that returns empty `eth_getCode` at block N can return a delegation pointer at block N+1, then empty again at N+2 — account "type" is no longer a static property.

EIP-7251 raised max effective balance to 2048 ETH; validator-tracking indexers must update consolidation logic. EIP-6110 surfaces validator deposits on the EL via the EIP-7685 *execution requests* framework (not a new tx type) — deposit-contract log events are aggregated into a block-level `executionRequests` field that the CL consumes directly, removing the eth1-data voting flow. EIP-7002 enables execution-layer-triggerable validator exits via the same requests framework. [verify exact request-type encoding]

### Fusaka (2025-12-03)

Headline indexer-relevant changes:

- **PeerDAS** (EIP-7594) — blob data is now sampled column-wise across nodes rather than fully replicated. Beacon API still serves full sidecars on request, but reconstruction may have additional latency, especially near the retention boundary.
- **EIP-7825** — per-transaction gas cap of 16,777,216 (2²⁴) gas. Single txs can no longer consume an entire block.
- **EIP-7951** — secp256r1 (P-256) precompile. Indexers tracking on-chain signature schemes need to recognize the new precompile address.
- **EIP-7910** — `eth_config` JSON-RPC method, returning the active EIP set, fork timing, and chain config. Useful for indexers that auto-detect chain capabilities.
- **EIP-7918** — blob base-fee bounded by execution costs (changes blob fee dynamics).
- **EIP-7892** — blob-parameter-only forks (allows blob limits to change without a full hard fork).

EIP-7685 *execution requests* (introduced framework-level in Pectra) continues to carry deposit and exit data; Fusaka does not change the requests format.
