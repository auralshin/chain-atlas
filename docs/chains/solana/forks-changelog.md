# Forks and Changelog

Solana doesn't fork in the EVM sense (named hard forks). Instead, protocol changes ship via **feature gates** that activate at specific epoch boundaries when a supermajority of stake has upgraded.

## Feature gates

Each protocol-altering change is associated with a feature pubkey. The validator runs `solana feature status` (or equivalent) to see active features. When ≥ 95% of stake has signaled support, the feature activates at an epoch boundary.

For indexer impact, the relevant features are those that change:
- Tx format (V0, ALTs).
- Instruction encoding.
- Account model semantics.
- Compute budget rules.
- Fee model.

Track active features against [`solana-foundation/specs`](https://github.com/solana-foundation/specs) {{unsourced: confirm correct repo}} or the `runtime` features list in `agave`.

## Notable feature gates / upgrades

| Era / feature | Approx date | Indexer impact |
|---|---|---|
| **Genesis** | 2020-03-16 | Mainnet beta launch. |
| **SPL Token** | 2020-2021 | Original SPL token program — dominant for fungible tokens. |
| **Stake redelegation, vote-related upgrades** | 2021-2022 | Protocol-level; no direct indexer impact. |
| **Address Lookup Tables (V0 transactions)** | 2022 (mainnet activation) | **V0 tx format.** Indexers must resolve ALT references. |
| **`return_data` syscall** | {{unsourced}} | Programs can return small data blobs to callers; observable in receipts. |
| **Token-2022 launch** | ~2023 | New SPL token program with extensions (transfer fees, confidential transfers, etc.). **Indexers must add Token-2022 alongside SPL Token.** |
| **`set_compute_unit_price` instruction** | ~2022 | Priority fee market emerges; indexers track per-tx CU price. |
| **Locked stake activation reformat** | {{unsourced}} | Internal; minor impact on Stake program decoders. |
| **SIMD-0033 (Timely Vote Credits)** | 2024 (governance vote ended at end of epoch 599 with 51.75% YES) | **Validator vote-reward calculation changes** — vote credits now scale with vote latency, not just vote correctness. Indexers building per-validator reward dashboards should not assume "1 correct vote = 1 credit" post-activation. ([SIMD-0033](https://github.com/solana-foundation/solana-improvement-documents/blob/main/proposals/0033-timely-vote-credits.md)) |
| **Partitioned Epoch Rewards** | 2024 mainnet (epoch {{unsourced: pin}}) | **SIMD-0118**; feature gate `enable_partitioned_epoch_reward`. **Staking-reward distribution moves from a single epoch-boundary block to a window of consecutive slots.** Detailed below. |
| **SIMD-0096 (100% priority fees to validators)** | **Mainnet activation: February 2025** (passed governance May 2024 with 77% approval) | **Pre-feature: 50% of priority fees were burned, 50% paid to validator. Post-feature: 100% to validator.** **Indexers tracking burn rates or validator revenue must change formulas at activation epoch.** Pre-activation a tx with `priority_fee = X` burned `X/2` SOL; post-activation it burns 0. ([SIMD-0096](https://github.com/solana-foundation/solana-improvement-documents/blob/main/proposals/0096-reward-collected-priority-fee-in-entirety.md), [The Block](https://www.theblock.co/post/296932/solana-validators-to-receive-full-priority-fees-as-simd-0096-proposal-gains-approval)) |
| **SIMD-0123 (auto priority-fee distribution to stakers)** | Approved March 2025; activation {{unsourced: pin epoch}} | Automates on-chain split of priority fees between validators and SOL stakers. **Indexers attributing per-staker fee-revenue need to add the SIMD-0123 distribution stream alongside staking inflation rewards.** |
| **Agave 1.18 / 2.x** | 2024+ | Anza forks `solana-labs/solana` → `agave`. **Validator software change**; protocol ABI is stable but Geyser plugin compatibility may change. |
| **Firedancer (Frankendancer) partial deploy** | 2024–2025 | Jump's high-performance validator. Uses same Solana protocol; no indexer-data impact, but RPC providers' performance / reliability profiles differ. |

## Big upgrades by phase

### Address Lookup Tables (V0 transactions)

Pre-V0: every account referenced in a tx had to fit in `accountKeys` (≤ 32 accounts per tx with all signatures). Post-V0: ALTs allow 200+ accounts per tx.

**Indexer must:**
- Detect tx version (`version: "legacy" | 0`).
- For V0, fetch ALT account state at the tx's slot to resolve `addressTableLookups` into actual pubkeys.
- ALT accounts are mutable — but for any given tx, you need the ALT state **at that tx's slot** (or the historical ALT state).

This is one of the biggest historical-indexer-breaking changes. ALTs were not standard practice early on; Jupiter, MEV, and large dApps adopted them aggressively from 2023+.

### Token-2022 (SPL Token-2022)

Token-2022 (`TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb`) is a separate program from the original SPL Token (`TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA`). New tokens may be issued under either.

For indexers:
- **Token transfer** instruction shape and decoding are similar but not identical.
- **Account state shape** differs (Token-2022 has TLV-encoded extensions in account data after the standard fields).
- **Mint state** differs.

Decoders must support both. `preTokenBalances` / `postTokenBalances` work for both, but raw-account-data decoders need branching.

### Partitioned Epoch Rewards (SIMD-0118)

Reference: [Partitioned Inflationary Rewards Distribution proposal (Anza)](https://docs.anza.xyz/proposals/partitioned-inflationary-rewards-distribution).

#### Pre-feature behavior

Staking rewards for epoch N were distributed in **a single block** — the first block of epoch N+1. That block's `rewards: [...]` array contained one entry per stake account earning during epoch N. With mainnet's stake-account count (millions of accounts), this single block was famously expensive — the leader had to compute and credit every account's reward in one slot.

Indexer pattern (pre-feature): query `getBlock(epoch_first_slot)`; the `rewards` array is the complete payout for the prior epoch.

#### Post-feature behavior

Rewards calculation still happens at epoch boundary, but **distribution is partitioned across a window of consecutive slots** at the start of the new epoch. Each stake account is assigned to a partition deterministically (by hash of its address); each block in the window pays out one partition's worth of rewards.

Total rewards per stake account are unchanged; only the slot-level timing differs.

#### Empirical distribution window — wider than docs suggest

The Anza proposal describes a fixed 32-slot partition window. Tickets and reference implementations sometimes suggest scanning `firstSlot..firstSlot+64` as a safety margin.

**On-chain observation contradicts both.** Concrete example: epoch 937 distributed staking rewards from `firstSlot + 1` (slot 405216000) through `firstSlot + 296` — close to 300 slots, ~5× the suggested 64-slot window.

**Recommended scan window for indexers: ~500 slots** from the epoch boundary. The reasoning:

- Empirical max observed (~296 slots) is already > 64.
- Stake-account count grows over time, increasing the reward-record count and (potentially) the partitioning window.
- The cost of a wider scan is small (a few hundred extra `getBlock` calls per epoch); the cost of missing rewards is silent under-counting.

#### Indexer checklist for Partitioned Rewards

1. **Detect activation** via the feature gate (`enable_partitioned_epoch_reward` {{unsourced: confirm exact gate name}}). Pre-activation: scan only `epoch_first_slot`. Post-activation: scan the window.
2. **Scan window**: iterate `getBlock(firstSlot + i)` for `i in 1..=500`; collect every `rewards` entry with `rewardType: "Staking"`.
3. **Cross-check**: track the maximum offset at which staking rewards appear per epoch. If you ever see rewards at offset > 400, **widen the window further** — silent under-counting otherwise.
4. **Aggregate per stake account / per validator** across all blocks in the window. The total per stake-account-per-epoch should match the validator's reward × the account's relative stake.
5. **Reward-attribution caveat**: at the moment of distribution, the staker authority on the stake account determines who is credited. If authority was transferred between the reward epoch and the distribution slot, attribution can diverge from "the entity who delegated." See [gotchas.md](gotchas.md) for the historical-reconstruction pattern.

### SIMD-0096: priority fees go fully to validator (Feb 2025)

A subtle indexer-breaking change for fee-revenue and supply-tracking dashboards.

| Era | Priority fee distribution |
|---|---|
| **Pre-SIMD-0096** (genesis → Feb 2025) | 50% paid to validator (block leader), **50% burned** (subtracted from circulating supply) |
| **Post-SIMD-0096** (Feb 2025+) | **100% paid to validator**, 0% burned |

For indexers building:
- **Validator revenue dashboards**: pre/post-activation, the validator-side share doubles for the same priority fee. Don't compare revenue across the boundary without accounting for this.
- **SOL supply / burn tracking**: pre-SIMD-0096, priority fees were a real burn channel reducing circulating supply. Post-activation, this burn channel is gone — supply grows faster relative to inflation alone.
- **MEV / tip analytics**: the relative incentive for validators to include high-priority-fee txs is now larger. Behavioral changes in tx ordering / inclusion likely follow the activation date.

Aggregate "total fees paid by users" remains unchanged — only the destination shifted.

### Agave / Firedancer split

Until 2024, the canonical validator was `solana-labs/solana`. Anza (founded by ex-Solana Labs team) forked into `anza-xyz/agave`, which became the upstream.

Jump's Firedancer (and the hybrid Frankendancer) is a from-scratch validator implementation in C/C++. Currently partially deployed (Frankendancer = Firedancer's networking + replay layers + Agave's runtime).

**For indexers**: protocol behavior is the same across implementations; what differs:
- Geyser plugin compatibility (Agave's Geyser API may differ from Firedancer's).
- Performance characteristics (Firedancer is faster but younger).
- Specific RPC method semantics (rare differences but they exist).

Pin against the validator software you depend on.

## What an indexer must do per upgrade

1. **Track active features** at startup (or per slot ingestion if you care about historical correctness).
2. **Detect tx version** per-tx to choose decoder (legacy vs V0).
3. **Resolve ALTs** for V0 txs at the tx's slot.
4. **Branch on token program** (SPL Token vs Token-2022) for any token decoding.
5. **Update Geyser plugin / gRPC schema** if your provider migrates between Agave / Firedancer.

## Where this doc is incomplete

The exact feature activation epochs / dates for ALTs, Token-2022, return_data, and other features are flagged because Solana doesn't have a single canonical changelog. Verify against:

- [`anza-xyz/agave`](https://github.com/anza-xyz/agave) feature gate list and runtime release notes.
- The `runtime` module's `feature_set.rs` in agave.
- `solana feature status` output against a synced node.
