# References

Primary sources for the claims in this chain doc. Items marked `{{unsourced}}` in other files are claims I could not verify from these sources in this writing pass — they need to be confirmed before this doc is promoted to `deep`.

## Specifications

- **OP Stack Specs** — [specs.optimism.io](https://specs.optimism.io/) — canonical for derivation pipeline, deposit txs, predeploys, L1-fee formulas, batch encoding.
- **Source repo** — [github.com/ethereum-optimism/specs](https://github.com/ethereum-optimism/specs) — the spec source; PR history is the changelog.
- **Superchain Registry** — [github.com/ethereum-optimism/superchain-registry](https://github.com/ethereum-optimism/superchain-registry) — chain-specific rollup configs (genesis, L1 contracts, fork timestamps).

## Client implementations

- **`op-geth`** — [github.com/ethereum-optimism/op-geth](https://github.com/ethereum-optimism/op-geth) — geth fork; commits show OP-specific deltas.
- **`op-node`** — [github.com/ethereum-optimism/optimism/tree/develop/op-node](https://github.com/ethereum-optimism/optimism/tree/develop/op-node) — rollup CL.
- **`op-batcher`, `op-proposer`** — same monorepo.
- **`op-reth`** — [github.com/paradigmxyz/reth](https://github.com/paradigmxyz/reth) — Reth-based EL with OP support.
- **`op-erigon`** — [github.com/testinprod-io/op-erigon](https://github.com/testinprod-io/op-erigon) — Erigon-based EL with OP support; flat traces.

## Type libraries (Rust)

- **`op-alloy`** — [github.com/alloy-rs/op-alloy](https://github.com/alloy-rs/op-alloy) — canonical Rust types for OP Stack networks. Tracks fork-shifting receipt fields.
- **`alloy`** — [github.com/alloy-rs/alloy](https://github.com/alloy-rs/alloy) — base ETH RPC + types.

## Bridge contracts (L1)

- `OptimismPortal` — entry/exit point for deposits and withdrawals.
- `L2OutputOracle` — pre-fault-proofs output root posting.
- `DisputeGameFactory`, `AnchorStateRegistry` — post-fault-proofs.
- `SystemConfig` — chain config (sequencer, fee parameters, gas limits).
- `L1CrossDomainMessenger` / `L2CrossDomainMessenger` — message-passing layer.

Source: [github.com/ethereum-optimism/optimism/tree/develop/packages/contracts-bedrock](https://github.com/ethereum-optimism/optimism/tree/develop/packages/contracts-bedrock).

## Operational dashboards

- **OP Mainnet rollup status** — [optimism.io/status](https://status.optimism.io/) {{unsourced: confirm URL}}
- **Etherscan Optimism explorer** — [optimistic.etherscan.io](https://optimistic.etherscan.io/) — L2 explorer; receipts include all OP-specific fields.
- **Blobscan** — [blobscan.com](https://blobscan.com/) — L1 blobs containing OP batches.
- **Dispute games dashboard** — TBD — for tracking output-root challenges.

## Post-mortems and incidents

These are the sources cited for "indexers were broken by X":

- **OVM 1.0 → OVM 2.0 → Bedrock data model changes** — [Bedrock launch retrospective](https://blog.oplabs.co/) {{unsourced: pin specific post}}.
- **Ecotone L1-fee formula change** — [Ecotone announcement post](https://blog.oplabs.co/) {{unsourced: pin specific post}}.
- **Permissioned → permissionless fault proofs** — Optimism governance forum posts {{unsourced: link the specific Snapshot vote}}.

## Where this doc is incomplete

The following claims in this chain doc carry `{{unsourced}}` markers that need primary-source backing before promotion:

- Exact L2 activation timestamps for Canyon, Delta, Fjord, Granite, Holocene, Isthmus on OP Mainnet.
- Exact date of permissioned fault proof rollout on OP Mainnet.
- Status of permissionless fault proofs on OP Mainnet at deep-dive time.
- Whether OP Mainnet has had a successful honest fault proof challenge on a posted output root.
- Current sequencer operator (Optimism Foundation vs successor).
- Whether `eth_getBlockReceipts` semantics changed at Ecotone.

Verify against the specs repo and the superchain-registry, then strip the `{{unsourced}}` markers.
