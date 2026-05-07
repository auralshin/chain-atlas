# Reorgs and Finality

Three relevant heads on Arbitrum, mapping roughly to OP Stack's three but with different mechanics.

## The three heads

| Head | Reported by | Lag from real time | Reorg likelihood |
|---|---|---|---|
| **Sequencer feed** | Sequencer broadcast / `eth_subscribe("newHeads")` | ~0 s | Real, see below |
| **L1-confirmed** | L1 `SequencerBatchDelivered` event + L1 finality of the batch tx | seconds to minutes | Only on L1 reorg |
| **Finalized** | L1 assertion confirmation (legacy `NodeConfirmed` or BoLD `EdgeConfirmed`) | ~7 days legacy / shorter under BoLD | Effectively never |

There is **no `eth_blockNumber` analog of `safe`/`finalized` block tags built into the Arbitrum RPC** the way OP Stack supports them — but Arbitrum nodes do support `eth_getBlockByNumber("safe")` and `eth_getBlockByNumber("finalized")` mapped to the `SequencerInbox`-confirmed head and assertion-confirmed head respectively {{unsourced: verify nitro support}}.

## Reorg sources, ranked

### 1. Sequencer rewinds the broadcast feed

Same as OP Stack: the sequencer can publish a different unsafe chain. The Arbitrum sequencer's broadcast feed is a separate WebSocket from the standard `eth_subscribe("newHeads")` — **the broadcast feed can rewind**, and the standard subscription will follow it.

In practice, sequencer rewinds on Arbitrum One are 1–3 blocks at a time, occasionally larger after sequencer software hiccups. **Do not commit on the sequencer feed** for any consumer that treats data as final.

### 2. L1 reorg invalidates a batch

If L1 reorgs past a `SequencerInbox.addSequencerL2Batch` tx, the batch's L2 blocks lose their L1-confirmed status. Re-derivation from the new L1 chain may produce a different L2 chain (different batch ordering, different delayed-inbox messages included).

The L2 reorg depth is bounded by the L2 blocks derived from the orphaned L1 range. With Arbitrum's variable block time and large batch sizes, this can be hundreds of L2 blocks per L1 block — much larger than OP Stack's ~6× ratio.

### 3. Successful BoLD challenge (theoretical)

A successful challenge invalidates an assertion and replaces the canonical state at finality. To date, no honest BoLD challenge has overturned an assertion on Arbitrum One {{unsourced: confirm}}.

### 4. Legacy challenge resolution (pre-BoLD)

Same shape as BoLD challenge, just with the older interactive bisection mechanism. To date, no successful honest challenge on legacy Arbitrum either {{unsourced: confirm}}.

## Recommended commit policy

| Use case | Commit on |
|---|---|
| Real-time analytics / dashboards | Sequencer feed, marked |
| Token transfers, contract state | L1-confirmed head |
| Bridge accounting, audit-grade ledgers | Finalized head (assertion confirmed) |

For most products, the **L1-confirmed head** is the right tradeoff: the sequencer feed is too volatile, the finalized head is days old. Only bridge withdrawal accounting genuinely needs the 7-day finalized state.

## Reorg detection

Arbitrum does not emit explicit reorg events. Detect by:

1. Subscribing to both the sequencer feed (`newHeads`) and L1 `SequencerBatchDelivered` events.
2. On each new L2 head, compare the parent hash with your stored chain. If mismatch, walk back until parent matches, rewind, re-ingest.
3. On each L1 batch, mark its L2 block range as L1-confirmed in storage.
4. On L1 reorg, invalidate the corresponding batch's L2 block range and re-derive.

Same shape as ETH/OP Stack reorg detection.

## Retryable ticket lifecycle

The most distinctive Arbitrum state machine. A user creates a retryable from L1; the indexer must track it through up to **three** observable on-chain events:

```
L1: Inbox.createRetryableTicket
  └─► L1: InboxMessageDelivered (kind = 9)
       └─► L2: ArbitrumSubmitRetryableTx (0x69) — ticket created
            ├─► L2: ArbitrumRetryTx (0x68) — auto-redeem if gas was paid
            └─► [optional] L2: ArbitrumRetryTx (0x68) — manual redeem within 7 days
                 └─► [if not redeemed in 7 days] ticket expires
```

Recommended state machine for indexers:

| State | Means |
|---|---|
| `pending` | Ticket created, not yet redeemed |
| `redeemed_auto` | Auto-redeem succeeded |
| `redeemed_manual` | Manual redemption succeeded |
| `expired` | 7 days elapsed without redemption |
| `cancelled` | Beneficiary called `ArbRetryableTx.cancel` |

Each state transition is observable in L2 tx data (or by 7-day timer for `expired`).

## Outbox lifecycle (L2→L1)

User burns on L2 → state must be confirmed on L1 → user proves+executes on L1 → funds released. Lifecycle:

```
L2: ArbSys.sendTxToL1 (emits L2ToL1Tx event)
  └─► [next L2 block] sendRoot of block updated
       └─► L1: assertion containing this sendRoot is posted
            └─► L1: assertion confirmed (legacy NodeConfirmed or BoLD EdgeConfirmed)
                 └─► L1: user calls Outbox.executeTransaction
                      └─► L1: OutBoxTransactionExecuted
                           └─► funds released
```

Time from `L2ToL1Tx` to `OutBoxTransactionExecuted` is **at minimum 7 days** (legacy challenge window) — even if the user proves immediately, they cannot execute until the assertion containing the message is confirmed.

For an indexer building "track my withdrawal" UIs, the four states are:

1. **Initiated** — `L2ToL1Tx` event emitted on L2.
2. **L1-pending** — sendRoot in an L1 assertion that is not yet confirmed.
3. **Executable** — assertion confirmed; user has not yet called `executeTransaction`.
4. **Completed** — `OutBoxTransactionExecuted`.

## Sequencer downtime / force inclusion

If the sequencer is offline:
- Sequencer feed stops advancing.
- L1-confirmed head continues to advance for already-batched blocks until the queue drains.
- After **24 hours** of no L1 batch posting {{unsourced: confirm window}}, anyone can force-include delayed-inbox messages, which causes L2 to advance from those messages without sequencer participation.

Indexers should not block on sequencer-feed advancement for liveness; use L1-confirmed.

## Inactivity / non-finalization

Unlike Ethereum, Arbitrum's "finalization" depends on **L1 finality + assertion confirmation**. If L1 stops finalizing (inactivity leak), Arbitrum's finalized head also stops advancing — even if the sequencer keeps producing blocks. This cascades the L1 finality risk to the L2.
