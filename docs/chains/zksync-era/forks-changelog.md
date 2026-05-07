# Forks and Changelog

zkSync Era versions its **protocol version** (incremented at upgrades) and its **proof system** (Plonkish → Boojum, with Boojum further upgraded over time).

## Protocol versions

Each protocol version corresponds to an L1 contract upgrade and L2 system contract upgrade. Indexer-relevant ones:

| Protocol version | Approx date | Indexer impact |
|---|---|---|
| **v1**–**v15** {{unsourced: era-pre-Boojum versions}} | 2023-03 → 2023-Q4 | Initial Era; original (non-Boojum) proof system. |
| **v16** {{unsourced}} | 2023-Q4 / 2024-Q1 | **Boojum upgrade.** New proof system. **Indexer impact: none at the L2 RPC level**, but L1 verifier contract changed; indexers tracking proof-verification events need updated decoders. |
| **v18**–**v22** {{unsourced}} | 2024 | Successive improvements; minor system contract updates. |
| **v23** {{unsourced: 2024}} | 2024 | Bridgehub introduced — new L1 contract architecture for ZK Stack chains. **Indexers must update L1 contract addresses.** |
| **v24** {{unsourced: 2024-2025}} | 2024-2025 | Cancun-equivalent EIPs supported on EraVM; blob DA used for L1 commits. |
| **v25**+ | 2025+ | Continued upgrades. {{unsourced: track via era-contracts releases}} |

Always pull current protocol version from `zks_getMainContract` / `Bridgehub` queries.

## Boojum proof system migration

The single biggest under-the-hood change. Pre-Boojum and post-Boojum produce the same L2 state from the same inputs (it's just a faster/cheaper prover) — but the L1 verifier contract is **different**. An L1-aware indexer that decodes verification events must handle both eras.

L2 RPC behavior is unchanged across the migration; standard `eth_*` and `zks_*` work identically.

## Bridgehub introduction (v23)

Pre-Bridgehub: each ZK Stack chain had its own deposit/withdrawal contracts on L1.
Post-Bridgehub: a single `Bridgehub` on L1 routes deposits/withdrawals to the correct chain via `chainId`.

Indexer impact for Era specifically:
- **Old deposit events** are at the per-chain Mailbox / Bridge addresses.
- **New deposit events** are at the Bridgehub.
- A chain-history indexer must read from **both** for the migration period.

For non-Era ZK Stack chains: the Bridgehub-routed flow is the only flow.

## Native AA evolution

Native AA has been present since Era genesis. Key indexer-relevant changes over time:

- **Default account contract** has been upgraded several times (bug fixes, gas optimizations). `eth_call` to query a default-account address returns different bytecode pre- and post-upgrade.
- **Paymaster interface** stabilized at v1 of Era; later upgrades added optional fields (e.g. `validationData` for ERC-4337-style returns) without breaking existing paymasters.
- **EIP-7702** support: zkSync's native AA largely supersedes 7702. {{unsourced: confirm whether Era recognizes 7702-style txs at all}}

## DA mode evolution

| Era | DA mode | Indexer impact |
|---|---|---|
| Pre-blob | Calldata to L1 | Each batch posts as a single L1 calldata blob. Indexer decodes from L1 tx input. |
| Post-blob {{unsourced: protocol version}} | EIP-4844 blobs | Batch data lives in L1 blobs; indexer must fetch from beacon chain. |

This mirrors OP Stack's pre/post-Ecotone migration.

## What an indexer must do per upgrade

1. **Pin the protocol version** at startup via `zks_getMainContract` and the chain's L1 verifier address.
2. **Detect Bridgehub vs legacy bridge**: if the chain is post-v23, route deposit/withdrawal queries through Bridgehub; else through per-chain Mailbox.
3. **Update verifier ABI** if you decode L1 verification events (Boojum vs legacy).
4. **Switch DA decoder** at the blob-DA activation block.
5. **Handle default account upgrades**: don't cache default-account bytecode across version boundaries.

## ZK Stack ecosystem

Era is the **first** chain on the ZK Stack. Other Hyperchains (chain IDs assigned via Bridgehub) share:
- The same EraVM, same system contracts, same tx types.
- Different sequencer operators, different fee tokens (some non-ETH), different settlement layers (some settle to Era directly, not L1).

Indexer patterns transfer; addresses do not. Reading `Bridgehub` events lets you enumerate registered chains.

## Where this doc is incomplete

Era's protocol versions, exact Boojum activation height, blob DA activation, Bridgehub introduction, and the relationship between v22/v23/v24 and Ethereum-equivalent EIPs are all flagged `{{unsourced}}`. Verify against:

- [`matter-labs/era-contracts`](https://github.com/matter-labs/era-contracts) release tags.
- [`matter-labs/zksync-era`](https://github.com/matter-labs/zksync-era) `CHANGELOG.md`.
- The L1 `StateTransitionManager` contract events for each protocol version bump.
