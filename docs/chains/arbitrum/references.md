# References

Primary sources for the claims in this chain doc. `{{unsourced}}` markers in other files are claims to verify before promotion to `deep` status.

## Specifications and source

- **Nitro source** — [github.com/OffchainLabs/nitro](https://github.com/OffchainLabs/nitro) — single binary; the on-chain consensus is whatever this code computes.
- **ArbOS precompile sources** — [`OffchainLabs/nitro/precompiles`](https://github.com/OffchainLabs/nitro/tree/master/precompiles) — authoritative for `ArbSys`, `ArbRetryableTx`, `ArbGasInfo`, `NodeInterface`, etc.
- **Nitro contracts** — [github.com/OffchainLabs/nitro-contracts](https://github.com/OffchainLabs/nitro-contracts) — L1 contracts (Inbox, Outbox, RollupProxy, SequencerInbox, EdgeChallengeManager).
- **BoLD spec** — [Offchain Labs BoLD docs](https://docs.arbitrum.io/how-arbitrum-works/bold/gentle-introduction) {{unsourced: pin URL}}.

## Documentation

- **Arbitrum Docs** — [docs.arbitrum.io](https://docs.arbitrum.io/) — official; covers tx types, retryable lifecycle, ArbOS, gas pricing, sequencer feed.
- **Arbitrum L2 chain inputs reference** — within `docs.arbitrum.io` — explains the inbox / outbox / sequencer-batch data flow.
- **Stylus docs** — [docs.arbitrum.io/stylus](https://docs.arbitrum.io/stylus/stylus-quickstart) {{unsourced: pin URL}}.

## Chain-specific config

- **Arbitrum One** — chain ID 42161, [Etherscan L1 contracts](https://etherscan.io/address/0xC12BA48c781F6e392B49Db2E25Cd0c28cD77531A) {{unsourced: confirm Inbox address — there are several over time}}.
- **Arbitrum Nova** — chain ID 42170; AnyTrust DAC; separate L1 contract set.

Pull current contract addresses from the Arbitrum docs' "Useful Addresses" page rather than hardcoding from old sources — they change at major upgrades.

## Block explorers

- **Arbiscan** — [arbiscan.io](https://arbiscan.io/) — Etherscan-family. Receipts include all Arbitrum-specific fields. Internal txs use `arbtrace_*`.
- **Nova explorer** — [nova.arbiscan.io](https://nova.arbiscan.io/).

## Operational

- **Arbitrum status** — [status.arbitrum.io](https://status.arbitrum.io/).
- **L2BEAT Arbitrum One page** — [l2beat.com/scaling/projects/arbitrum](https://l2beat.com/scaling/projects/arbitrum) — independent monitor; tracks finality state and assertion lag.

## Where this doc is incomplete

The following claims carry `{{unsourced}}` markers and need primary-source backing:

- Exact ArbOS version activation timestamps on Arbitrum One (versions 9, 10, 11, 20, 30, 31).
- Exact date of blob-DA activation on Arbitrum One.
- Exact date of BoLD mainnet activation on Arbitrum One.
- Stylus mainnet activation date (ArbOS 30/31).
- Whether any honest fault proof challenge has overturned an assertion on Arbitrum One (legacy or BoLD).
- Sequencer downtime force-inclusion window (currently believed to be 24 hours).
- Effective average L2 block time on Arbitrum One.
- Solidity `block.number` semantics on Arbitrum (`arbBlockNumber` vs `eth_blockNumber` distinction).
- Nitro genesis block height on Arbitrum One (claimed 22207817).
- Existence and maturity of Rust types crate equivalent to `op-alloy` for Arbitrum.

Verify against the Nitro source, the official docs, and the on-chain `ArbOwner.getChainConfig` / `ArbSys.arbosVersion()` queries before promoting this doc to `deep`.
