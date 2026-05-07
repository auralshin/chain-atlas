# RPC Surface

## Standard endpoints

<Methods every indexer needs: get_block, get_transaction, get_receipt, get_logs (or non-EVM equivalents).>

## Trace / debug methods

<`debug_traceTransaction`, `debug_traceBlock`, `ots_*`, parity-style `trace_*`, custom equivalents. Note which clients support which methods — this is rarely uniform. Cross-link to [`docs/concepts/trace-apis.md`](../../concepts/trace-apis.md).>

| Method | Purpose | Supported clients | Archive required |
|---|---|---|---|
| ... | ... | ... | ... |

## Subscription / streaming

<WebSocket subscriptions, gRPC streams, Geyser plugins, Firehose, Substreams. What does each give you that polling does not?>

## Archive requirements

<Which methods require an archive node. Disk size estimate at a recent block height. Pruning options.>

## Rate limit reality

<Public RPC vs self-hosted. Typical req/sec a public endpoint will tolerate before banning. The indexer's request budget per block.>
