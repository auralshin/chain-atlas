# Gotchas

Highest blast radius first.

## PBS / mev-boost: the proposer didn't choose those txs

**Symptom:** Indexer attributes block contents to the proposer's address; analytics show "validator X is responsible for tx Y" — but it isn't.

**Cause:** Most mainnet blocks today are built by external block builders and delivered via mev-boost relays. The proposer signs the entire block as a unit; the proposer rarely picks individual txs.

**Fix:** Pull builder identity from mev-boost relay APIs (flashbots, ultrasound, agnostic, etc.) and store as a separate field. Treat `block.miner` (proposer fee recipient) as a payment destination, not authorship. Once EIP-7732 (ePBS, scheduled Glamsterdam) ships, builder identity becomes part of the protocol and on-chain attestable.

**Source:** [mev-boost](https://github.com/flashbots/mev-boost), [Flashbots transparency](https://transparency.flashbots.net/)

---

## Blobs aren't in JSON-RPC

**Symptom:** Indexer fetches type-3 transactions; blob bodies are missing. `eth_getTransaction` returns blob versioned hashes only, no payloads.

**Cause:** Blobs live on the beacon node, not the EL. They are retrieved via Beacon API `/eth/v1/beacon/blob_sidecars/{block_id}` and **retained ~18 days** (4096 epochs).

**Fix:** Run a beacon client alongside. Persist blob bodies at ingest if they're needed for downstream replay, fraud-proof reconstruction, or rollup transaction reconstruction.

**Source:** [EIP-4844](https://eips.ethereum.org/EIPS/eip-4844)

---

## Withdrawals are not transactions

**Symptom:** Indexer's `transactions` table is missing validator withdrawal credits. Address balance reconstruction is wrong from Shapella onward.

**Cause:** Validator withdrawals appear in block bodies as a separate list. They have no tx hash, no signature, no log emission.

**Fix:** Add a `withdrawals` table; index by destination address. **Amounts are gwei, not wei** — multiply by 1e9 before adding to a wei-denominated balance.

**Source:** [EIP-4895](https://eips.ethereum.org/EIPS/eip-4895)

---

## Internal calls don't emit logs

**Symptom:** A contract receives ETH via `CALL` from another contract; indexer shows zero balance change, no event.

**Cause:** EVM `CALL`/`DELEGATECALL` operations are not logged unless the receiver explicitly emits an event. ETH transfers between contracts are invisible to `eth_getLogs`.

**Fix:** Use trace methods (`debug_traceBlock` with `callTracer`, or `trace_block` on parity-style clients) to extract internal calls. Note that this is an order-of-magnitude more expensive than receipts-only indexing — most indexers run a separate trace-aware pipeline only for the addresses that need it.

---

## EIP-7702: an account's "type" is not static

**Symptom:** Indexer caches `eth_getCode` results to classify accounts as EOA vs contract; classification flips between blocks.

**Cause:** Post-Pectra, an EOA can attach delegation code via a type-`0x04` tx. `eth_getCode` returns a delegation pointer of the form `0xef0100 || address` until the authorization is revoked or replaced (by signing a new authorization tuple).

**Fix:** Don't cache `eth_getCode` across blocks for accounts that have ever been the authority of a type-`0x04` tx. Treat "is contract" as a per-block property. Maintain a per-address timeline of delegation state if downstream consumers need it.

**Source:** [EIP-7702](https://eips.ethereum.org/EIPS/eip-7702). Delegation indicator is 23 bytes: 3-byte prefix `0xef0100` + 20-byte target address. `EXTCODESIZE` returns `23`; `CODESIZE` inside delegated execution returns the size of the actual code at the target.

---

## Reverted txs are still in the block

**Symptom:** Analytical query "txs to address X" includes failed txs; downstream consumer assumes all are successful.

**Cause:** `status = 0` receipts are still recorded — they consumed gas, paid the fee, and exist in the block. Logs from a reverted tx are rolled back (the receipt's `logs` field is empty), but the tx itself, gas usage, and the recipient are recorded normally.

**Fix:** Always filter on `status = 1` for "actually happened" semantics. For fee analytics, include all receipts.

---

## Finality can stall (inactivity leak)

**Symptom:** Indexer's `finalized_watermark` stops advancing for hours. Downstream commits queue up.

**Cause:** Beacon chain entered inactivity leak — under non-finalizing conditions (mass validator outage, network partition), finality halts while validator stake is slowly burned to restore the 2/3 threshold.

**Fix:** Either fall back to `safe` checkpoint with explicit downgrade and an alert, or pause and surface the stall to operators. **Don't silently retry on finality** — if Eth is in inactivity leak, retrying is a hot loop that makes nothing happen faster.

---

## Pre-Byzantium receipt format

**Symptom:** Indexing pre-2017 blocks; receipts have a `postStateRoot` field instead of `status`.

**Cause:** Byzantium (2017-10-16, block 4,370,000) replaced post-state roots with a 1/0 status field (EIP-658). Pre-Byzantium receipts have the older format.

**Fix:** Branch on block number. For blocks before 4,370,000, derive success from gas-used heuristics or skip status-based filtering altogether — most indexers covering pre-Byzantium data accept this loss of fidelity.
