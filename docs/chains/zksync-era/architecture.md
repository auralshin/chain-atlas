# Architecture

zkSync Era is a ZK rollup. L2 state transitions are proven on L1 by a SNARK; once the proof verifies, the L2 state at that block is canonical on L1.

## Components

| Component | Role |
|---|---|
| **Sequencer** | Orders txs, executes on EraVM, produces L2 blocks |
| **Prover** | Generates ZK proofs of L2 state transitions (off-chain compute) |
| **L1 contracts** | `Mailbox` (deposits), `ExecutorFacet` (commit/prove/execute), `StateTransitionManager` (multi-chain coordination), `Bridgehub` |
| **L2 system contracts** | At addresses `0x0000…0001` – `0x0000…800B`+; provide AA, deployer, nonce holder, etc. |

Single sequencer/prover, currently operated by Matter Labs. Decentralization roadmap exists.

## Block production

| Parameter | Value |
|---|---|
| L2 block time | ~1 s {{unsourced: confirm — has been reduced over time}} |
| Sequencer | Single, Matter Labs |
| Batch (proving unit) cadence | A batch contains many L2 blocks; proven in batches of {{unsourced: typical batch size}} |
| Settlement layer | Ethereum L1 (default); some Hyperchains settle to Era itself |

L2 produces fast, cheap blocks. L1 sees compressed state diffs grouped into **batches**, with each batch going through three L1 transactions:

1. **Commit** — sequencer posts the batch's L2 state diff to L1 (calldata or blobs depending on era).
2. **Prove** — sequencer (or anyone, post-decentralization) submits a SNARK proving the batch's state transition is valid.
3. **Execute** — after a delay (allowing emergency response), the batch is "executed" on L1 — its state becomes part of the canonical L1-known L2 state.

Three L1 events per batch: `BlockCommit`, `BlocksVerification`, `BlockExecution` (names per `ExecutorFacet`).

## EraVM (not EVM-bytecode-equivalent)

Solidity sources compile via `zksolc` to **EraVM bytecode**, which is a different VM:

- **Word size**: 256-bit, like EVM.
- **Opcodes**: substantially overlap with EVM but with non-EVM additions (e.g. precompile-style ops for AA, bootloader interactions).
- **Memory model**: differs from EVM.
- **Costs**: gas costs differ; pricing is by L2-VM-specific units.

Indexer impact: **bytecode-level tools** (decompilers, bytecode comparison, signature inference from bytecode) require EraVM-aware tooling. ABI-level tools (event decoding, function call decoding) work as on EVM because the ABI is preserved by the compiler.

## Native account abstraction

There are **no EOAs in the Ethereum sense**. Every L2 account can have code that defines its own:

- `validateTransaction` — checks signature / authorization
- `executeTransaction` — actual execution
- `payForTransaction` (or paymaster) — pays gas

Default account contract is deployed at every "EOA-style" address with standard ECDSA validation. Users can replace it.

For an indexer, this means:
- **The "from" of a transaction is the account contract**, not a private-key-controlled EOA.
- **The signer might not be the account holder** — the account contract decides validity; can implement social recovery, multisig, etc.
- **`tx.from` and "who paid gas" can be different addresses** when a paymaster is used.

## Paymasters

A paymaster is a contract that pays for some user's transaction. The flow:

1. User submits a tx with `paymaster` and `paymasterInput` fields.
2. EraVM calls `paymaster.validateAndPayForPaymasterTransaction(...)` before execution.
3. Paymaster pre-pays gas (often in non-ETH tokens, often after taking ERC-20 from user).
4. Tx executes; gas is settled with paymaster, not sender.

For indexers:
- Receipt gives you `from` (sender) and `paymaster` (gas payer).
- "Who paid for this tx in ETH" = paymaster, when set.
- "User-paid amount in non-ETH" requires inspecting the paymaster's logic / events.

## System contracts (L2 predeploys)

Many. Most relevant to indexers:

| Address | Name | Purpose |
|---|---|---|
| `0x0000…0001`–`0x0000…0009` | Standard EVM precompiles (ecrecover, sha256, etc.) | Same as EVM where supported |
| `0x0000…8001` | `Bootloader` | Pseudo-account at the start of every batch; "from" of system txs |
| `0x0000…8002` | `AccountCodeStorage` | Tracks code hash per account |
| `0x0000…8003` | `NonceHolder` | Tracks both deployment and tx nonces |
| `0x0000…8004` | `KnownCodesStorage` | Codes published to L1 |
| `0x0000…8005` | `ImmutableSimulator` | Manages immutables for contracts |
| `0x0000…8006` | `ContractDeployer` | All `CREATE` / `CREATE2` go through this |
| `0x0000…8008` | `L1Messenger` | L2→L1 message emission (analog of OP's L2ToL1MessagePasser) |
| `0x0000…800A` | `L2BaseToken` | Native ETH on L2 (system token contract) |
| `0x0000…800B` | `SystemContext` | Block context: number, hash, timestamp |

Full list: [`matter-labs/era-contracts`](https://github.com/matter-labs/era-contracts).

`ContractDeployer` is the most surprising: **all contract deployments** go through this address, not via a `CREATE` opcode in user code. Indexers building "who deployed contract X" need to read `ContractDeployer.ContractDeployed` events, not infer from tx-level `to == null`.

## Where the data lives

| Data | Where to fetch | Notes |
|---|---|---|
| L2 blocks, txs, receipts, logs | `zksync-era` JSON-RPC | Standard ETH RPC + zkSync extras |
| L2 finality state (committed/proven/executed) | L1 `ExecutorFacet` events + `zks_getL1BatchDetails` | Three states per batch |
| L1→L2 deposits (priority ops) | L1 `Mailbox.NewPriorityRequest` events | Become priority txs on L2 |
| L2→L1 messages | L2 `L1Messenger` events; merkle root in `BlockExecution` event on L1 | Two-step exit |
| Validators / verifiers | L1 `StateTransitionManager` and `Bridgehub` | For Hyperchain awareness |

A complete indexer reads L1 (Mailbox, ExecutorFacet, StateTransitionManager) + L2.
