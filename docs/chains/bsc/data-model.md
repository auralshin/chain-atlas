# Data Model

EVM-equivalent. Standard ETH JSON-RPC, standard tx types, standard receipts. The deltas are at the consensus / system-contract layer, not the EVM data model.

## Block

Standard Ethereum execution block. Notable fields with BSC-specific use:

| Field | Notes |
|---|---|
| `extraData` | Encodes validator set rotation at epoch boundaries (block N where N % 200 == 0). Pre-Plato: list of validator addresses. Post-Plato: validator addresses + BLS public keys + (in some blocks) attestation aggregates. |
| `miner` | Block proposer. Validators take turns; `miner` cycles through the active 21 over each ~21-block window. |
| `difficulty` | Either 1 (in-turn proposer) or 2 (out-of-turn). Used by Parlia fork choice — **not** PoW difficulty. |

No `withdrawals` array — BSC does not implement Shanghai withdrawals (no L1 staking queue analog).

## Transactions

Standard Ethereum types only:
- `0x00` Legacy
- `0x01` EIP-2930
- `0x02` EIP-1559 (post-London-equivalent fork)
- `0x04` [EIP-7702](https://eips.ethereum.org/EIPS/eip-7702) `SET_CODE_TX_TYPE` — activated on BSC at the Pascal/Prague fork (2025-03-20 02:10:00 UTC, see [BSCChainConfig](https://github.com/bnb-chain/bsc/blob/master/params/config.go))

**No BSC-specific transaction types.** No system tx, no deposit tx.

## Receipts

Standard Ethereum receipts. No BSC-specific fields.

There is **no L1-fee field** — BSC is its own L1, no DA cost to attribute.

## Logs

Standard ETH logs. Format unchanged.

Notable system events:

| Event | Source | Why |
|---|---|---|
| `validatorSetUpdated` | `0x0000…1000` ValidatorContract | Active validator changes per epoch |
| `validatorSlashed` | `0x0000…1001` SlashContract | Slashing events |
| Stake / unbonding events | `0x0000…2000` StakeHub (post-fusion) | Native staking activity |
| `crossChainPackage`-related | `0x0000…1003`–`0x0000…1007` (pre-fusion) | Beacon-chain cross-chain — historical |

## System contracts ("genesis contracts")

Same set listed in [architecture.md](architecture.md). They are upgradeable via cross-validator consensus — bytecode changes happen at hard-fork-style events, not at every epoch.

| Indexer impact | Action |
|---|---|
| Validator set changes | Subscribe to `ValidatorContract` events, OR parse epoch-boundary header extra-data. |
| Slashing events | Subscribe to `SlashContract` events. Useful for chain-health monitoring. |
| Reward distribution | Subscribe to `SystemRewardContract` events. Useful for revenue dashboards. |
| Native staking (post-fusion) | Subscribe to `StakeHub` events. Replaces all the historical Beacon-chain staking flow. |

## Cross-chain (deprecated post-2024)

Pre-fusion, BSC and BC exchanged messages via:
- `TokenHub` for token transfers
- `RelayerHub` for relayer registration
- `GovHub` for governance proposals
- `LightClient` (BC light client on BSC)

These contracts still exist in the on-chain state but are **inactive** post-fusion. Historical events from 2020–2024 are present in their event logs but new events stopped after the fusion completion block {{unsourced: confirm specific block}}.

Indexers that scan from genesis encounter substantial cross-chain activity in this period — most of it not relevant to current applications. Decide upfront whether to backfill or skip.

## Native token: BNB

Native gas token on BSC. Historically the same BNB as on Beacon Chain (1:1 with cross-chain bridging). Post-fusion, all BNB lives on BSC.

There is **no `BNBToken` ERC-20 contract** for the native asset. BNB-as-ERC-20 is `WBNB` at `0xbb4CdB9CBd36B01bD1cBaEBF2De08d9173bc095c` — that's a wrapper, not the native token.

## Block timestamps

Pre-Lorentz: ~3-second resolution. Post-Lorentz: sub-3-second; the `timestamp` field can have higher precision {{unsourced: confirm precision}}.

Indexers that hardcode 3-second block intervals for "blocks per hour" estimates are wrong post-Lorentz. **Compute from actual timestamps**, never from a constant.

## State

Same MPT as Ethereum. State trie format identical. The system contracts at `0x0000…10xx` and `0x0000…2000` exist in genesis-allocated state.

## Genesis

BSC genesis block date: 2020-09-01. Genesis allocation includes:
- The system contracts.
- Initial validator stake distribution.
- Beacon-chain bridge contracts (legacy).

For indexers, genesis is the natural sync starting point. There is no pre-genesis era to handle.
