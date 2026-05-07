# Linea

**Status:** in-progress (template populated — pending review)
**Family:** ZK-EVM rollup (Consensys)
**Mainnet launch:** 2023-07-11 (alpha) → public availability 2023-08
**Native node clients:** [`linea-besu`](https://github.com/Consensys/linea-besu) (Besu fork) — also a `linea-geth` for sequencer-side execution
**Chain ID:** 59144

## TL;DR for indexers

- **ZK-EVM rollup, "Type 2" equivalence target.** EVM-bytecode-equivalent for application contracts. Some gas costs differ from Ethereum.
- **Same shape as [Scroll](../scroll/).** L1 commit + finalize, L1 fee receipt fields, L1↔L2 messaging via Linea's `MessageService`, single sequencer.
- **Closest cousin in this guide is Scroll.** Cross-link aggressively.
- **Standard tx types** (0/1/2; 0x04 EIP-7702 if/when ported). No system tx types.
- **Single sequencer**, operated by Consensys. Decentralization on the roadmap.

## Read order

If you've read [scroll/](../scroll/), the only Linea files with substantively different content are:

1. [forks-changelog.md](forks-changelog.md) — Linea-specific upgrades (Beta v1, Beta v2, Pectra-equivalent).
2. [gotchas.md](gotchas.md) — Linea-specific quirks (fee burn, ETH-deflationary mechanics, Type-2 zkEVM gas differences).
3. [rpc-surface.md](rpc-surface.md) — `linea_*` namespace, including the `linea_estimateGas` extension.
4. [storage.md](storage.md) — what changes from the Scroll schema (almost nothing).
5. [references.md](references.md) — Linea-specific links.

The rest are short deltas pointing to scroll/.

## What's *not* different from Scroll

| Aspect | Status |
|---|---|
| Standard tx types only | Same |
| Block-level fields | Same shape (no Linea-specific block additions in JSON-RPC) |
| Receipt L1-fee fields | Same shape (`l1Fee`, `l1GasUsed`, etc.) |
| Two-stage L1 finality (commit → finalize) | Same |
| L1↔L2 messaging via dedicated contracts | Same |
| Validation: ZK proof, no fault proofs | Same |
| Reorg model | Same (sequencer rewinds + L1 reorg) |

## What *is* different

| Aspect | Linea | Scroll |
|---|---|---|
| Execution client | `linea-besu` (Besu fork) — distinct from Scroll's geth fork | `l2geth` (geth fork) |
| L1 contracts | `ZkEvm.sol`, `MessageService`, `TokenBridge` | `ScrollChain`, `L1ScrollMessenger`, `L1MessageQueue` |
| Predeploy addresses | Linea-specific {{unsourced: confirm}} | Scroll-specific |
| Proof system | Vortex / PLONK-derived {{unsourced: confirm}} | Halo2-based |
| Type-2 zkEVM | Some opcode gas costs differ from Ethereum | Bytecode-equivalent |
| Fee burn (BEP-95-style) | 20% of fees burned (Linea's deflationary mechanic) {{unsourced: confirm}} | None |
