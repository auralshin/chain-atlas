# Reorgs and Finality

## Finality model

<Restate the architecture-level fact in indexer-relevant terms: "an event is final when ___".>

## Empirical reorg depth

<What's actually been observed in production. Cite block ranges or post-mortems. Not the theoretical bound.>

## Safe confirmation depth

<Recommended N for "will not reorg". Different from the theoretical bound. Different for different consumers — a swap router and a payments processor have different tolerances.>

## Reorg detection

<How the indexer notices a reorg. Block-hash mismatch on parent? Finality gadget message? L1 batch inclusion? Slot vote weight? Be specific to this chain.>

## Recovery pattern

<What the indexer rolls back, in what order, with what guarantees. Code sketch acceptable.>

```rust
// Sketch of the rollback loop.
```

## Cross-link

See [`docs/concepts/reorgs.md`](../../concepts/reorgs.md) for the cross-chain framing.
