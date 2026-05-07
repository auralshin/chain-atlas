# Forks and Changelog

Scroll's mainnet upgrade timeline. Block heights and timestamps below are pulled from `scroll-tech/go-ethereum`'s chain config and announcement posts.

## Major upgrades

| Upgrade | Activation | Indexer impact |
|---|---|---|
| **Genesis** | 2023-10-17 | Block 0. Pre-Bernoulli era. Pubdata posted as L1 calldata. |
| **Bernoulli** | **block 5,220,340** (~2024-04-29) | **EIP-4844 blob support.** Pubdata moves from calldata to blobs. ~10× DA cost reduction. ([source: Scroll blog](https://scroll.io/blog/blobs-are-here-scrolls-bernoulli-upgrade)) |
| **Curie** | **block 7,096,836 (2024-07-03 08:00 UTC)** | **EIP-1559 + EIP-2930 tx types adopted.** Base-fee schema introduced (until Curie, Scroll did not have an L2 base fee). **zstd compression** of blob contents (~2.8× compression). ([source: Scroll blog "Compressing the Gas"](https://scroll.io/blog/compressing-the-gas-scrolls-curie-upgrade)) |
| **Darwin** | **timestamp 1724227200 = 2024-08-21 08:00 UTC** | **Aggregated bundle proofs:** multiple batches finalized via a single aggregated proof. A batch becomes "a collection of chunks encoded into one EIP-4844 blob." Reduces per-batch L1 verification cost. ([source: Scroll Darwin docs](https://docs.scroll.io/en/technology/overview/scroll-upgrades/darwin-upgrade/)) |
| **DarwinV2** | **timestamp 1725264000 = 2024-09-02 08:00 UTC** | Darwin follow-up; minor adjustments to bundle-proof verifier. |
| **Euclid Phase 1** | **2025-04-16 15:00 UTC** | **OpenVM prover** replaces halo2 circuits. **MPT state commitment** (matches Ethereum's state structure). **EIP-7702** (set-code transactions) + **RIP-7212** (P256 secp256r1) added. **Enforced tx inclusion** + **permissionless batch submission**. Scroll reaches L2BEAT Stage 1. ([source: Scroll Euclid docs](https://docs.scroll.io/en/technology/overview/scroll-upgrades/euclid-upgrade/)) |
| **Euclid Phase 2** | **2025-04-22 07:00 UTC** | Phase-1 follow-up; activation of remaining Euclid components. |
| **Feynman** | {{unsourced: 2025-2026 — verify activation block/timestamp from `scroll-tech/go-ethereum` chain config}} | Successor to Euclid. {{unsourced: confirm scope}} |

Always pull current fork heights from the Scroll chain config or [`scroll-tech/go-ethereum/releases`](https://github.com/scroll-tech/go-ethereum/releases) at runtime — the values above are pinned to the canonical client config as of the anchor date 2026-05-08.

## L1 fee formula evolution

| Era | What changed |
|---|---|
| **Pre-Bernoulli** | Calldata-based pubdata cost. Receipts include `l1Fee` derived from compressed RLP size × L1 gas price × scalar. |
| **Bernoulli (post 2024-04-29)** | Blob-based pubdata. Receipt schema gains `l1BlobBaseFee`-shaped fields. **L1 fee formula changes** to reflect blob base fee + base fee with separate scalars. |
| **Curie (post 2024-07-03)** | L2 base fee introduced (EIP-1559 on L2). User-paid fee gains a base-fee component on top of the L1-fee component. **Fee math shifts.** |
| **Darwin (post 2024-08-21)** | Bundle-proof verification on L1 — **L1 verification cost is amortized across multiple batches**. Indexers attributing L1 verification cost per L2 batch must split the bundle proof's L1 gas across the batches it covers. |
| **Euclid (post 2025-04-16)** | Reduced batch commitment costs (~90% claimed). Cost model under OpenVM differs from halo2 era; if your indexer recomputes L1 verification costs, update the formula. |

If your indexer recomputes L1 fees from receipt inputs, version the formula by L2 block timestamp against the activation timestamps.

## What an indexer must do per upgrade

1. **Pin fork heights / timestamps** at startup from `scroll-tech/go-ethereum`'s chain config.
2. **Version receipt L1-fee decoder** by upgrade (Bernoulli adds blob fields; Curie adds base fee; Darwin shifts amortization).
3. **Update batch decoder** at Bernoulli (calldata → blob), Curie (zstd compression introduced), Darwin (bundle-proof structure).
4. **Update L1 verifier ABI** at Darwin (bundle proofs) and Euclid (OpenVM verifier replaces halo2).
5. **Add `safe`/`finalized` block-tag handling** if you supported pre-Bernoulli history without it.
6. **Detect EIP-7702 txs** post-Euclid.
7. **Backfill any derived fee-in-USD computations** for affected blocks if the formula shifted.

## Prover system migrations

The biggest under-the-hood changes affect the **L1 verifier contract** but not L2 RPC behavior:

| Era | Prover system |
|---|---|
| Genesis → Darwin | halo2-based Plonkish circuits |
| **Darwin** | Aggregated bundle proofs (still halo2 circuits internally; bundling is a higher layer) |
| **Euclid+** | **OpenVM**-based prover (general zkVM, not chain-specific circuits) |

L2 state and RPC are stable across the migration; only L1 verifier changes. An indexer decoding L1 verification events must update at Euclid.

## Stage 1 status (Euclid)

Scroll became the first ZK rollup to reach L2BEAT Stage 1 status at the Euclid upgrade (2025-04). For indexers, this means:

- **Permissionless batch submission**: anyone can post and finalize batches if the sequencer stops. Indexers must not assume only the official sequencer's L1 tx-from address.
- **Enforced tx inclusion**: users can enqueue txs directly from L1 that the sequencer cannot censor.

Both have indexer implications for L1 ingestion: monitor the relevant events from any submitter, not just a hardcoded sequencer address.

## Where this doc is incomplete

The Feynman activation date and scope are flagged. Verify against:

- [`scroll-tech/go-ethereum`](https://github.com/scroll-tech/go-ethereum/blob/develop/params/config.go) chain config — `BernoulliBlock`, `CurieBlock`, `DarwinTime`, `DarwinV2Time`, `EuclidTime`, `EuclidV2Time`, etc. fields.
- [`scroll-tech/scroll-contracts`](https://github.com/scroll-tech/scroll-contracts) release tags.
- Scroll governance proposals at [gov.scroll.io](https://gov.scroll.io/).
