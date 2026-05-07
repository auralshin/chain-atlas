# Forks and Changelog

zkSync Era versions its **protocol version** (incremented at upgrades) and its **proof system** (Plonkish → Boojum, with Boojum further upgraded over time).

## Protocol versions

Each protocol version corresponds to an L1 contract upgrade and L2 system contract upgrade. Indexer-relevant ones:

| Protocol version | Mainnet activation | Indexer impact |
|---|---|---|
| **v1–v15** | 2023-03-24 (Era genesis) → 2023-Q4 {{unsourced: pin per-version}} | Initial Era. Original (non-Boojum) PLONK-style proof system. |
| **v16 (Boojum integration)** | Late 2023 / early 2024 {{unsourced: confirm full activation date}} | **Boojum proof system migration.** L1 verifier contract changed; indexers tracking proof-verification events need updated decoders. L2 RPC unchanged. See "Boojum proof system migration" below. |
| **v17–v21** | 2024-Q1 → 2024-Q1 {{unsourced: pin per-version}} | Successive improvements; minor system contract updates. |
| **v22** | 2024 {{unsourced: confirm exact mainnet activation date}} | **EIP-4844 blob DA support.** `pubdataCommitments` field can hold either calldata-encoded pubdata or KZG blob commitments; batch poster begins using blobs. ([source: zkSync developer discussion](https://github.com/zkSync-Community-Hub/zksync-developers/discussions/494)) |
| **v23** | **Skipped on mainnet.** | Internal-only. zkSync hit issues with v23 and went directly v22 → v24. ([source: zkSync developer discussion #519](https://github.com/zkSync-Community-Hub/zksync-developers/discussions/519)) |
| **v24** | **2024-06-07** (after delays from initial 2024-05-13 / 2024-05-16 schedule) | **Bridgehub** + **P256Verify precompile** + **blob capacity 2 → 16** + **Validium support** + ecAdd/ecMul/ecPairing precompiles + cold/warm storage cost differentiation. ([source: zkSync developer discussion #519](https://github.com/zkSync-Community-Hub/zksync-developers/discussions/519)) |
| **v25**+ | 2024-2025 | Continued upgrades. Verify against `era-contracts` release tags ([latest as of 2025-12-11: zkos-v0.30.2](https://github.com/matter-labs/era-contracts/releases)). {{unsourced: enumerate v25/v26/v27 mainnet activations}} |

Always pull current protocol version from `zks_getMainContract` / `Bridgehub` queries against L1 at runtime.

## Boojum proof system migration

The biggest under-the-hood change in Era's history. Pre-Boojum used PLONK-style SNARKs requiring 100-GPU clusters with 80 GB RAM each; Boojum is STARK-based and runs on consumer GPUs with 16 GB RAM ([source: The Block](https://www.theblock.co/post/239880/zksync-launches-new-proof-system-called-boojum-for-era-mainnet)).

Rollout phases:
- **2023-07-17**: Boojum activated on mainnet in **shadow-proof mode** — generated and verified proofs alongside the legacy system using real production data, but the legacy system remained authoritative.
- **Late 2023 / early 2024 {{unsourced: confirm date}}**: Full migration; Boojum becomes the canonical prover and the legacy verifier is sunset.

For an indexer, the L2 RPC behavior is **unchanged** across the migration — same blocks, same txs, same state. The L1 verifier contract is **different**, so any indexer that decodes proof-verification events must handle both eras.

## Bridgehub introduction (v24)

Pre-Bridgehub: each ZK Stack chain had its own deposit/withdrawal contracts on L1.
Post-Bridgehub (mainnet 2024-06-07): a single `Bridgehub` on L1 routes deposits/withdrawals to the correct chain via `chainId`.

Indexer impact for Era specifically:
- **Old deposit events** are at the per-chain Mailbox / Bridge addresses (pre-2024-06-07).
- **New deposit events** flow through Bridgehub (post-2024-06-07).
- A chain-history indexer covering both eras must read from **both** for the migration period.

For non-Era ZK Stack chains (Hyperchains): the Bridgehub-routed flow is the only flow they've ever had.

## Native AA evolution

Native AA has been present since Era genesis. Key indexer-relevant changes over time:

- **Default account contract** has been upgraded several times (bug fixes, gas optimizations). `eth_call` to query a default-account address returns different bytecode pre- and post-upgrade.
- **Paymaster interface** stabilized at v1 of Era; later upgrades added optional fields (e.g. `validationData` for ERC-4337-style returns) without breaking existing paymasters.
- **EIP-7702** support: zkSync's native AA largely supersedes 7702. {{unsourced: confirm whether Era recognizes 7702-style txs at all}}

## DA mode evolution

| Era | DA mode | Indexer impact |
|---|---|---|
| Pre-v22 | Calldata to L1 | Each batch posts pubdata as a single L1 calldata blob. Indexer decodes from L1 tx input. ~125 KB cap per batch. |
| **v22** (2024 {{unsourced: pin exact mainnet activation}}) | **EIP-4844 blobs (2 per batch)** | Batch data lives in L1 blobs; indexer must fetch from beacon chain (≤ 18-day blob retention) or use a blob archive. The `CommitBatchInfo.pubdataCommitments` field holds KZG blob commitments. |
| **v24** (2024-06-07) | **EIP-4844 blobs (up to 16 per batch)** | Same decoder; higher per-batch capacity. Single L2 batch can now reference up to 16 blobs. |

This mirrors OP Stack's pre/post-Ecotone migration in spirit but the field-level encoding is zkSync-specific.

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
