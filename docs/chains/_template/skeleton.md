# Indexer skeleton

Illustrative Rust code showing the minimum viable indexer for this chain. Not runnable as written — intentionally trims setup, error handling, and dependency wiring.

## Connecting to the node

```rust
// Establish RPC / WebSocket / gRPC connection.
```

## Streaming the primary unit

The "primary unit" is whatever this chain's indexer consumes one of at a time — block, slot, checkpoint, finalized header.

```rust
// Loop or subscription that produces the primary unit.
```

## Decoding the unit

```rust
// What the data type looks like after the client deserializes it.
// Highlight any chain-specific tx types or envelope formats.
```

## Extracting indexable data

```rust
// Logs, events, instructions, state diffs — whatever this chain emits.
```

## Writing to storage

```rust
// Insert into Postgres or ClickHouse. Schema reference: storage.md.
```

## Reorg handling

```rust
// Detect divergence, roll back, re-apply. Reference: reorgs-finality.md.
```

## Notes

<Anything chain-specific that doesn't fit the standard skeleton. Example: for Solana, the fact that you don't have logs in the EVM sense and must parse instructions from inner instructions; for OP Stack chains, the fact that finality requires watching L1.>
