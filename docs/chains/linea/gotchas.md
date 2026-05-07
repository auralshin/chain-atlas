# Gotchas

Most Scroll gotchas (in [scroll/gotchas.md](../scroll/gotchas.md)) apply to Linea. Linea-specific ones below.

## Type-2 zkEVM gas costs differ from Ethereum

Linea targets "Type 2" equivalence — meaning EVM-equivalent at the spec level, but with **gas cost differences** for some opcodes. Indexers that simulate execution off-chain (e.g. for "what would this tx have cost" estimates) get wrong answers.

**Fix:** use the actual Linea node for any execution simulation. Don't use a generic ETH-equivalent EVM library.

## Excluded opcode txs (early Linea)

In early Linea, certain opcodes were not yet covered by the prover; txs using them were sequencer-rejected. An indexer reading historical Linea data may see "expected behavior" txs missing because the sequencer excluded them.

The set of excluded opcodes shrank over time and is now empty for {{unsourced: confirm: most or all of the cancun-equivalent opcodes are covered}}.

For historical analysis, check the per-fork excluded opcode list and document which periods had which restrictions.

## Fee burn isn't visible at the receipt level

Linea's fee burn (a portion of fees retained off-chain or burned by Linea-specific accounting) doesn't appear directly in receipts. The receipt shows the user's gas paid; the burn is part of how the protocol distributes that gas internally.

Indexers building "where does fee revenue go" dashboards must inspect off-chain accounting or the L1 contract's reporting, not just receipt sums.

## Besu vs geth tracer subtleties

Linea's L2 client is Besu-based. `debug_traceTransaction` works but with Besu's tracer flavor, not geth's:
- Some tracer names are different (`callTracer` works; some custom JS tracers may not).
- Output JSON shape can have minor differences.

If you have an indexer pipeline built against geth's `debug_*` output, expect to update some parsers when adding Linea.

## Slower finality cadence

Compared to Scroll and OP Stack, Linea's L1 finalization has been slower in some periods (hours-to-days {{unsourced: confirm current cadence}}). Bridge-aware indexers should not assume Linea finality timing matches other rollups.

## Same chain id = mainnet 59144, Sepolia 59141

Common copy-paste error. Verify when configuring indexer environments.

## L1 contract upgrade events

The `ZkEvm.sol` and `MessageService` contracts have been upgraded multiple times. Indexers caching L1 contract ABIs at startup must invalidate / update at upgrades. Subscribe to upgrade events on the proxy contract.

## L2 → L1 message claim semantics

Unlike OP Stack's two-step prove-then-finalize (with 7-day window), Linea's flow is closer to Scroll's: once the batch is finalized on L1, the user can claim immediately with a Merkle proof. There is no challenge window.

But: **the time from L2 burn to "claimable on L1" depends on the batch finalization cadence**, which varies. UI flows should show the in-flight stage, not promise a specific wait time.

## EIP-1559 was active from Linea's mainnet

Unlike BSC (where EIP-1559 came late), Linea has had EIP-1559 since mainnet. Standard ETH gas-pricing logic works without era-conditional code.
