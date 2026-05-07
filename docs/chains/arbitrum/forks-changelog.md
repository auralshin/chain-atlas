# Forks and Changelog

Arbitrum has two semi-independent versioning axes: **Nitro stack version** (the binary's release line) and **ArbOS version** (the consensus-relevant L2-OS layer baked into Nitro). Indexers care about the latter.

## Eras

| Era | Span | Indexer impact |
|---|---|---|
| **Classic** | 2021-08-31 → 2022-08-31 | AVM-based, distinct data model. **Out of scope for this guide.** |
| **Nitro** | 2022-08-31 → present | EVM-equivalent. **This guide covers Nitro+ only.** |

The Classic→Nitro migration on Arbitrum One occurred at L2 block height **22207817** {{unsourced: confirm}}. Most modern indexers ignore Classic data or import a one-time snapshot at the migration height.

## ArbOS versions

ArbOS is the consensus-layer "operating system" that runs inside the Nitro EL. Each version bumps a chain-wide parameter set and may enable new precompile methods, change gas pricing, or alter tx-type behavior.

| ArbOS | Activation on Arbitrum One | Notes |
|---|---|---|
| **6** (Classic) | Pre-Nitro | Classic-era; out of scope. |
| **7** | 2022-08-31 (Nitro genesis) | First Nitro version. |
| **9** | {{unsourced: 2022-Q4}} | Minor fixes. |
| **10** | {{unsourced: 2023-Q1}} | Gas pricing tweaks. |
| **11** | {{unsourced: 2023-08}} | Shanghai-equivalent: `PUSH0`, EVM upgrades. |
| **20** | {{unsourced: 2024-Q1}} | Cancun-equivalent: blob support on L1 (batches as blobs), transient storage, MCOPY, beacon root precompile. |
| **30** | {{unsourced: 2024}} | **Stylus activation** — WASM contracts go live. |
| **31** | {{unsourced: 2024-late}} | Stylus refinements; bug fixes. |
| **32** | {{unsourced: 2025}} | TBD post-BoLD; Pectra-equivalent porting expected. |

The canonical, per-chain activation timestamps are in [`OffchainLabs/nitro/util/headerreader/upgrade.go`](https://github.com/OffchainLabs/nitro) {{unsourced: confirm path}} and in each chain's onchain configuration via `ArbOwner.getChainParameters` and `ArbOwner.getChainConfig`. Pull at runtime, do not hardcode.

### Per-chain ArbOS timing differs

Arbitrum Nova and the various Orbit chains run on their own ArbOS upgrade schedules. **Do not assume Arbitrum One's ArbOS activation timestamps apply to other Arbitrum chains.**

## DA upgrades

### Arbitrum One: calldata → blobs

Pre-EIP-4844 era: batches posted as L1 calldata. Post-Ecotone activation on Arbitrum One: batches as EIP-4844 blobs {{unsourced: confirm Arbitrum One blob activation date}}.

Indexer impact: a batch decoder must handle **both** modes. The `SequencerInbox` event tells you which mode was used per-batch.

### Arbitrum Nova: AnyTrust DAC

Nova has **never** posted full data to L1 calldata in normal operation. AnyTrust DAC mode is the default; L1 calldata is the fallback when DAC fails. Indexers for Nova must integrate with the DAC's REST API to fetch batch contents.

## BoLD rollout (legacy → BoLD fault proofs)

| Phase | Status |
|---|---|
| **Legacy interactive challenges** | Originally permissioned-validator. Active on Arbitrum One since Nitro genesis. |
| **BoLD on testnet** | 2024 testnet activation {{unsourced}}. |
| **BoLD on Arbitrum One mainnet** | Late 2024 / 2025 {{unsourced: confirm}}. Permissionless. |
| **BoLD on Nova** | TBD. |

During the BoLD migration window, an indexer must read **both** sets of L1 contract events (`RollupProxy` legacy assertions AND `EdgeChallengeManager` BoLD edges) to get a complete finality picture.

## Stylus rollout

Stylus enables WASM smart contracts (Rust, C, C++, Sway) on Arbitrum chains. ArbOS 30 added the activation precompile (`ArbWasm.activateProgram`); ArbOS 31 added refinements.

Indexer impact:
- **Tx interface unchanged** — Stylus contracts are called via standard `eth_call` / regular txs.
- **Contract code is WASM after activation**, not EVM bytecode. `eth_getCode` returns the bytecode header indicating WASM; reading further requires understanding the activated-program format.
- **Decompilation, ABI inference, and source verification tools differ** for Stylus contracts. Most indexer-side ABI flows still work because Solidity ABI conventions are reused.

## What an indexer must do per upgrade

1. **Pin the ArbOS activation timestamps** at startup from the chain's onchain config or from the `nitro` source.
2. **Version receipt and tx decoders** by ArbOS version (mostly stable across versions, but blob-DA at ArbOS 20+ changes batch decoding).
3. **Rerun anything dependent on gas pricing** if the ArbOS version changed — fee parameters change.
4. **Add BoLD-aware finality detection** alongside legacy assertion tracking during migration.
5. **Backfill nothing on Classic** unless explicitly needed.
