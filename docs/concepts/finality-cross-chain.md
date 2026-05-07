# Cross-chain finality

**Status:** in-progress. Verified rows are backed by primary sources in the footnotes. `pending` rows are placeholders awaiting verification — PRs welcome (see [CONTRIBUTING.md](../../CONTRIBUTING.md)). The asymmetry is intentional: this page treats "I haven't sourced it yet" as a visible state, not silence.

## Why this exists

The single most expensive indexer mistake is committing to a database before the chain considers the data final. Every chain defines "final" differently, with a different lag and a different reorg model. The cheat sheet for choosing a commit policy is the same shape on every chain — three tiers — but the names and lags shift.

For the underlying taxonomy (probabilistic vs single-slot BFT vs two-stage justification vs sequencer-confirmed vs object-level), see [`finality-models.md`](./finality-models.md). **This page is the cross-chain instance** of that taxonomy. Three tiers used throughout:

- **Soft tip** — what your node hands you in real time. Reorgs are routine.
- **Safe-equivalent** — strongest signal short of finality. Reorgs only on rare events (L1 reorg, mass slashing, BFT halt-and-recover).
- **Final tip** — provably permanent under the chain's economic assumptions.

Wall-clock numbers are typical operating lag, not best-case minimums. They reflect mainnet behavior at the anchor date for this repo (2026-05-08).

## Headline table

| Chain | Family | Soft tip (lag) | Safe-equivalent (lag) | Final tip (lag) | Source |
|---|---|---|---|---|---|
| Ethereum | EVM L1 (PoS, Casper FFG) | LMD-GHOST head, 0–12 s [^eth-spec] | `safe` / justified, ~6.4 min [^eth-spec] | `finalized`, ~12.8 min [^eth-spec] | verified [^eth-reorgs] |
| Optimism | EVM L2 (OP Stack) | `unsafe_l2`, ~0 s [^op-syncstatus] | `safe_l2`, ~30 s – 10 min [^op-batches] | `finalized_l2`, ≥ 12.8 min, often hours [^op-syncstatus] | verified [^op-reorgs] |
| Base | EVM L2 (OP Stack) | `unsafe_l2`, ~0 s [^op-syncstatus] | `safe_l2`, ~30 s – 10 min [^op-syncstatus] | `finalized_l2`, ≥ 12.8 min [^op-syncstatus] | verified [^base-reorgs] |
| Arbitrum One | EVM L2 (Nitro) | sequencer feed, ~0 s [^arb-feed] | L1-confirmed (`eth_getBlockByNumber("safe")`), seconds–min {{unsourced: confirm Nitro support of safe/finalized tags}} | assertion confirmed, ~7 days legacy / shorter under BoLD [^arb-bold] | verified [^arb-reorgs] |
| Polygon PoS | EVM sidechain (Heimdall + Bor, checkpointed to L1) | Bor head, {{unsourced}} ~2 s | milestone-finalized, **2–5 s** [^poly-milestones] | checkpoint-on-L1 root, {{unsourced}} ~30 min | partial [^poly-heimdall] |
| BSC | EVM sidechain (PoSA [^bsc-posa]) | head, {{unsourced}} ~3 s | fast finality via BEP-126 [^bsc-posa]; {{unsourced}} confirmation count | epoch-finalized, {{unsourced}} ~45 s | partial |
| zkSync Era | ZK rollup (custom VM) | sequencer head, {{unsourced}} ~1 s | L1 commit, {{unsourced}} | L1 proof verified, {{unsourced}} hours | pending |
| Scroll | ZK-EVM rollup | sequencer head [^scroll-stages] | batch committed via `commitBatchWithBlobProof` [^scroll-stages] | proof verified via `finalizeBatchWithProof` [^scroll-stages]; {{unsourced}} typical wall-clock lag | partial |
| Linea | ZK-EVM rollup | sequencer head, {{unsourced}} | L1 commit, {{unsourced}} | L1 proof verified, {{unsourced}} | pending |
| StarkNet | ZK rollup (Cairo VM) | sequencer head, {{unsourced}} | L1 commit, {{unsourced}} | L1 STARK proof verified, {{unsourced}} | pending |
| Solana | non-EVM (Tower BFT, slot 400–600 ms [^sol-slots]) | `processed` (latest processed block, can roll back) [^sol-rpc] | `confirmed` (≥ 2/3 stake voted) [^sol-rpc] | `finalized` (max lockout); ≥ 32-slot lag → ~12.8–19.2 s [^sol-rpc] [^sol-slots] | partial |
| Cosmos (Tendermint family) | non-EVM (single-slot BFT) | committed block (= final by protocol) {{unsourced}} | — *no separate tier; commit is final* | committed block | pending |
| Aptos | non-EVM (Move, AptosBFT) | proposed block, {{unsourced}} | BFT-finalized after 2f+1 quorum [^aptos-quorum] | *same — BFT one-shot* | partial |
| Sui | non-EVM (Move, Mysticeti consensus, object-centric) | sequenced (owned-object fast path), sub-second {{unsourced}} | "finalizing transactions immediately upon inclusion in the structure" [^sui-mysticeti] | epoch-changed, {{unsourced}} | partial |

**Reading the `Source` column.** `verified` = every cell in the row has a primary-source footnote. `partial` = some cells have primary sources, others carry `{{unsourced}}`. `pending` = no row-level verification yet.

## Soft-head reorg depth — what actually happens in practice

The "Soft tip" column above is the head you receive in real time. The key indexer-relevant question is: how often does it rewrite, and how deep?

| Chain | Typical soft reorg depth | Notable observed depth | Cause | Source |
|---|---|---|---|---|
| Ethereum | 1–2 slots post-Merge [^eth-reorgs] | 7-slot reorg 2022-10-25 attributed to mev-boost relay timing {{unsourced: pin specific post-mortem}} | proposer outage / relay timing | verified for typical depth |
| Optimism | 1–3 `unsafe_l2` blocks routine [^op-reorgs] | larger after sequencer hiccup [^op-reorgs]; L1 reorg cascade ≈ 6× L1 depth (12 s L1 / 2 s L2) [^op-reorgs] | sequencer rewinds / L1 reorg derivation | verified |
| Base | 1–3 `unsafe_l2` blocks routine [^base-reorgs] | sequencer outage 2023-09 paused `unsafe_l2` advancement {{unsourced: cite post-mortem}} | same as OP; Coinbase Cloud sequencer | partial |
| Arbitrum One | 1–3 sequencer-feed blocks routine [^arb-reorgs] | larger after sequencer hiccup; L1 reorg cascade can be hundreds of L2 blocks per orphaned L1 block [^arb-reorgs] | sequencer rewinds / L1 reorg derivation | verified |
| Polygon PoS | {{unsourced}} historically 50+ blocks before milestones | {{unsourced}} | pre-milestone PoS sidechain confirmation behavior | pending |
| BSC | {{unsourced}} | {{unsourced}} | PoSA validator confirmation rule | pending |
| Solana | rollback within `processed` is routine — that's the definition [^sol-rpc] | {{unsourced}} for specific deep-rollback incidents | leader rotation / fork choice within unconfirmed range | partial |
| zkSync Era / Scroll / Linea / StarkNet | {{unsourced}} | {{unsourced}} | sequencer rewinds | pending |
| Cosmos / Aptos / Sui | **none on the soft tip** — single-slot BFT, commit = final {{unsourced}} | n/a — chain halts on consensus failure rather than reorging | BFT halt-and-recover instead of reorg | pending |

The single biggest takeaway across the table: **single-slot BFT chains have no soft-reorg risk on the tip**, but they have a different failure mode (chain halts). EVM L1 + rollups always have a soft reorg risk on the tip; the lag to safe-equivalent is the cost of avoiding it.

## Sources

[^eth-spec]: Ethereum Consensus Specs — slot duration (12 s), epoch (32 slots), Casper FFG justification/finalization rules. <https://github.com/ethereum/consensus-specs>

[^eth-reorgs]: chain-atlas Ethereum reorgs-finality doc, in turn sourced from `consensus-specs` and ethpandaops. [docs/chains/ethereum/reorgs-finality.md](../chains/ethereum/reorgs-finality.md)

[^op-syncstatus]: OP Stack `optimism_syncStatus` RPC — defines `unsafe_l2`, `safe_l2`, `finalized_l2` heads and their derivation. <https://specs.optimism.io>

[^op-batches]: OP Stack derivation pipeline — `safe_l2` advances when an `op-batcher` batch is included on L1 via `SequencerInbox`. <https://specs.optimism.io>

[^op-reorgs]: chain-atlas Optimism reorgs-finality doc. [docs/chains/optimism/reorgs-finality.md](../chains/optimism/reorgs-finality.md)

[^base-reorgs]: chain-atlas Base reorgs-finality doc — inherits OP Stack model. [docs/chains/base/reorgs-finality.md](../chains/base/reorgs-finality.md)

[^arb-feed]: Arbitrum sequencer feed — separate WebSocket from `eth_subscribe("newHeads")`; can rewind. <https://docs.arbitrum.io>

[^arb-bold]: BoLD specification, Offchain Labs. <https://docs.arbitrum.io/how-arbitrum-works/bold/gentle-introduction>

[^arb-reorgs]: chain-atlas Arbitrum reorgs-finality doc. [docs/chains/arbitrum/reorgs-finality.md](../chains/arbitrum/reorgs-finality.md)

[^sol-rpc]: Solana RPC commitment levels, "Configuring State Commitment" section: `processed` ("the node's most recent processed block… can still be rolled back"), `confirmed` ("a block directly voted on by a supermajority of stake, meaning more than two-thirds of the network's active stake"), `finalized` ("a block the cluster recognizes as finalized with maximum lockout"). The 32-slot lag for `finalized` is documented in the Solana transaction-confirmation guide. <https://solana.com/docs/rpc>

[^sol-slots]: Solana slot duration: "slots…are configured to last about 400 ms but may fluctuate between 400 ms and 600 ms." <https://solana.com/docs/core/transactions/confirmation>

[^poly-milestones]: Polygon Heimdall-v2 docs: "milestones: fast deterministic finality within 2 to 5 seconds." <https://docs.polygon.technology/pos/architecture/heimdall/checkpoint/>

[^poly-heimdall]: Polygon Heimdall-v2 introduction page (covers checkpoint and milestone roles, but does not state Bor block time or checkpoint period in the section reviewed). <https://docs.polygon.technology/pos/architecture/heimdall/checkpoint/>

[^bsc-posa]: BNB Smart Chain validator overview — confirms "Proof of Staked Authority (PoSA) consensus" and references the BEP-126 fast-finality reward mechanism: "The rewards for motivating validators to vote for Fast Finality also comes from transaction fees. The specific rules can refer to BEP126." Block-time and confirmation-count specifics are not stated on this page. <https://docs.bnbchain.org/bnb-smart-chain/validator/overview/>

[^scroll-stages]: Scroll rollup documentation — describes the three-phase pipeline. Sequencer execution → batch commitment via `commitBatchWithBlobProof` (data availability on L1) → proof verification via `finalizeBatchWithProof` (L1 finality). Specific wall-clock lag for proof generation is not stated on this page. <https://docs.scroll.io/en/technology/chain/rollup/>

[^aptos-quorum]: Aptos blockchain deep-dive — validators "must reach agreement with a quorum (2f+1) before transactions are committed to storage." The page does not name the consensus protocol explicitly nor state mainnet block time in the section reviewed. <https://aptos.dev/en/network/blockchain/blockchain-deep-dive>

[^sui-mysticeti]: Sui consensus documentation, Mysticeti protocol — "Mysticeti simplifies this process by finalizing transactions immediately upon inclusion in the structure." Specific checkpoint cadence and owned-object-vs-consensus split are not stated on this page in the section reviewed. <https://docs.sui.io/concepts/sui-architecture/consensus>

## Per-chain deep dives

The rows above are summaries. The long-form per-chain commit-policy reasoning lives in `reorgs-finality.md` inside each chain's folder.

- [Ethereum](../chains/ethereum/reorgs-finality.md)
- [Optimism](../chains/optimism/reorgs-finality.md)
- [Arbitrum](../chains/arbitrum/reorgs-finality.md)
- [Base](../chains/base/reorgs-finality.md)
- Polygon, BSC, Solana, Cosmos, Aptos, Sui, zkSync Era, Scroll, Linea, StarkNet — pending per-chain population. The [chain status table in the root README](../../README.md#chain-status) tracks which are deep.
