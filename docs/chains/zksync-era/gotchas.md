# Gotchas

zkSync Era breaks more ETH-trained indexer assumptions than any other EVM-adjacent chain. Each of these is a real failure mode.

## "From" is the account contract; signer might be different; gas payer might be different again

Three distinct identities per transaction:

| Concept | What it is |
|---|---|
| `tx.from` | The account contract address (the "sender") |
| Signer | Whoever produced the signature; the account contract validates it |
| Gas payer | The paymaster, if set; else `tx.from` |

ETH-style indexers conflate all three under "the from address." On Era, you must track each:

```
attribution.user = tx.from
attribution.signer = recover_signer_via_account_contract(tx)  // if you care
attribution.gas_payer = tx.paymaster if tx.paymaster else tx.from
```

**Fix:** every tx-level attribution row in your DB needs explicit columns for these three.

## `eth_getTransactionReceipt` does not include the paymaster

The standard receipt has no `paymaster` field. To get it, use `zks_getTransactionDetails`. Indexers that ingest only `eth_*` results miss paymaster attribution entirely.

**Fix:** ingest via `zks_getTransactionDetails` for every Era tx.

## Type 0x71 deserialization breaks naive ETH parsers

A type-0x71 tx has fields not present in any ETH tx type: `gasPerPubdataByteLimit`, `factoryDeps`, `paymasterParams`, `customSignature`. Naive RLP parsers (or `alloy`'s standard tx envelope) deserialize the type byte and fail or silently drop fields.

**Fix:** use `zksync-web3`-derived types (`zksync-types` Rust crate {{unsourced: confirm}}) or hand-roll a struct that matches the EIP-712 schema. Don't assume any of the standard ETH crates handle 0x71 by default.

## `to == null` is **not** the contract creation pattern

On Era, contract creation goes through `ContractDeployer` (`0x0000…8006`). The user submits a tx **`to = ContractDeployer`**, with calldata that selects `create`/`create2`.

Indexers that detect deployments by `tx.to == null` find none on Era and conclude there are no contracts.

**Fix:** subscribe to `ContractDeployed` events emitted by `ContractDeployer`.

## EraVM bytecode is not EVM bytecode

`eth_getCode` returns EraVM bytecode, not EVM bytecode. Tools that:
- Decompile EVM (e.g. heimdall-rs decompiler).
- Match function selectors via 4-byte hash → bytecode pattern.
- Run static analysis (Slither, Mythril patterns adapted for bytecode).

…all fail or produce garbage on EraVM bytecode.

**Fix:** ABI-level analysis works fine (event decoding, function call decoding via known signatures). Bytecode-level requires `zksolc`-aware tooling.

## Solidity opcodes can behave differently

Some EVM opcodes have different semantics in EraVM. Notable {{unsourced: confirm exact list}}:
- `SELFDESTRUCT` — disabled / no-op on EraVM.
- `BLOCKHASH` — only retains a small recent window.
- `MSTORE8`, some memory ops — different gas costs.

Indexers that simulate execution off-chain to predict outcomes will diverge from on-chain reality. **Use the actual node for any execution simulation, not an EVM library.**

## System tx attribution

System operations (refunds, paymaster pre-charges, batch bookkeeping) appear in traces as calls **from `Bootloader`** (`0x0000…8001`). A user tx that the user "wrote" may produce a dozen frames of system activity around the user-facing call.

Indexers building "what did this user do" call trees must filter system frames. Specifically: skip frames where `from = 0x0000…8001` unless you're interested in protocol-level activity.

## Native AA delegate-call patterns

Some account contracts implement `validateTransaction` via `delegatecall`. The validating logic may live at a different code address than the account itself. For an indexer doing static analysis on accounts, this means:
- The account at address X may not contain the validation logic.
- Different validation rules apply per account.
- Multisig / social recovery accounts have different signers per tx.

**Fix:** if you care about who *can* sign for an account, you cannot determine it without dynamic execution. Capture the actual signer per-tx (via the account's emitted events or the validation context) at indexing time.

## Paymaster-paid amount is in tokens, not ETH (sometimes)

Token paymasters take ERC-20 from the user upfront and pay gas in ETH. The "what did the user pay" answer is the ERC-20 amount, not the ETH gas cost.

Indexers building user-facing fee dashboards must inspect the paymaster's `Transfer` events on the user's ERC-20 balance, not the receipt's gas fields.

## Two nonces per account

`NonceHolder` distinguishes:
- **Deployment nonce** (incremented on `CREATE` / `CREATE2`).
- **Tx nonce** (incremented on each tx the account sends).

Indexers tracking "next nonce" for an account need both. `eth_getTransactionCount` returns the tx nonce; `zks_*` helpers return others. Standard ETH wallets / SDKs sometimes get this wrong on Era.

## L2 timestamp can drift from L1

Like other rollups, Era's L2 timestamp is set by the sequencer and can drift from L1 timestamp (within bounds). Don't join L2 events to L1 events on `timestamp == timestamp`.

## The "default account" address has bytecode

On Ethereum, an EOA's `eth_getCode` returns `0x` (empty). On Era, **every** account that has ever transacted has the `DefaultAccount` bytecode (or a custom contract). `eth_getCode` returns non-empty for "EOA-equivalent" addresses.

Indexers using `eth_getCode` to classify "is this an EOA or contract" misclassify everything as a contract on Era. **There are no EOAs to classify.**

## Withdrawal claim flow is two-step on L1

1. L2 burn (`L1Messenger` event).
2. Wait for batch to reach `executed` state on L1.
3. User calls `L1Bridge.finalizeWithdrawal` (or chain-specific equivalent) on L1.

Funds are released at step 3. An indexer's "your withdrawal" UI must show the in-flight stage between L2 burn and L1 finalization.

## Hyperchain settlement may not be Ethereum L1

Some ZK Stack chains settle to Era itself (or another Hyperchain) rather than to Ethereum. For those chains, "L1" in the indexer's mental model is **Era L1**, not Ethereum. Confusing addresses and contracts when you switch indexers between chains.

**Fix:** for any ZK Stack chain you index, query its `Bridgehub` registration to identify the actual settlement layer.
