# References

## Source code

- **`go-ethereum` (Scroll fork)** — [github.com/scroll-tech/go-ethereum](https://github.com/scroll-tech/go-ethereum) — `l2geth`. The L2 execution client.
- **Scroll contracts** — [github.com/scroll-tech/scroll-contracts](https://github.com/scroll-tech/scroll-contracts) — L1 + L2 system contracts (`ScrollChain`, `L1ScrollMessenger`, `L1MessageQueue`, `L2ScrollMessenger`, `L1GasPriceOracle`, `L2GatewayRouter`).
- **Prover / zkEVM circuits** — [github.com/scroll-tech](https://github.com/scroll-tech) (multiple repos for prover, halo2 circuits, etc.) — not needed for indexing but useful for understanding what's being proven.

## Documentation

- **Scroll docs** — [docs.scroll.io](https://docs.scroll.io/) — official; covers architecture, RPC, contract addresses, fork timeline.
- **Bridge / messaging guide** — within docs.scroll.io; explains the L1↔L2 messenger flow.

## Block explorers

- **Scrollscan** — [scrollscan.com](https://scrollscan.com/) — Etherscan-family; supports Scroll's L1-fee receipt fields.

## Operational

- **Scroll status** — [status.scroll.io](https://status.scroll.io/) {{unsourced: confirm URL}}.
- **L2BEAT Scroll page** — [l2beat.com/scaling/projects/scroll](https://l2beat.com/scaling/projects/scroll) — independent monitor.

## Type libraries

There is no Scroll-specific Rust types crate. Use `alloy` for everything; supplement with a custom receipt struct that includes the L1-fee fields (alloy's standard receipt does not include them by default).

## Where this doc is incomplete

The following claims carry `{{unsourced}}` markers and need primary-source backing:

- Bernoulli, Curie, Darwin, Euclid, Feynman activation block heights or timestamps.
- Whether and when EIP-7702 (set-code transactions) lands on Scroll.
- Exact predeploy addresses (`0x5300…0002`, `0x5300…0005`, `0x5300…0006`).
- L2 block time (claimed ~3 s; may have changed).
- Whether Scroll has a `scroll_*` RPC namespace and what it includes.
- Whether `eth_getBlockByNumber("safe"/"finalized")` block tags work on Scroll (claimed yes).
- Force-include / EnforcedTxGateway mechanism for sequencer downtime.
- Specific opcode coverage gaps (whether Cancun-equivalent opcodes are all supported).

Verify against the Scroll docs, the contracts repo, and the L2BEAT timeline before promoting this doc to `deep`.
