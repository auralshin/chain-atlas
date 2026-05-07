# Skeleton

The [Scroll skeleton](../scroll/skeleton.md) works for Linea unchanged, with config substitutions: Linea's L1 contract addresses, Linea's RPC endpoints.

## Configuration delta

```rust
let config = match chain {
    Chain::Scroll => ChainConfig::scroll_mainnet(),
    Chain::Linea  => ChainConfig::linea_mainnet(),  // chain_id 59144, different L1 contracts
    // …
};
```

## Besu-specific tracer caveat

If your indexer uses `debug_traceBlockByNumber` with custom JS tracers built for geth, validate against Linea's Besu tracer engine before relying on them. The standard `callTracer` and `prestateTracer` work; custom JS may not.

For Rust callers using `alloy`'s tracer types, this is mostly transparent — the JSON shapes match.

## Linea-specific RPC enrichment

If you need `linea_estimateGas` for UIs:

```rust
async fn linea_estimate_gas(p: &Provider, tx: &TxRequest) -> anyhow::Result<U256> {
    let v: serde_json::Value = p.raw_request("linea_estimateGas", (tx,)).await?;
    Ok(U256::from_str(v.as_str().unwrap())?)
}
```

This returns gas estimate **including** the L1 fee component, so it's a single-call substitute for `eth_estimateGas + L1FeeOracle.getL1Fee`.

## What can be reused from the Scroll skeleton

Almost everything:
- L2 block + tx + receipt ingestion.
- L1-fee field decoding.
- L1 batch lifecycle tracking (commit + finalize) — but watch Linea's `ZkEvm.sol` events, not Scroll's `ScrollChain` events.
- L1 → L2 message tracking — watch Linea's `MessageService`, not Scroll's `L1MessageQueue`.
- Reorg walk-back.

Adapting a Scroll indexer to Linea is mostly a config + L1 contract address change.

## What's not shown

Same set of things-not-shown as the Scroll skeleton — they apply identically here.
