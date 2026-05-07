# Architecture

Solana's consensus is **Proof of History** (PoH) + **Tower BFT** (a PBFT variant), with stake-weighted voting on a continuous slot stream.

## Slots, blocks, leaders

| Concept | Definition |
|---|---|
| **Slot** | A 400 ms time window. Every slot has a designated leader. |
| **Block** | The output of a slot if the leader produced one. **Skipped slots have no block** — slot N is not the same as block N. |
| **Leader** | One validator scheduled per slot via the leader schedule (deterministic, ~one epoch ahead). |
| **Epoch** | 432,000 slots (~2 days). Validators rotate, stake updates, etc. |
| **Slot leader rotation** | Each leader holds 4 consecutive slots before rotating. |

The leader produces ticks (proof-of-history) continuously and bundles transactions into entries; entries are grouped into a block at the end of the slot.

## Proof of History

Sequential SHA-256 hashing produces a "verifiable delay" — each hash's input is the previous hash's output. The hash stream encodes time order without coordination.

For an indexer: PoH is invisible at the data layer. Slots are timestamped, but the timestamps come from the leader's wall clock, not PoH directly.

## Tower BFT consensus

Validators vote on slots. Each vote has a **lockout** that doubles each time the validator votes for descendants of an earlier-voted slot. This makes voting against your own previous vote progressively expensive — the chain converges.

| Confirmation level | Means |
|---|---|
| **`processed`** | The validator has seen the block. May be on a fork. **Not safe for state.** |
| **`confirmed`** | A supermajority of voting stake has voted on this block (or a descendant). Reorgs theoretically possible but rare. |
| **`finalized`** | Rooted by 2/3+ of voting stake; effectively immutable. |

Most production indexers index on `confirmed`; audit-grade workloads use `finalized`.

## Account model

| Concept | Identifier | Properties |
|---|---|---|
| **Account** | 32-byte ed25519 pubkey | `lamports`, `data: Vec<u8>`, `owner` (program), `executable` (bool), `rent_epoch` |

Every entity is an account. There is no contract storage in the EVM sense — instead, programs operate on accounts owned by them.

| Account kind | `executable` | `owner` |
|---|---|---|
| User wallet | false | System Program (`11111…1`) |
| Program (smart contract) | true | BPF Loader (`BPFLoader…`) or BPF Loader Upgradeable |
| Program data (state) | false | The owning program (e.g. SPL Token, Token-2022, custom dApp program) |

A "transfer" is the System Program reducing the sender's `lamports` and increasing the receiver's. There is no `Transfer` event; the change is observable in the **post-tx account state diff**.

## Transactions

| Field | Notes |
|---|---|
| `signatures` | Ed25519 signatures over the message |
| `message.header` | Counts of signers, read-only signers, read-only non-signers |
| `message.account_keys` | All account pubkeys this tx references |
| `message.recent_blockhash` | A recent slot's blockhash (anti-replay) |
| `message.instructions` | Array of instructions; each names a program (by index into account_keys) and the accounts it operates on (by index) |

A single tx can have multiple instructions — they execute sequentially in the same atomic context. If any instruction fails, the entire tx fails.

### Address Lookup Tables (V0 transactions)

Post-V0 transactions support **Address Lookup Tables (ALTs)** — a separate account holding lists of pubkeys. A V0 tx can reference accounts via ALT indexes instead of fully-listing them, allowing more accounts per tx.

For indexers: when ingesting V0 txs, you must **resolve ALT references** by reading the ALT account state. The decoded account list is the union of `static_account_keys + writable_lookups + readonly_lookups`.

## Cross-Program Invocation (CPI)

A program can invoke another program during its execution. CPIs are recorded as **inner instructions** on the resulting tx:

```
Top-level instructions:
  [0] Program A.do_thing(accs...)
       Inner instructions:
         [0,0] Program B.helper(accs...)
         [0,1] Program C.transfer(accs...)
              Inner instructions:
                [0,1,0] System Program.transfer(accs...)
```

This is **the call tree**. Indexers building "internal txs" / "what programs were called" reconstruct it from `inner_instructions` in the tx response.

CPI depth is bounded (currently 4 levels {{unsourced: confirm}}); recursion is prohibited.

## Compute budget

Solana's resource model is **compute units (CU)**, not gas. Each tx has:
- **Compute unit limit** — max CUs the tx can consume.
- **Compute unit price** — priority fee per CU paid to the leader.
- **Heap size** — extra heap allocation if requested.

These are set via `ComputeBudget` instructions at the start of a tx. Indexer impact: receipts include `compute_units_consumed`; for "fee paid" computations, use both base fee + priority fee × CU.

## Where the data lives

| Data | Where to fetch | Notes |
|---|---|---|
| Slots, blocks, txs, instructions | RPC `getBlock(slot)` / Geyser plugin | Geyser preferred for live data |
| Account states | RPC `getAccountInfo(pubkey)` / `getProgramAccounts(program_id)` | Snapshot only; no historical by default |
| Tx logs and inner instructions | RPC `getTransaction(signature)` | Per-tx |
| Confirmation status | RPC `getSlot(commitment=...)` | Different RPC per commitment level |
| Validators / vote accounts | RPC `getVoteAccounts` | For consensus-aware indexing |
| Token balances | RPC `getTokenAccountBalance` etc. | SPL Token-specific helpers |

For high-throughput indexing, **Geyser plugin streams** of accounts + transactions + slots + blocks are the canonical interface.
