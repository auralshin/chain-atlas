# Data Model

Different from Ethereum at every level. **Read this whole file** if adapting from EVM indexer experience — assumptions transfer poorly.

## Field elements (felt252)

The primitive type. 251 bits, in a Stark-friendly prime field. Hex-encoded in JSON-RPC as `0x...` with up to 64 hex characters.

**Not a 256-bit integer.** A felt252 has a maximum value just under 2^252. Indexers using `U256` for "all fields that fit" must take care:
- Tx hashes: felt252.
- Class hashes: felt252.
- Block hashes: felt252.
- Storage keys / values: felt252.
- Account addresses: felt252.

Use a 32-byte buffer (with the high byte often zero) as a pragmatic representation; treat as `felt252` semantically.

## Block

| Field | Type | Notes |
|---|---|---|
| `block_number` | u64 | Sequential L2 block number. |
| `block_hash` | felt252 | Cryptographic block hash. |
| `parent_hash` | felt252 | Previous block's hash. |
| `new_root` | felt252 | State root after this block. |
| `timestamp` | u64 | Sequencer-set timestamp. |
| `sequencer_address` | felt252 | The sequencer's contract address. |
| `l1_da_mode` | enum | `BLOB` or `CALLDATA` (post-V0.13). |
| `l1_gas_price.price_in_wei` | u128 | L1 base fee in ETH at this block. |
| `l1_gas_price.price_in_strk` | u128 | L1 base fee in STRK. |
| `l1_data_gas_price.price_in_wei` | u128 | L1 blob data gas price in ETH. |
| `l1_data_gas_price.price_in_strk` | u128 | Same in STRK. |
| `starknet_version` | string | Protocol version (e.g. "0.13.1"). |
| `transactions` | array of TxHash | List of tx hashes in this block. |
| `status` | enum | `PENDING` / `ACCEPTED_ON_L2` / `ACCEPTED_ON_L1`. |

The sequencer chooses fee parameters at block production. **Two prices per gas type** because StarkNet supports paying fees in both ETH and STRK.

## Transaction types

Five types (post-V0.13; v3 versions of each):

| Type | Purpose |
|---|---|
| **`INVOKE`** | Standard tx: account calls one or more functions. The most common type. |
| **`DECLARE`** | Register a new class (contract code). |
| **`DEPLOY_ACCOUNT`** | Instantiate a new account contract. |
| **`L1_HANDLER`** | System-injected: L1→L2 message arrived. No signature; created by the sequencer. |
| **`DEPLOY`** (deprecated) | Pre-V0.10 contract deployment. No new ones; existing receipts may reference. |

Each type has multiple version numbers (`v0`, `v1`, `v2`, `v3`). **`v3` is the current standard** (V0.13+) and supports paying fees in either ETH or STRK.

### INVOKE v3 fields

```
{
  "type": "INVOKE",
  "version": "0x3",
  "sender_address": "0x...",
  "calldata": ["0x...", ...],          // Array of felt252
  "max_fee": "0x...",                  // Or resource_bounds in v3
  "signature": ["0x...", ...],         // AA-defined signature format
  "nonce": "0x...",
  "resource_bounds": {                 // v3-specific
    "l1_gas": { "max_amount": "0x...", "max_price_per_unit": "0x..." },
    "l2_gas": { "max_amount": "0x...", "max_price_per_unit": "0x..." },
    "l1_data_gas": { ... }             // post-V0.13
  },
  "tip": "0x...",
  "paymaster_data": ["0x..."],
  "account_deployment_data": ["0x..."],
  "nonce_data_availability_mode": "L1" | "L2",
  "fee_data_availability_mode": "L1" | "L2"
}
```

The signature is **a list of felts** — what they mean is up to the account contract's `__validate__`. Some accounts use ECDSA over Stark curve; others use SECP256r1; others use BLS; others use multisig schemes.

### L1_HANDLER

```
{
  "type": "L1_HANDLER",
  "version": "0x0",
  "nonce": "0x...",
  "contract_address": "0x...",
  "entry_point_selector": "0x...",
  "calldata": ["0x..."]
}
```

No signature. Created by the sequencer when an L1 `sendMessageToL2` arrives.

## Receipts

| Field | Notes |
|---|---|
| `transaction_hash` | felt252 |
| `actual_fee` | { amount, unit ("WEI" or "FRI" — STRK fri, the smallest unit) } |
| `execution_status` | "SUCCEEDED" or "REVERTED" |
| `revert_reason` | string (if reverted) |
| `events` | Array of Event objects |
| `messages_sent` | Array of L2→L1 messages sent in this tx |
| `execution_resources` | Cairo VM step counts, memory holes, builtin instances |
| `block_number`, `block_hash`, `transaction_index` | Standard |

`execution_resources` is a Cairo-specific block: counts of Cairo VM steps, plus per-builtin counts (Pedersen, range_check, ECDSA, etc.). Useful for fee analysis.

## Events

```
{
  "from_address": "0x...",       // Contract that emitted
  "keys": ["0x...", ...],        // Topic-equivalent (felts)
  "data": ["0x...", ...]         // Data-equivalent (array of felts)
}
```

Similar in concept to Ethereum logs but **all data is felt array**, not bytes. ABI decoding goes from felt array to typed values per the contract's ABI.

The first key (`keys[0]`) is conventionally the **event selector** — `selector!("EventName")`, similar to Ethereum's `topic0` being `keccak256("Event(...)")` but using StarkNet's selector function (Pedersen hash of the name, truncated).

## State diffs (`starknet_getStateUpdate`)

Per block, the state update returns:

```
{
  "state_diff": {
    "storage_diffs": [{ "address": "0x...", "storage_entries": [{ "key": "0x...", "value": "0x..." }] }],
    "deployed_contracts": [{ "address": "0x...", "class_hash": "0x..." }],
    "declared_classes": [{ "class_hash": "0x...", "compiled_class_hash": "0x..." }],
    "deprecated_declared_classes": ["0x..."],
    "nonces": [{ "contract_address": "0x...", "nonce": "0x..." }],
    "replaced_classes": [{ "contract_address": "0x...", "class_hash": "0x..." }]
  }
}
```

Indexers can ingest the entire L2 state by following `state_diff` per block. This is **fundamentally simpler** than Ethereum-style trie scanning.

## Classes vs contracts (the deployment model)

| Concept | Identifier | Purpose |
|---|---|---|
| **Class** | `class_hash` (felt252) | The code. Hash of the compiled (CASM) bytecode. |
| **Compiled class hash** | `compiled_class_hash` | Sierra → CASM compilation hash. Stored alongside the class. |
| **Contract** | `contract_address` (felt252) | An instance. Computed deterministically from class_hash + salt + deployer + calldata. |

To know "what contract code lives at address X":
1. Query `starknet_getClassHashAt(block, address)` → returns `class_hash`.
2. Query `starknet_getClass(block, class_hash)` → returns the actual code (Sierra + CASM).

Caching: classes are immutable per `class_hash`, so they can be cached forever; the (address → class_hash) mapping changes only on `replace_class` syscall (rare).

## L1 → L2 messages

When an L1 user calls `Starknet Core.sendMessageToL2`:

1. L1 emits `LogMessageToL2(from_l1_address, to_address, selector, payload, nonce)`.
2. The sequencer creates an `L1_HANDLER` tx on L2 with:
   - `contract_address` = `to_address`
   - `entry_point_selector` = `selector`
   - `calldata` = `[from_l1_address, ...payload]`
3. Tx executes on L2, target contract handles the message.

For an indexer correlating L1→L2: use the `nonce` field on the L1 event to match the L2 `L1_HANDLER` tx.

## L2 → L1 messages

L2 contract calls `send_message_to_l1(to_l1_address, payload)` syscall. This emits an entry in the receipt's `messages_sent` array.

After the L2 batch containing this message is proven on L1:
- L1's `Starknet Core` records the message hash.
- The L1 receiver can call `consumeMessageFromL2(...)` to process it.

Indexers track the lifecycle: emitted on L2 → batch proven on L1 → consumed on L1.

## Storage

State is a sparse Merkle tree of `(contract_address, storage_key) → value`. Different from Ethereum's MPT — Stark-friendlier hashing (Pedersen / Poseidon).

For indexers, `starknet_getStorageAt(block, address, key)` works as expected; the underlying tree is invisible.

## Genesis

StarkNet mainnet genesis was 2021-11. Genesis allocation seeded the chain with deployed predeploys (fee tokens, governance, etc.).

## Predeploys / system contracts

| Address | Name | Purpose |
|---|---|---|
| `0x049d36570d4e46f48e99674bd3fcc8463…` | ETH (fee token, ERC-20) | ETH on StarkNet, used for fees |
| `0x04718f5a0fc34cc1af16a1cdee98ffb20c31f5cd61d6ab07201858f4287c938d` {{unsourced: confirm}} | STRK (fee token, ERC-20) | STRK on StarkNet, alternative fee token |
| Various | UDC (Universal Deployer Contract) | Convenient deployment via INVOKE rather than DEPLOY_ACCOUNT |

Verify exact addresses against the StarkNet docs.
