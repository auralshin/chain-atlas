# Storage

Schema sketches for the two storage backends this repo covers. Cross-link to the cross-chain framing in [`docs/concepts/storage-postgres.md`](../../concepts/storage-postgres.md) and [`docs/concepts/storage-clickhouse.md`](../../concepts/storage-clickhouse.md).

## Postgres schema

```sql
-- Tables you'd actually create. Include partitioning where relevant.
-- Aim for: blocks, transactions, logs/events, traces (if available),
-- and any chain-specific entities (e.g. Solana instructions, OP deposits).
```

When PG is the right choice for this chain: <conditions>.

## ClickHouse schema

```sql
-- ORDER BY, partitioning, and projections matter here.
-- Aim for: same logical entities as PG, but designed for analytical queries.
```

When CH is the right choice for this chain: <conditions>.

## Tradeoffs for this chain

<Throughput estimates: blocks/day, txs/day, logs/day. At what scale does PG break for this specific chain? Some chains (Solana, BSC) saturate PG faster than others.>

## Partitioning strategy

<By block range? By date? By chain ID for multi-chain stores? Why.>

## Reorg-aware writes

<Soft-delete columns? Per-block write atomicity? How does this chain's reorg behavior shape the schema?>
