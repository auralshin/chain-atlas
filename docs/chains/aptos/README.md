# Aptos

**Status:** stub
**Family:** non-EVM L1 — Move VM, account-based
**Mainnet launch:** 2022-10-17
**Native node clients:** aptos-core

## Anchor facts (verify in deep dive)

- Move language, AptosFramework standard library.
- Parallel execution via Block-STM — affects how transactions appear (txn ordering vs effective execution order).
- Account-based model with resources; not the same as Sui's object-centric Move.
- Events emitted via `event_handle`s on resources; structured, not log-based.
- REST API (Aptos node API) is the typical indexer interface; gRPC streaming exists for higher throughput.

## To populate

Use [template](../_template/). Move event model and parallel-execution semantics are the headline sections.
