# Data Model

## Block

Header fields an indexer extracts (post-Fusaka). Listed in approximate order of when they were introduced.

| Field | Type | Introduced | Notes |
|---|---|---|---|
| `parentHash` | bytes32 | Frontier | |
| `sha3Uncles` / `ommersHash` | bytes32 | Frontier | Post-Merge: always `0x1dcc4de8de...` (empty list hash) |
| `miner` / `feeRecipient` | address | Frontier | Post-Merge: proposer's fee recipient |
| `stateRoot` | bytes32 | Frontier | MPT root after this block |
| `transactionsRoot` | bytes32 | Frontier | |
| `receiptsRoot` | bytes32 | Frontier | |
| `logsBloom` | bytes256 | Frontier | |
| `difficulty` | u256 | Frontier | Post-Merge: always `0` |
| `number` | u64 | Frontier | |
| `gasLimit` | u64 | Frontier | |
| `gasUsed` | u64 | Frontier | |
| `timestamp` | u64 | Frontier | Unix seconds |
| `extraData` | bytes | Frontier | Up to 32 bytes |
| `mixHash` / `prevRandao` | bytes32 | Frontier / Merge | Re-purposed at Merge: now the RANDAO output |
| `nonce` | u64 | Frontier | Post-Merge: always `0x0000000000000000` |
| `baseFeePerGas` | u256 | London | Pre-London blocks have this field absent |
| `withdrawalsRoot` | bytes32 | Shapella | |
| `blobGasUsed` | u64 | Cancun-Deneb | |
| `excessBlobGas` | u64 | Cancun-Deneb | |
| `parentBeaconBlockRoot` | bytes32 | Cancun-Deneb | EIP-4788 |

**Indexer note:** historical blocks pre-London do not have `baseFeePerGas`. Pre-Shapella blocks do not have `withdrawalsRoot`. Pre-Cancun do not have blob fields. A unified schema must allow these to be NULL.

Body components: transactions list (typed; see below), uncles list (always empty post-Merge), withdrawals list (post-Shapella).

## Transactions

Five live transaction types as of 2026-05-08 (post-Pectra):

| Type | EIP | Fork | Distinguishing fields |
|---|---|---|---|
| `0x00` (legacy) | — | Frontier | `gasPrice` |
| `0x01` | EIP-2930 | Berlin | `accessList` |
| `0x02` | EIP-1559 | London | `maxPriorityFeePerGas`, `maxFeePerGas`, `accessList` |
| `0x03` | EIP-4844 | Cancun-Deneb | `maxFeePerBlobGas`, `blobVersionedHashes` |
| `0x04` | EIP-7702 | Pectra | `authorizationList` |

**Decoding:**

- Type-0 txs are legacy RLP without a leading type byte.
- Type-1+ are EIP-2718 typed envelopes: a one-byte type prefix followed by RLP of the type-specific payload.
- An indexer that only handles type-0 silently mis-parses everything from Berlin onward.

### Type 0x03 — blob-carrying

The tx body contains `blobVersionedHashes: [bytes32]`, but the **blob payloads themselves** travel separately as a "sidecar" alongside the block on the consensus layer.

```
TxEip4844:
    chainId, nonce, maxPriorityFeePerGas, maxFeePerGas,
    gasLimit, to, value, data, accessList,
    maxFeePerBlobGas, blobVersionedHashes,
    signature
```

Mainnet retains blob sidecars for ~18 days (4096 epochs, derived from EIP-4444 retention policy). Indexers that need historical blob bytes must persist them at ingest time. See [gotchas.md](gotchas.md#blobs-arent-in-json-rpc).

### Type 0x04 — set-code (EIP-7702)

The tx contains an `authorizationList`. Each entry is a tuple `[chain_id, address, nonce, y_parity, r, s]`, signed independently by the *authority* (the EOA delegating). The signing pre-image is `keccak(0x05 || rlp([chain_id, address, nonce]))` — note the `0x05` domain separator.

```
TxEip7702:
    chainId, nonce, maxPriorityFeePerGas, maxFeePerGas,
    gasLimit, to, value, data, accessList,
    authorizationList,
    signature
```

After execution, the authority's `code` is set to the 23-byte delegation indicator `0xef0100 || address` (3-byte prefix + 20-byte target address). The authority is now an EOA-with-code: subsequent calls execute the delegated code with the authority's storage as the execution context. Note: `EXTCODESIZE` returns 23 (the indicator length), but `CODESIZE` inside the delegated execution returns the size of the actual code at the target — these intentionally diverge per [EIP-7702 spec](https://eips.ethereum.org/EIPS/eip-7702).

**Indexer consequence:** "is this address a contract?" is no longer a static property. An account that returned empty `eth_getCode` at block N can return a delegation pointer at N+1. Cache invalidation logic must account for this.

## Receipts

Per-tx receipt fields:

| Field | Notes |
|---|---|
| `status` | 1 = success, 0 = revert. **Post-Byzantium** (block 4,370,000). Pre-Byzantium used `postStateRoot`. |
| `cumulativeGasUsed` | |
| `gasUsed` | |
| `logs` | List of log entries |
| `logsBloom` | |
| `type` | Matches tx type |
| `effectiveGasPrice` | Post-London. For type-2: `min(maxFeePerGas, baseFeePerGas + maxPriorityFeePerGas)`. |
| `blobGasUsed`, `blobGasPrice` | Type-3 only |
| `contractAddress` | Set when tx is contract creation, NULL otherwise |

## Logs / Events

Each log entry:

| Field | Notes |
|---|---|
| `address` | The contract that emitted |
| `topics` | 0–4 entries. `topics[0]` is the event signature hash for non-anonymous events. |
| `data` | Non-indexed event arguments, ABI-encoded |
| `blockNumber`, `transactionHash`, `logIndex`, `transactionIndex`, `removed` | Standard metadata |

`removed: true` indicates the log was un-mined by a reorg. Some clients emit removal events on subscription channels; some don't — confirm per client.

**Internal calls do not emit logs.** A `CALL`/`DELEGATECALL` from contract A to contract B is invisible to `eth_getLogs` unless B emits an event. To extract internal value flows, use trace APIs ([rpc-surface.md](rpc-surface.md#trace--debug-methods)).

## Withdrawals

Since Shapella (2023-04-12), validator withdrawals appear in block bodies as a separate list:

```
[ index, validatorIndex, address, amount ]
```

**Withdrawals are not transactions.** They have no hash, are not signed, do not appear in `eth_getBlockReceipts`, and do not emit logs. An indexer tracking ETH balances must add a separate code path for withdrawal credits. **`amount` is denominated in gwei**, not wei — multiply by 1e9 before adding to a wei-denominated balance.

## Blobs (EIP-4844)

A blob is a 4096-element vector of BLS12-381 field elements (~125 KB). Blobs are referenced from type-3 transactions via KZG commitment hashes (`blobVersionedHashes`). The blob bytes themselves:

- Are **not** in the EL block body
- Are **not** retrievable via JSON-RPC
- Are retained ~18 days by beacon nodes
- Must be fetched via Beacon API: `GET /eth/v1/beacon/blob_sidecars/{block_id}`

Each sidecar carries: blob bytes, KZG commitment, KZG proof, parent block index, and an inclusion proof.

Post-Fusaka (2025-12-03), **PeerDAS** (EIP-7594) changes how blobs are stored across the network — nodes custody column subsets and sample others. Beacon API still serves full sidecars on request, but reconstruction may have additional latency. [verify: PeerDAS beacon API behavior under partial custody, especially for older blobs near the retention boundary]

## State

Account fields: `nonce`, `balance`, `codeHash`, `storageRoot`. Storage is per-account, mapped by 32-byte slot.

Indexers that need historical state typically use:

- `eth_getProof` (archive node, hash-based MPT) — Merkle proofs at block
- `debug_traceBlock` with `prestateTracer` — pre-tx state diffs
- `trace_replayBlockTransactions` (parity-style, erigon/nethermind/besu) — full state diffs

See [rpc-surface.md](rpc-surface.md).
