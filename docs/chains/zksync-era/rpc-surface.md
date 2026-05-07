# RPC Surface

Era exposes standard `eth_*` plus a `zks_*` namespace for zkSync-specific data (batch status, paymaster info, EIP-712 helpers).

## Standard ETH JSON-RPC

All `eth_*` methods work, with caveats:

| Method | Notes |
|---|---|
| `eth_getBlockByNumber` | Returns the L2 block. **Does not** include batch info — use `zks_getBlockDetails` for that. |
| `eth_getTransactionReceipt` | Standard fields. **Does not** include `paymaster`, `l1BatchNumber`, etc. — use `zks_getTransactionDetails` for full receipt. |
| `eth_call` / `eth_estimateGas` | Estimates **include** the L1 DA cost component (analogous to OP receipts but baked into estimation). |
| `eth_getBlockByNumber("safe"/"finalized")` | Maps to L1-committed / L1-executed batch states {{unsourced: confirm exact mapping}}. |
| `eth_subscribe("newHeads")` | Standard. |

## zkSync-specific (`zks_*`)

| Method | Returns | Why |
|---|---|---|
| `zks_getBlockDetails` | L2 block + L1 batch number, timestamps, status | Get batch correlation for a block |
| `zks_getTransactionDetails` | Tx + paymaster, batch info, fee data | Full tx info including AA fields |
| `zks_getL1BatchDetails` | Batch's commit/prove/execute L1 events | Track finality stage per batch |
| `zks_getL1BatchBlockRange` | First and last L2 block of a batch | For batch-aware analytics |
| `zks_getProof` | Storage proof for a slot at a given block | For light-client / bridge proofs |
| `zks_getMainContract` | Address of the main `DiamondProxy` on L1 | L1 contract resolution |
| `zks_getBridgeContracts` | Bridge contract addresses | Bridge tracking |
| `zks_getTestnetPaymaster` | Testnet helper | Dev only |
| `zks_getAllAccountBalances` | All token balances for an account | Includes ETH and ERC-20 |
| `zks_estimateFee` | Fee estimate including paymaster considerations | EIP-712 tx estimation |

`zks_getTransactionDetails` is the canonical way to read tx data. Use it instead of `eth_getTransactionReceipt` for indexing — it gives you `paymaster`, `l1BatchNumber`, and the full fee breakdown.

## Trace APIs

`zksync-era` supports a subset of trace methods:

| Method | Available |
|---|---|
| `debug_traceTransaction` | Yes ({{unsourced: confirm tracer support}}) |
| `debug_traceBlockByNumber` | Yes |
| `debug_traceCall` | Yes |
| Erigon-style `trace_*` | No (single client) |

EraVM execution is traced by the Era client. Trace output shape differs from EVM in that **system contract calls (Bootloader, ContractDeployer, etc.) appear in traces** — a single user tx produces multiple "from Bootloader to X" frames. Indexers building "user-attributed call trees" must filter system frames.

## Archive node requirements

Era archive: full L2 historical state for `eth_call` / `debug_*` at any block. Disk size growing fast with traffic.

Most production indexers use Matter Labs' RPC or Alchemy/QuickNode's Era endpoints. Self-hosting is possible but Era's archive disk requirements are substantial.

## Public provider quirks

- **Block range limits** on `eth_getLogs`: typical 1k–10k L2 blocks.
- **`zks_*` methods are provider-dependent**: Matter Labs's RPC has full support; some third-party providers only expose `eth_*`. Test before depending.
- **EIP-712 tx submission**: requires properly-typed `eth_sendRawTransaction` with the EIP-712 domain. Some libraries don't handle this well.

## Picking transports

| Transport | Use when |
|---|---|
| HTTP | Bulk historical scans, batch correlation queries |
| WebSocket | Real-time `newHeads` + `logs` |
| IPC (local) | Self-hosted; uncommon for Era |

Most indexers run against a hosted provider for Era due to the operational cost of self-hosting an Era archive node.
