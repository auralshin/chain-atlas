# Architecture

ZK rollup on Ethereum, with a Cairo VM execution layer. Validity proofs are STARKs (not SNARKs); the proof system is StarkWare's STONE / Stwo prover.

## Components

| Component | Role |
|---|---|
| **Sequencer** | Orders txs, executes Cairo, produces L2 blocks. Currently closed-source StarkWare-operated. |
| **Prover** | Generates STARK proofs over Cairo execution traces. |
| **Full nodes** | `pathfinder` / `juno` / `papyrus` — replicate the chain, serve RPC, do not produce blocks. |
| **L1 contracts** | `Starknet Core Contract` (state commitments, L1↔L2 messaging), L1 token bridges, fact registry. |

The sequencer + prover are closed-source. Full nodes are open-source. **Indexers run a full node**, not a sequencer.

## Block production

| Parameter | Value |
|---|---|
| L2 block time | Variable, ~30 s historically; decreasing with V0.13+ {{unsourced: confirm current value}} |
| Sequencer | Single, StarkWare |
| Settlement layer | Ethereum L1 |
| Proof cadence | Every few hours; batched proofs cover many L2 blocks |

The sequencer publishes L2 blocks at variable intervals (depends on block-fullness and protocol parameters). Once enough L2 blocks have been produced, the prover generates a STARK proof and the sequencer submits it to L1's `Starknet Core Contract`.

## Cairo VM

Stack-based, **not** EVM-shaped. Key differences:

- **Field elements (felt252)**: native data type is a 251-bit prime field element, not 256-bit integer. All arithmetic is mod p (a Stark-friendly prime).
- **Storage**: key-value store keyed on felt252 → felt252.
- **Memory model**: read-once memory; the trace is fundamentally different from EVM.
- **Calling convention**: explicit `entry_point` selectors (computed from function name hash); calldata is an array of felts.
- **No `msg.sender`-equivalent for cross-contract calls** — instead, the calling contract address is passed explicitly via syscalls.

For an indexer, you don't need to understand Cairo execution semantics — the RPC abstracts state queries. But you do need to understand **what the data shapes are**.

## Cairo languages: 0 vs 1/2

- **Cairo 0** (pre-2023): the original Cairo. Contracts compiled to a flat assembly-like format.
- **Cairo 1 / Cairo 2** (2023+): newer Rust-influenced syntax. Compiles to **Sierra** (Safe Intermediate Representation) and then to **CASM** (Cairo Assembly).

On-chain:
- A contract's **class** is the code. Identified by `class_hash` (felt252 hash of the Sierra/CASM bytecode).
- A contract's **address** is where it lives. Computed from `class_hash`, salt, deployer address, calldata.
- Cairo 0 classes can no longer be `DECLARE`d on mainnet (post-V0.11+ {{unsourced: confirm}}). Existing Cairo 0 contracts continue to work.

## Two-step deployment: DECLARE + DEPLOY

Unlike EVM's atomic `CREATE` / `CREATE2`, StarkNet deployments are **two transactions**:

1. **DECLARE**: registers the contract class (the code) on-chain. Returns a `class_hash`.
2. **DEPLOY** (or `DEPLOY_ACCOUNT` for accounts): instantiates a contract at a deterministic address using a previously-DECLAREd `class_hash`.

A class only needs to be DECLAREd once across the entire chain — different contracts can DEPLOY against the same class. Indexers tracking "what contract code does this address have" must look up the class via `class_hash`, not store bytecode at the address.

## Native account abstraction

There are **no EOAs**. Every account is a contract that defines:
- `__validate__` — accept or reject an incoming tx (signature check).
- `__execute__` — execute the tx's calls.
- `__validate_declare__` and `__validate_deploy__` — for DECLARE / DEPLOY_ACCOUNT txs originating from this account.

The user "submits" a tx by sending it to the StarkNet sequencer; the sequencer routes it through the account's `__validate__` then `__execute__`.

For indexers:
- **Tx `from`** is the account contract address.
- **Signers** are determined by the account contract's `__validate__` logic — could be one ECDSA key, multiple keys, multisig, social recovery, etc.
- **No standard signature format** — each account contract defines its own.

## L1 ↔ L2 messaging

### L1 → L2

User calls `Starknet Core.sendMessageToL2` on Ethereum L1. The message is queued; the sequencer picks it up and converts it to an `L1_HANDLER` tx on L2.

`L1_HANDLER` is a special tx type that has no signature (system-injected) and targets a specific entry point on the destination contract.

### L2 → L1

L2 contract emits a message via the `send_message_to_l1` syscall. After the L2 batch containing this message is proven on L1, the user can call `Starknet Core.consumeMessageFromL2` to consume the message and trigger any L1 effect.

Two-step: L2 emit + L1 consume. The L1 consume is **after** L1 settlement of the L2 batch.

## Three-stage finality (block status)

| Status | Means |
|---|---|
| `PENDING` | In the pending block being built; not yet sealed |
| `ACCEPTED_ON_L2` | Sealed in an L2 block by the sequencer |
| `ACCEPTED_ON_L1` | Proof verified on L1; canonical settled state |

These are explicit fields in the block / tx response, unlike Ethereum's implicit confirmation depth.

## Where the data lives

| Data | Where to fetch | Notes |
|---|---|---|
| L2 blocks, txs, receipts, events | StarkNet full node JSON-RPC | `starknet_*` namespace |
| L2 state diffs | Same | `starknet_getStateUpdate` |
| L1 settlement | L1 `Starknet Core Contract` events | `LogStateUpdate`, `LogStateTransitionFact` |
| L1 → L2 messages | L1 `Starknet Core Contract` events | `LogMessageToL2` |
| L2 → L1 messages | L2 events emitted by sender contracts; L1 consumption logged on L1 | Two-side tracking |
| Class declarations | L2 blocks with DECLARE txs | Plus `starknet_getClass` to fetch the code |

A complete indexer reads L1 (Starknet Core Contract) + L2 (full node RPC).
