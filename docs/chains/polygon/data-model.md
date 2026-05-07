# Data Model

EVM-equivalent at the contract level. The deltas from Ethereum are mostly in **what shows up alongside transactions** (state syncs as ghost activity) and in the **system-level events** that Bor emits.

## Block

Standard Ethereum execution block. Same fields, no Polygon-specific block-level additions in JSON-RPC output.

What does change:
- **`miner` (= block producer)** is the elected sprint producer's bor address. With sprint length 64, the same producer fills 64 consecutive blocks' `miner` field.
- **`difficulty`** is small and meaningful: bor uses a difficulty-based fork choice where the assigned producer for the current slot has higher difficulty than out-of-turn validators. Reorg analysis often references it.

No `withdrawals` array — Polygon does not implement the Shanghai withdrawals queue (that's an Ethereum L1 feature for staking withdrawals; Polygon staking is on L1 anyway).

## Transactions

Standard Ethereum types only:
- `0x00` Legacy
- `0x01` EIP-2930
- `0x02` EIP-1559 (since Polygon's Delhi/London-equivalent fork)
- `0x04` EIP-7702 (post-Pectra-equivalent — verify Polygon activation date {{unsourced}})

**No Polygon-specific transaction types.** This is a critical difference from OP/Arbitrum: there is no `Deposit` tx type at the EVM level. State syncs do not appear as transactions.

## State syncs (the ghost activity)

L1→L2 messages from `RootChainManager` arrive on bor as **synthetic state changes**, not as transactions:

- They have **no transaction hash**.
- They emit **logs** at the receiving contract (the L2 child contract for the bridged token, etc.).
- The logs appear in the receipts of the **last transaction** of the sprint-boundary block, OR are recorded in a special "system tx" pseudo-receipt depending on bor version {{unsourced: confirm current behavior}}.

To index state syncs, the canonical approach is:

1. Subscribe to L1 `StateSender.StateSynced(stateId, contractAddress, data)`.
2. Subscribe to bor `StateSender`-equivalent destination events on the receiving L2 contract.
3. Correlate by `stateId` (a monotonic heimdall-issued ID).

**Do not** try to attribute state sync effects to L2 txs by tx hash — there isn't one.

Additionally, heimdall exposes:
- `GET /clerk/event-record/<stateId>` — the queued / executed state-sync record.
- `GET /clerk/event-record/list` — list of state syncs.

A state-sync table keyed on `state_id` with columns `(state_id, l1_block, l1_tx, l2_block, l2_log_indexes, status)` is the right canonical structure.

## Logs

Standard ETH logs. No format changes.

Notable system events:

| Event | Source | Why |
|---|---|---|
| `StateSynced` | L1 `StateSender` | Each L1→L2 state sync |
| `NewHeaderBlock` | L1 `RootChain` | Each checkpoint |
| `Staked`, `UnstakeInit`, `Restaked` | L1 `StakeManager` | Validator changes |
| Heimdall events (Tendermint EVM) | Heimdall RPC | Span elections, checkpoint signing |

For most application indexers, the L2 contract logs are sufficient. Bridge-aware and validator-aware indexers need the L1 + heimdall streams.

## Receipts

Standard ETH receipt. No Polygon-specific fields.

Specifically: there is **no `l1Fee` / `l1GasUsed`** equivalent. Polygon does not pay an L1 DA fee — it's a sidechain, not a rollup.

## Predeploys / system contracts

Polygon does not have OP-style or Arbitrum-style high-address or low-address system precompiles. Instead, **regular L2 contracts** at known addresses serve as the system layer:

| Address | Contract | Purpose |
|---|---|---|
| `0x0000000000000000000000000000000000001000` | `BorValidatorSet` | Validator set state |
| `0x0000000000000000000000000000000000001001` | `StateReceiver` | Where state syncs are committed; emits the receipt-like log |
| `0x0000000000000000000000000000000000001010` | `MaticToken` (legacy) / `POLToken` (post-migration) | Native token contract |

Genesis-time state initializes these. They are upgradeable via heimdall governance proposals.

## Native token: MATIC → POL migration

In **2024-09** {{unsourced: confirm date}}, Polygon migrated the native token from MATIC to POL via a 1:1 swap. The bor-native gas token also migrated.

Indexer impact:
- **Token contract address is unchanged** at `0x...1010` — but the symbol and name in `eth_call` for `name()` / `symbol()` returned different values pre- and post-migration.
- **Off-chain MATIC/POL price feeds** are the same coin, but tickers shifted. Coingecko/CMC use POL post-migration.
- **Historical receipts paid gas in "MATIC"** at the time, "POL" post-migration. Same chain, same balances, just relabeled.

For most indexers, this is a UI / display concern, not a data concern.

## State

Same MPT as Ethereum. No state-format changes from Polygon. State trie format identical.

## Genesis

Polygon's genesis is bor block 0, dated 2020-05-30. Genesis allocation includes the system contracts at the addresses above and a substantial validator stake distribution.
