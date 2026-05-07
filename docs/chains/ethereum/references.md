# References

## Specifications

- [Ethereum Execution Layer Specs](https://github.com/ethereum/execution-specs) — pyspec; canonical EL reference
- [Ethereum Consensus Layer Specs](https://github.com/ethereum/consensus-specs) — canonical CL reference
- [Yellow Paper](https://ethereum.github.io/yellowpaper/paper.pdf) — formal definition (now mostly historical)
- [EIPs index](https://eips.ethereum.org/) — all EIPs by status and category
- [Engine API spec](https://github.com/ethereum/execution-apis/tree/main/src/engine) — EL ↔ CL coupling
- [JSON-RPC spec](https://github.com/ethereum/execution-apis) — `eth_*`, `debug_*`, `net_*`, `web3_*`
- [Beacon API spec](https://ethereum.github.io/beacon-APIs/) — `/eth/v1` and `/eth/v2` endpoints

## Network upgrades

- [ethereum.org/en/history](https://ethereum.org/en/history/) — chronological list of all upgrades
- [Pectra mainnet announcement (2025-04-23)](https://blog.ethereum.org/2025/04/23/pectra-mainnet)
- [Fusaka roadmap](https://ethereum.org/roadmap/fusaka/)
- [Glamsterdam roadmap](https://ethereum.org/roadmap/glamsterdam/)

## Client documentation

- [geth docs](https://geth.ethereum.org/docs)
- [reth book](https://reth.rs/)
- [erigon docs](https://github.com/erigontech/erigon)
- [nethermind docs](https://docs.nethermind.io/)
- [besu docs](https://besu.hyperledger.org/)
- [lighthouse book](https://lighthouse-book.sigmaprime.io/)
- [prysm docs](https://docs.prylabs.network/)

## Indexer infrastructure

- [alloy](https://github.com/alloy-rs/alloy) — canonical Rust client (1.7+)
- [reth ExEx](https://reth.rs/exex/) — in-process indexer plugins
- [Substreams](https://substreams.streamingfast.io/) — out-of-band streaming layer
- [ethpandaops](https://ethpandaops.io/) — client diversity, network monitoring

## Trace API references

- [debug_trace* (geth)](https://geth.ethereum.org/docs/interacting-with-geth/rpc/ns-debug)
- [Trace API (parity-style, erigon)](https://github.com/erigontech/erigon/blob/main/cmd/rpcdaemon/commands/trace_adhoc.go)
- [Otterscan API](https://docs.otterscan.io/api-docs/ots-api)

## MEV / PBS

- [mev-boost](https://github.com/flashbots/mev-boost)
- [Flashbots transparency dashboard](https://transparency.flashbots.net/)
- [EIP-7732 (ePBS)](https://eips.ethereum.org/EIPS/eip-7732)

## Blob references

- [EIP-4844 (Proto-Danksharding)](https://eips.ethereum.org/EIPS/eip-4844)
- [EIP-7594 (PeerDAS)](https://eips.ethereum.org/EIPS/eip-7594)
- [EIP-4444 (history expiry; retention rationale)](https://eips.ethereum.org/EIPS/eip-4444)

## EIP-7702 references

- [EIP-7702 (Set Code for EOAs)](https://eips.ethereum.org/EIPS/eip-7702)
- [EIP-7702 deep dives and implementation guides](https://eip7702.io/)
