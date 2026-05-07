# Gotchas

EVM-trained indexers consistently break on Solana. These are the failure modes.

## Slots ≠ blocks

A slot is a 400 ms time window. A block is the output of a slot **if** the leader produced one. Many slots produce no block (skipped). `getBlock(N)` may return null.

Indexers tracking "blocks" that increment-by-1 from genesis are wrong — they need to either:
- Use `getBlockHeight` (sequential block counter, distinct from slot).
- Use `getBlocks(start, end)` to enumerate which slots produced blocks.

## "Logs" don't exist as a first-class concept

Solana has no `Log` table. What looks like "events" are:
- Free-form text strings (`Program log: ...`) for human debug.
- Base64-encoded program data (`Program data: ...`) for structured Anchor-style events.
- Inner instructions (the call tree) for "what programs were called."
- `preBalances` / `postBalances` and `preTokenBalances` / `postTokenBalances` for "what changed."

Indexers expecting `eth_getLogs`-like access find nothing. Pick the right pattern per dApp:
- **Token transfers**: `pre/postTokenBalances` is canonical. Inner instructions to SPL Token's `Transfer` / `TransferChecked` are supplementary.
- **DEX swaps, NFT mints, etc.**: program-specific. Most use Anchor; decode via IDL.
- **System / native operations**: `pre/postBalances` shows lamports moves.

## CPI traversal is required for "internal txs"

A user submits a tx that calls Program A. Program A calls Program B and Program C via CPI. The full call tree is in `meta.innerInstructions`.

Indexers building "what really happened" need to walk this tree, not just the top-level instructions. Many dApp activity (Jupiter swaps, lending protocol borrows-and-deposits, etc.) happens entirely in inner instructions.

## V0 transactions need ALT resolution

A V0 tx's `accountKeys` is **incomplete** — it has the static keys but not the ALT-referenced keys. To know "what accounts did this tx touch":

```
all_accounts = accountKeys 
            + meta.loadedAddresses.writable 
            + meta.loadedAddresses.readonly
```

Indexers using only `accountKeys` for V0 txs miss the majority of touched accounts on aggregator / MEV transactions.

## ALT state is mutable; tx state must be from the tx's slot

ALTs can be extended over time. A V0 tx submitted at slot N references ALT entries valid at slot N — entries added later are not retroactively in the tx.

For backfill, `meta.loadedAddresses` gives you the resolved keys directly (validator already did the lookup). For real-time, the Geyser stream includes resolved keys.

If you ever need to **re-resolve** an ALT for a historical tx (e.g. you stored only `accountKeys` and now need the full set), you need historical ALT state — which most validators don't have. This is an argument for storing `loadedAddresses` from the tx receipt at ingest time.

## SPL Token vs Token-2022 require separate decoders

Two distinct programs. New tokens are issued under either. Your decoder must:
- Detect program ID (different addresses).
- Decode account data using the appropriate format (Token-2022 has extensions in TLV format after the base fields).
- Handle the differing instruction sets.

`preTokenBalances` / `postTokenBalances` give decoded balances that work for **both** programs — these are your friend.

## No `tx.from` — multi-signer txs are common

A tx has `signatures[]`. The first signature's signer is conventionally the "fee payer." But other signers in the tx are not implicitly senders — they're co-signers with their own roles per the program's logic.

Indexers attributing "this user did X" must look at the program's semantics, not the first signer. For SOL transfers, the System Program's `transfer` instruction explicitly names source and destination accounts; that's the source of truth.

## No historical state queries

Solana validators don't retain historical state by default. `getAccountInfo(pubkey, slot=N)` is generally **not** supported (most providers ignore the slot parameter or return current state).

For historical queries:
- Use a snapshot service (Helius, Triton).
- Reconstruct from `pre/postBalances` over time.
- Use Big Table (Solana's archive on Google Cloud).

This is fundamentally different from EVM archive nodes.

## `getProgramAccounts` is operationally expensive

Querying "all accounts owned by program X" can return millions of accounts. Most providers throttle or disable this method. Use it sparingly and with filters.

For "all token accounts owned by user", use `getTokenAccountsByOwner` instead — it's much cheaper.

## Reorgs at `processed` are real and frequent

Slots committed at `processed` can change between calls. Indexers that ingest at `processed` must:
- Track slot status transitions explicitly.
- Re-fetch / reconcile when slots transition to `confirmed` or `finalized`.

The simpler approach: **never index at `processed`**; ingest at `confirmed`. Lose 1–2 seconds of latency, gain reorg-free correctness.

## Compute units, not gas

A tx specifies `compute_unit_limit` (max CU) and `compute_unit_price` (microlamports per CU). Total tx cost = base fee + (CU × price / 1e6). Indexers building fee dashboards must aggregate both components.

`compute_units_consumed` in the receipt is the actual usage; `limit` is the max.

## Cluster halts / stalls

Solana has had multi-hour cluster halts. During halts:
- Slots stop advancing.
- Indexer streams go quiet.
- On restart, slots resume; **there is no replay of "missed" slots** — they never existed.

Indexers must:
- Tolerate stale state without crashing.
- Reconcile on restart by reading the cluster's actual current slot, not assuming continuity from your last seen slot.
- Read official channels for retroactive corrections.

## `signature` is base58, not hex

Tx hashes are base58-encoded ed25519 signatures. ETH-trained code that assumes "hash is hex" gets character-set errors.

Pubkeys are also base58 (32-byte ed25519). All Solana addresses look like `B58…` mixed-case strings.

## Lamports vs SOL

Native unit is **lamports** (10^-9 SOL). Receipts and account balances use lamports. Don't confuse with SOL when computing dashboards. 1 SOL = 10^9 lamports.

## Rent

Every account pays "rent" for storage — historically deducted per epoch, now most accounts are **rent-exempt** via a 2-year deposit. Rent-exempt accounts have no rent deduction. Rent is observable in `pre/postBalances` deltas, but for most active accounts it's zero.

## Vote accounts, stake accounts, vote txs

Validators send `Vote` txs constantly — these dominate Solana's tx volume. ~50%+ of all txs are votes. Indexers analyzing user activity must filter these out:

```
filter where program_id != "Vote111111111111111111111111111111111111111"
```

Otherwise your "activity" graphs are dominated by validator votes.

## Programs can be upgraded (BPF Loader Upgradeable)

A program at a given address can have its bytecode replaced. The IDL and instruction handlers can change. Indexers depending on a program's behavior must:
- Track upgrade events on `BPFLoaderUpgradeable`.
- Refresh decoders if the program's IDL changed.

## `transactions[].signatures[0]` ≠ event/op identifier

A tx with multiple instructions doesn't have a single "operation type." A tx might be: ComputeBudget setup + ALT extend + Jupiter swap + Compute returns. Calling it "a Jupiter swap tx" is a dApp-level interpretation; on-chain it's just a tx with N instructions.

For activity analytics, classify by **instruction**, not by tx.

## Historical stake state cannot be queried — `getAccountInfo` returns current state only

Solana validators don't retain historical account state. `getAccountInfo` and `getMultipleAccountsInfo` return **current** state, which disagrees with what was truthful at a past reward epoch in three common cases:

1. **`accountInfo == null`** — the stake account was withdrawn-and-closed after the reward epoch, but did exist and earn during it.
2. **Account exists but is not in `Stake` state** — e.g. it was reinitialized or deinitialized; not currently delegated, but it was during the reward epoch.
3. **`activationEpoch > rewardEpoch`** — the account is currently delegated, but the active delegation is **newer** than the reward epoch you're attributing to. The validator and the stake amount at reward time were different from now.

For correct attribution at any past epoch, **replay the stake account's instruction history** up to the target epoch boundary using the Stake program instructions that touched it:

| Instruction | What it changes |
|---|---|
| `Initialize` / `InitializeChecked` | Sets initial staker + withdrawer authorities, lockup |
| `Authorize` / `AuthorizeChecked` / `AuthorizeWithSeed` / `AuthorizeCheckedWithSeed` | Changes staker or withdrawer authority |
| `DelegateStake` | Sets vote account; sets `activation_epoch` |
| `Deactivate` | Sets `deactivation_epoch` |
| `Split` | Creates a new account with a copy of state |
| `Merge` | Combines two stake accounts into one |
| `Withdraw` | Reduces lamports; can close the account if drained |
| `SetLockup` / `SetLockupChecked` | Updates lockup |
| `MoveStake` / `MoveLamports` | (newer) moves stake or lamports between accounts owned by the same withdrawer |

The active delegation at epoch `N` requires:
- `activation_epoch ≤ N`, **and**
- `deactivation_epoch is None` or `deactivation_epoch > N`.

This is a real indexing pipeline, not a one-off query. For dApps that only need recent rewards, a sliding-window cache of stake-account states at recent epoch boundaries is usually sufficient — full backward replay is reserved for arbitrary historical attribution. See [skeleton.md](skeleton.md#historical-stake-account-reconstruction) for an illustrative implementation.

## Partitioned-rewards distribution window is wider than docs suggest

Post-SIMD-0118 (see [forks-changelog.md](forks-changelog.md#partitioned-epoch-rewards-simd-0118)), staking rewards are distributed across a window of slots at the start of each new epoch — not in a single block. Tickets and reference implementations sometimes suggest scanning `firstSlot..firstSlot+32` (per the [Anza proposal](https://docs.anza.xyz/proposals/partitioned-inflationary-rewards-distribution)) or `firstSlot..firstSlot+64` as a safety margin.

**These are too narrow.** Concrete on-chain observation: epoch 937 distributed staking rewards from `firstSlot + 1` (slot 405216000) through `firstSlot + 296` — close to 5× the 64-slot suggestion.

**Recommended scan window: ~500 slots.** The cost of a wider scan is small (a few hundred extra `getBlock` calls per epoch); the cost of missing rewards is silent under-counting that compounds across epochs.

Track the maximum offset per epoch as a metric. If you ever observe rewards at offset > 400, widen the window further — stake-account count grows over time, and the partition window grows with it.
