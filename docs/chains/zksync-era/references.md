# References

## Source code

- **`zksync-era`** — [github.com/matter-labs/zksync-era](https://github.com/matter-labs/zksync-era) — sequencer, prover, RPC. The single source of truth for L2 behavior.
- **`era-contracts`** — [github.com/matter-labs/era-contracts](https://github.com/matter-labs/era-contracts) — L1 + L2 system contracts, including `Bootloader`, `ContractDeployer`, `NonceHolder`, `L1Messenger`, `Bridgehub`, `ExecutorFacet`.
- **`zksolc`** — [github.com/matter-labs/era-compiler-solidity](https://github.com/matter-labs/era-compiler-solidity) — Solidity → EraVM compiler.
- **`zkvyper`** — Vyper → EraVM compiler. Smaller usage.

## Type libraries

- **`zksync-types`** (Rust) — [github.com/matter-labs/zksync-era/tree/main/core/lib/types](https://github.com/matter-labs/zksync-era/tree/main/core/lib/types) — canonical Rust types for Era. Use this rather than rolling your own.
- **`zksync-web3`** (TypeScript) — [npmjs.com/package/zksync-web3](https://www.npmjs.com/package/zksync-web3) — TS SDK; useful as reference for the EIP-712 format if porting to Rust.
- **`zksync-go`** (Go) — community / Matter Labs Go SDK.

`alloy` does **not** support type 0x71 out of the box. Use `zksync-types` to deserialize it correctly.

## Documentation

- **zkSync docs** — [docs.zksync.io](https://docs.zksync.io/) — covers AA, paymasters, system contracts, EraVM, RPC.
- **Block explorer + dev portal** — [explorer.zksync.io](https://explorer.zksync.io/) and [dev.zksync.io](https://dev.zksync.io/) {{unsourced: confirm domains}}.

## Block explorers

- **zkSync Era explorer (Matter Labs)** — [explorer.zksync.io](https://explorer.zksync.io/) — official; exposes paymaster info, AA signer, batch status.
- **zkScan** — [zkscan.io](https://zkscan.io/) {{unsourced: confirm if active}} — alternative.

## Operational

- **Status page** — [status.zksync.io](https://status.zksync.io/) {{unsourced: confirm URL}}.
- **L2BEAT zkSync page** — [l2beat.com/scaling/projects/zksync-era](https://l2beat.com/scaling/projects/zksync-era) — independent monitor; tracks finality state, batch posting cadence.

## Where this doc is incomplete

The following claims carry `{{unsourced}}` markers and need primary-source backing:

- Current L2 block time (claimed ~1 s; was higher historically).
- Exact protocol version → date mapping for v1–v25+.
- Boojum activation protocol version.
- Bridgehub introduction protocol version.
- Blob DA activation protocol version.
- Execution delay between `prove` and `execute` on L1 (claimed ~24 hours).
- Whether Era recognizes EIP-7702 transactions at all.
- Force-include path for delayed inbox messages (whether and how an L1 priority op bypasses sequencer).
- Specific opcode semantics differences from EVM (`SELFDESTRUCT`, `BLOCKHASH`, etc.).
- Latest `zksync-types` Rust crate version.
- Default account upgrades (when the standard ECDSA-validation contract was upgraded and what changed).
- Whether `zks_getBlockByNumber("safe"/"finalized")` block tags map to L1-committed / L1-executed exactly.

Verify against the `zksync-era` source, the `era-contracts` releases, and the Matter Labs CHANGELOG before promoting this doc to `deep`.
