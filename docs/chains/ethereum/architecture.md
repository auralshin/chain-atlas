# Architecture

## Consensus

Proof-of-Stake since The Merge (2022-09-15). Two coupled algorithms:

- **LMD-GHOST** — fork choice in the unfinalized window. The canonical head is the descendant of the most recently justified checkpoint with the most attestation weight, breaking ties by latest message.
- **Casper FFG** — finality. Validators vote on epoch checkpoints. Two consecutive justified checkpoints finalize the older one. Under normal conditions this happens every ~12.8 min.

Validators stake 32 ETH minimum. **EIP-7251 (Pectra, 2025-05-07) raised the maximum effective balance to 2048 ETH**, allowing a single validator to consolidate up to 64 traditional validators' worth of stake. Validators are slashed for double-signing or surround-voting.

## Block production

| Parameter | Value |
|---|---|
| Slot time | 12 seconds |
| Slots per epoch | 32 |
| Epoch duration | ~6.4 min |
| Finality target | 2 epochs (~12.8 min) |

One proposer per slot, selected pseudorandomly weighted by stake. The proposer's job is to publish a block — but **most mainnet blocks are built by external block builders via MEV-Boost** (Proposer-Builder Separation, currently out-of-protocol). The proposer signs whatever block the builder hands them; the proposer rarely chooses individual transactions.

EIP-7732 (Enshrined PBS) is scheduled for Glamsterdam (H1 2026). Once activated, PBS becomes part of consensus rather than a relay market. Today it remains an off-chain coordination layer.

## Fork choice

LMD-GHOST. Indexer-relevant consequence: a block at slot N can be produced for ~12 seconds before being orphaned by slot N+1's proposer choosing a different parent. **An indexer that writes block N to storage at slot N's deadline must be prepared to roll it back.** See [reorgs-finality.md](reorgs-finality.md).

## Finality model

Casper FFG. An epoch checkpoint becomes:

- **justified** when 2/3+ of validators attest to it
- **finalized** when the *next* checkpoint is justified

| State | Reorg requirement | Typical age |
|---|---|---|
| `head` | 1 honest proposer flips | 0–12 s |
| `safe` | Slashing 1/3+ of stake | ~6.4 min |
| `finalized` | Slashing 1/3+ of stake (two epochs) | ~12.8 min |

**An event is final** when the block containing it is at or below the `finalized` checkpoint reported by the beacon node. Use `/eth/v2/beacon/states/head/finality_checkpoints` and resolve the slot to an EL block.

Under non-finalizing conditions (validator outage, network partition), the **inactivity leak** activates: the chain keeps producing blocks but does not finalize, while inactive validators' stake is slowly burned to restore the 2/3 threshold. **`finalized` can stop advancing for hours or days.** Indexers that block on finality for downstream commits must handle this — see [gotchas.md](gotchas.md).

## State model

Account-based. Two account kinds historically:

- **Externally Owned Account (EOA):** controlled by a private key, no code.
- **Contract Account:** code at a fixed address, no private key.

Post-Pectra (EIP-7702): an EOA can **temporarily attach code** for the duration of a transaction. The code lives in a designation contract; calls to the EOA execute that code with the EOA's storage as `address(this).storage`. This blurs the EOA/contract distinction — indexers that classify accounts by `eth_getCode` returning empty must update their model. See [data-model.md](data-model.md#transactions) and [gotchas.md](gotchas.md).

State is committed in a Merkle Patricia Trie (MPT). Verkle trees are on the long-term roadmap; not live as of 2026-05-08.

## Where the data lives

| Data | Where to fetch | Notes |
|---|---|---|
| Headers, txs, receipts, logs | Execution layer JSON-RPC (geth/reth/erigon/...) | |
| Withdrawals | EL block body | Since Shapella (2023-04-12) |
| Blobs (EIP-4844) | **Beacon node**, not EL | Retained ~18 days |
| Finality status | Beacon node `finalized_checkpoint` | |
| Validator info, attestations | Beacon node | |
| MEV-Boost block source | Off-chain (relay APIs) | Flashbots, ultrasound, agnostic, etc. |

A complete Ethereum indexer needs **both** an execution client and a beacon client. Indexers that only run an EL miss blob bodies and must approximate finality by depth.
