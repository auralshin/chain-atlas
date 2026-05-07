# Data Model

Post-Bedrock OP Stack. ≈ 95% identical to Ethereum's data model — this file documents the deltas.

## Block

Same fields as an Ethereum execution block, **plus**:

| Field | Type | Meaning |
|---|---|---|
| `mixHash` | bytes32 | On L2 this is `0` (no PoW, no RANDAO from L2 consensus). Some clients alias it to `prevRandao` in JSON. |
| L1 origin | derived | Not a header field, but **every L2 block has a unique L1 origin block**. Reachable via `op-node` `optimism_outputAtBlock` or by replaying derivation. Required to attribute L1-fee inputs. |

### What's not here

- No `withdrawals` array on L2 blocks. (L2 withdrawals are `L2ToL1MessagePasser` events, see below.)
- No beacon root field — but `parentBeaconBlockRoot` from the L1 origin **is** exposed via the predeploy at `0x4200000000000000000000000000000000000015` after Ecotone.

## Transaction types

| Type | Hex | Origin | Notes |
|---|---|---|---|
| `Legacy` | `0x00` | User | Standard pre-EIP-1559. |
| `EIP-2930` | `0x01` | User | Access lists. |
| `EIP-1559` | `0x02` | User | Default for most wallets. |
| `EIP-4844` | `0x03` | **Not on L2.** | Blobs are an L1-only construct in OP Stack. |
| **`Deposit`** | `0x7E` | **System** | OP Stack-specific. See below. |
| `EIP-7702` | `0x04` | User | After Isthmus. {{unsourced: confirm}} |

### Deposit transaction (0x7E)

The single most important addition to the data model. Created when:
- A user calls `OptimismPortal.depositTransaction()` on L1 → produces a **user deposit** on L2.
- A new L1 epoch starts → produces a **system deposit** (the `L1Block.setL1BlockValues()` call as the first tx of every L2 block).

```
DepositTx {
  source_hash:    bytes32,    // domain-separated hash of L1 tx + log index
  from:           Address,    // L1 sender, ALIASED for contracts
  to:             Option<Address>,
  mint:           U256,       // ETH minted on L2 (deposit ETH)
  value:          U256,
  gas_limit:      U64,
  is_system_tx:   bool,       // first tx of every L2 block when true
  data:           Bytes,
}
```

**No signature.** Indexers that validate signatures for non-deposit txs must skip this for type 0x7E. **The `from` is L1-aliased for contracts** (adds `0x1111000000000000000000000000000000001111` to the L1 address). Many indexer bugs trace to this — see [gotchas.md](gotchas.md).

The first deposit in each L2 block is the system tx that sets the L1 block context (number, hash, base fee, blob base fee, sequence number). All subsequent txs in the block can read these via the `L1Block` predeploy.

## Receipts

Standard ETH receipt **plus**:

| Field | Type | Meaning |
|---|---|---|
| `l1Fee` | U256 | Fee paid (in L2 ETH) to cover L1 DA cost for this tx. |
| `l1GasUsed` | U256 | Approximate L1 gas the tx's data would consume. |
| `l1GasPrice` | U256 | L1 base fee at the L1 origin. |
| `l1FeeScalar` | string (legacy) / { `l1BaseFeeScalar`, `l1BlobBaseFeeScalar` } (Ecotone+) | Multiplier(s). |
| `l1BlobBaseFee` | U256 (Ecotone+) | L1 blob base fee. |

**The L1-fee formula changes per fork.** Pre-Ecotone, post-Ecotone, post-Fjord (FastLZ-estimated size) all differ. See [forks-changelog.md](forks-changelog.md). Indexers that recompute L1 fees must version the formula by block.

Deposit transactions have `l1Fee = 0` (no L1 cost — the user paid for the L1 inclusion separately). Be defensive when computing fee revenue.

## Logs

Standard ETH logs. Topics and data are unchanged.

Notable system events to index:

| Event | Source | Why it matters |
|---|---|---|
| `TransactionDeposited` | L1 `OptimismPortal` | Each user deposit. Match to L2 deposit tx by `source_hash`. |
| `MessagePassed` | L2 `0x4200…0016` (`L2ToL1MessagePasser`) | Each withdrawal initiation. |
| `OutputProposed` | L1 `L2OutputOracle` (pre-fault-proofs) | Output roots posted. |
| `DisputeGameCreated` | L1 `DisputeGameFactory` (post) | Output roots posted. |
| `WithdrawalProven`, `WithdrawalFinalized` | L1 `OptimismPortal` | Bridge accounting. |
| `ConfigUpdate` | L1 `SystemConfig` | Sequencer changes, gas config changes. **Must index** to track upgrade activations. |

## Predeploys (system contracts at fixed L2 addresses)

All in the `0x4200000000000000000000000000000000000000`–`0x42000000000000000000000000000000000000FF` range.

| Address | Name | Purpose |
|---|---|---|
| `0x42…0002` | `DeployerWhitelist` | Legacy, deprecated post-Bedrock. |
| `0x42…0006` | `WETH9` | OP Mainnet's wrapped ETH. |
| `0x42…0007` | `L2CrossDomainMessenger` | Cross-chain message inbox/outbox. |
| `0x42…000F` | `GasPriceOracle` | L1 fee parameters readable from L2. |
| `0x42…0010` | `L2StandardBridge` | Native bridge for ERC-20s. |
| `0x42…0011` | `SequencerFeeVault` | Where sequencer fees accumulate. |
| `0x42…0014` | `L1Block` | L1 context (number, hash, base fee, blob base fee). Set by system deposit tx. |
| `0x42…0015` | `L1BlockNumber` (legacy) | Older accessor. |
| `0x42…0016` | `L2ToL1MessagePasser` | Where withdrawals are recorded. |
| `0x42…001A` | `BaseFeeVault` | Where L2 base fee revenue accumulates. |
| `0x42…001B` | `L1FeeVault` | Where L1 fee revenue accumulates. |

Full list: [optimism/specs predeploys](https://specs.optimism.io/protocol/predeploys.html). Indexers that decode known contracts should include these by default.

## State

Same MPT as Ethereum. State trie format identical. The only difference is the set of accounts that exist at genesis (predeploys) and the system addresses that the OP Stack reserves for deposits/system txs.
