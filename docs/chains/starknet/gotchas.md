# Gotchas

StarkNet is sufficiently different from EVM that most ETH-trained indexer assumptions do not apply. Each entry below is something that breaks an EVM-style indexer.

## Felt252, not U256

Tx hashes, addresses, storage keys, etc. are all **251-bit field elements**, not 256-bit integers. Hex-encoded in JSON, the high bits are always zero — but representing them as `U256` and arithmetic on them is wrong (the field arithmetic is mod p, not mod 2^256).

For storage and serialization: 32-byte buffers are pragmatic. For semantics: treat as `felt252` (a wrapper type) with field-arithmetic operators.

## Class hash ≠ contract address

In EVM, you ask "what code is at this address" via `eth_getCode(address)`. On StarkNet, you ask:
1. `starknet_getClassHashAt(address)` → returns `class_hash`.
2. `starknet_getClass(class_hash)` → returns the actual code.

Indexers caching "code at address" must store `class_hash` separately from "code by class_hash." Class data is large (Sierra + CASM); cache aggressively because classes are immutable.

## Two-step deployment: DECLARE then DEPLOY

A contract appearing on StarkNet was deployed in **two transactions** (not necessarily by the same user). DECLARE registered the class; DEPLOY (or `DEPLOY_ACCOUNT`) instantiated. Many contracts at different addresses can share the same class.

Indexers building "this account deployed this contract" need:
- The DEPLOY tx for the address.
- The DECLARE tx for the class (if you want to track class authorship).

## No contract bytecode at the address

`starknet_getClassHashAt` is the EVM-equivalent of `eth_getCode`. Querying for "the bytecode" doesn't make sense — you query the class.

EVM indexers that scan for "deployed contracts via `tx.to == null`" find none on StarkNet because deployment doesn't use that pattern.

## Native AA: signers can be anything the account contract accepts

The signature is a **felt array** that the account's `__validate__` interprets. Common patterns:

- Single ECDSA over Stark curve: `[r, s]`.
- Multisig: `[count, sig1.r, sig1.s, sig2.r, sig2.s, ...]`.
- SECP256R1 (post-V0.13.x): different shape.
- BLS / aggregate: per-account format.

Indexers cannot assume a signature format. Capture the signature as raw felts; let the account contract's ABI guide interpretation if you need to attribute to a specific signer.

## L2 → L1 messages have a separate L1 consume step

The user's funds move when L1 `consumeMessageFromL2` is called, **not** when the L2 batch is proven. An indexer's "your withdrawal" UI must distinguish:
- Emitted on L2 (in receipt's `messages_sent`).
- Batch proven on L1 (consumable but not yet consumed).
- Consumed on L1 (funds released).

## `event.keys` contains the selector AND optional indexed-param-style data

In Ethereum, `topic0 = keccak256(event_signature)`, and `topic1`, `topic2`, `topic3` are indexed parameters. On StarkNet, `keys[0]` is conventionally the event selector (Pedersen-hash-derived from the name), and `keys[1...]` are additional values determined by the contract's ABI — **but the ABI controls this**, not a protocol convention.

This means:
- Some contracts put indexed parameters in `keys[1]+`.
- Some contracts put nothing in `keys` after `keys[0]`, and put all data in `data[]`.
- The ABI tells you which.

Indexers cannot blindly map `keys[1] → first indexed param` without knowing the contract's ABI.

## STRK fees vs ETH fees

A V0.13+ tx pays in either ETH (WEI) or STRK (FRI). Receipts include the unit. **Don't sum across units.** If you need a "USD fee" total, convert each tx's fee using its block's exchange rates.

Pre-V0.13: only ETH. Post-V0.13: both. The `actual_fee.unit` field tells you which.

## Resource bounds, not gas

V3 transactions specify resource bounds:
- `l1_gas` — bounds for L1 gas (proving costs).
- `l2_gas` — bounds for Cairo VM steps + builtin invocations.
- `l1_data_gas` — bounds for L1 blob data (post-V0.13.2).

Each has a `max_amount` and `max_price_per_unit`. Indexers tracking "gas used" must distinguish these resources — they do not collapse into a single number.

## Cairo 0 vs Cairo 1 classes have different ABI shapes

`starknet_getClass(class_hash)` returns either:
- Cairo 0: `{ "program": ..., "entry_points_by_type": ..., "abi": [...] }` with Cairo 0 ABI format.
- Cairo 1: `{ "sierra_program": ..., "entry_points_by_type": ..., "abi": "..." (string), "contract_class_version": "..." }` with Cairo 1 ABI format.

ABI parsers must handle both. Function selector hashing differs in some edge cases.

## State updates are the canonical way to track state changes

Unlike Ethereum where you reconstruct state from txs + traces, on StarkNet every block has a complete `state_diff` exposed via `starknet_getStateUpdate`. **Use it.** It's much simpler than scraping txs.

The state diff contains: storage changes, deployed contracts, declared classes, nonce updates, replaced classes (rare).

## Pending block has different block_hash semantics

A pending block has `block_hash = null` and `block_number = null` until sealed. A tx in the pending block has the same — `block_hash`, `block_number`, `transaction_index` are all null until inclusion in a sealed block.

Indexers must handle pending data with explicit nullability; don't expect the same field set as accepted blocks.

## Trace output format is Cairo-specific

`starknet_traceTransaction` returns a structured object with `validate_invocation`, `execute_invocation`, `fee_transfer_invocation` phases — different from geth's `debug_traceTransaction` output. There's no easy reuse of EVM-style trace tooling.

For indexers building internal-tx-equivalent analytics, you'll write StarkNet-specific decoders.

## L1 messaging uses `Starknet Core Contract`, not OP/Arbitrum-style addresses

The L1 contract orchestrating StarkNet on Ethereum is `Starknet Core Contract` at `0xc662c410C0ECf747543f5bA90660f6ABeBD9C8c4` {{unsourced: confirm address}}. Different from any OP Stack or Arbitrum Inbox address. Don't copy-paste from EVM rollup indexers.

## RPC spec version compatibility

A client supporting RPC spec v0.7 may not be compatible with v0.5 indexer code. Field names, response shapes, and method signatures change between major spec versions. Always pin spec version at startup and ensure your client library matches.
