# Data Model

## Block

Standard Ethereum-shape JSON-RPC block, plus zkSync-specific fields exposed via `zks_*` methods:

| Field | Source | Meaning |
|---|---|---|
| Standard ETH header fields | `eth_getBlockByNumber` | Same as ETH. |
| `l1BatchNumber` | `zks_getBlockDetails` | The L1 batch this L2 block belongs to. |
| `l1BatchTimestamp` | `zks_getBlockDetails` | When the batch was committed. |

Multiple L2 blocks compose a single L1 batch. The batch is the unit of L1 commit/prove/execute.

## Transaction types

| Type | Hex | Origin | Notes |
|---|---|---|---|
| Legacy / EIP-2930 / EIP-1559 | `0x00` / `0x01` / `0x02` | User | Supported. Uncommon in practice. |
| **EIP-712 zkSync** | `0x71` (113) | User | **The default for zkSync wallets.** Carries paymaster data, custom nonces, AA validation hooks. |
| **Priority op** | `0xff` (255) | System (L1-originated) | Created when an L1 deposit message arrives. Equivalent of OP's deposit tx. |

### Type 0x71 (EIP-712) — the dominant case

Fields:

```
{
  "type": "0x71",
  "from": <sender account contract address>,
  "to": <target>,
  "value": <wei>,
  "data": <calldata>,
  "gas": <gas limit>,
  "maxFeePerGas": <wei>,
  "maxPriorityFeePerGas": <wei>,
  "nonce": <NonceHolder-managed nonce>,
  "chainId": <chain id>,
  // zkSync-specific:
  "gasPerPubdataByteLimit": <gas per pubdata byte>,
  "factoryDeps": [...],         // Bytecodes referenced by CREATE2 / CREATE
  "paymasterParams": {
    "paymaster": <address or 0>,
    "paymasterInput": <bytes>
  },
  "customSignature": <bytes>    // For non-ECDSA AA
}
```

**The `from` is the sender account contract address.** **The signer** is whoever the contract's `validateTransaction` accepts. **The gas payer** is `paymaster` if set, else `from`.

Three different identities. Indexers need to track all three for accurate attribution.

### Priority op (0xFF)

Originates from L1 deposits via `Mailbox.requestL2Transaction`. Becomes a tx on L2 with type 0xFF. Has no signature; identified by `from = ` aliased L1 sender.

## Receipts

Standard ETH receipt **plus** `zksync-era`-specific fields:

| Field | Type | Meaning |
|---|---|---|
| `l1BatchNumber` | u64 | Which L1 batch this tx is in |
| `l1BatchTxIndex` | u64 | Tx position within the batch |
| `l2ToL1Logs` | array | L2→L1 messages emitted by this tx (as observed by `L1Messenger`) |

Available via `zks_getTransactionDetails` (richer than standard `eth_getTransactionReceipt`).

## Logs

Standard ETH `Log` shape. zkSync-specific events live at the system contracts:

| Event | Source | Why |
|---|---|---|
| `ContractDeployed` | `0x0000…8006` ContractDeployer | Every contract deployment |
| `L2ToL1LogSent` (or similar) | `0x0000…8008` L1Messenger | Each L2→L1 message |
| `BlockCommit`, `BlocksVerification`, `BlockExecution` | L1 `ExecutorFacet` | Batch lifecycle on L1 |
| `NewPriorityRequest` | L1 `Mailbox` | L1→L2 deposits / priority ops |

`ContractDeployer` events are how an indexer enumerates deployments. **Do not** rely on `tx.to == null` — on zkSync, deployments don't use that pattern.

## Account abstraction at the data-model level

Every account address has:
- A code hash (in `AccountCodeStorage`).
- A nonce in `NonceHolder` (which separates **deployment nonce** from **tx nonce**).

EOA-style accounts use the `DefaultAccount` system contract code (ECDSA validation). Smart accounts deploy custom code.

For indexers:
- **Account creation**: emitted by `ContractDeployer.ContractDeployed`.
- **Account code change** (e.g. via `setBytecodeHash`): emitted by `AccountCodeStorage` events.
- **Nonces**: query `NonceHolder.getMinNonce(account)` for the current nonce; deploys and txs each have their own counter.

## Paymaster data

Paymaster usage is visible at the receipt level only via `tx.paymaster` field; the paymaster's internal logic (e.g. ERC-20 fee taking) is in its own events.

Common paymaster patterns:
1. **Token paymaster**: takes ERC-20 from user upfront; pays gas in ETH; emits `Transfer(user, paymaster, amount)` for the ERC-20.
2. **Sponsorship paymaster**: a dApp sponsors users' gas; no payment from user.
3. **Subscription paymaster**: validated against an off-chain authorization.

Indexers building "user-paid amount in $TOKEN" must inspect paymaster events, not just the tx-level paymaster field.

## Native ETH on L2

Tracked by `L2BaseToken` (`0x0000…800A`) — the native ETH balance is **a system contract's accounting**, not a chain-level balance in the EVM sense. `eth_getBalance` works as expected (queries L2BaseToken under the hood) but bytecode-level analysis must know about this.

Deposits credit ETH via the priority op (0xFF) tx flow. Withdrawals burn via L1Messenger.

## State

State stored in a Merkle Patricia Trie variant tuned for SNARK-friendliness — it's not the same MPT as Ethereum. Indexer impact is mostly nil (RPC abstracts state queries) but bytecode-level state proving differs.

## Genesis

zkSync Era's L2 genesis is at L2 batch number 0, dated 2023-03-24. Genesis allocates:
- The system contracts at the addresses above.
- Initial bridged ETH balances if any.

There is no pre-Era era to worry about; zkSync Lite (1.0) is a separate chain entirely.
