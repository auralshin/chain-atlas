# Architecture

ZK-EVM rollup architecturally near-identical to [Scroll](../scroll/architecture.md). Read the Scroll architecture doc first; the Linea-specific deltas are below.

## Components

| Component | Linea-specific |
|---|---|
| **Sequencer** | Operated by Consensys |
| **Prover** | Generates Vortex/PLONK-derived ZK proofs |
| **Execution client** | `linea-besu` (Besu fork). Sequencer side may run `linea-geth`. |
| **L1 contracts** | `ZkEvm.sol` (state commitments + verifier), `MessageService` (L1↔L2 messaging), `TokenBridge` |
| **L2 system contracts** | Linea-specific predeploys for L2 messaging and L1 fee oracle |

The execution-client distinction (Besu vs geth) is mostly invisible to indexers — both expose standard ETH JSON-RPC. It matters for self-hosters and for some rare tracer differences.

## Block production

| Parameter | Value |
|---|---|
| L2 block time | ~2 s {{unsourced: confirm — has been tuned}} |
| Sequencer | Single, Consensys |
| Batch / proof cadence | Every few minutes |
| Settlement layer | Ethereum L1 |

Sequencer publishes L2 blocks immediately. Batches are committed to L1 via `ZkEvm.sol`'s commit function. Proofs are submitted to finalize batches.

## Two-stage L1 finality

Same as Scroll:

| Stage | What's settled |
|---|---|
| **Committed** | Batch data + claimed state root on L1 |
| **Finalized** | ZK proof verifies the state transition; canonical |

L1 events: `BlockFinalized` (or similar; verify event names on `ZkEvm.sol`) {{unsourced}}.

## L1 → L2 messaging

User calls `MessageService.sendMessage` on L1. The message is queued; the sequencer includes it in a future L2 block. L2-side execution is via Linea's L2 `MessageService` predeploy.

Same pattern as Scroll's `L1ScrollMessenger` / `L2ScrollMessenger`, with different addresses.

## L2 → L1 messaging

L2 contracts call `L2 MessageService.sendMessage`. After the batch containing the message is finalized, users can claim on L1 via `MessageService.claimMessage` (or a chain-specific equivalent) {{unsourced: confirm method name}} with a Merkle proof.

No challenge window — claimable immediately after batch finalization.

## Type-2 zkEVM consequences

Linea targets "Type 2" zkEVM equivalence (per Vitalik's classification): EVM-equivalent at the spec level **except** for some gas costs. For indexers:

- **Bytecode is the same** — same Solidity, same compiled bytecode, deploys identically.
- **Execution semantics are the same** for normal Solidity contracts.
- **Gas costs differ** for some opcodes — affects fee accounting if you simulate execution off-chain.
- **Some opcodes may be gated** — verify which Cancun-equivalent opcodes are active per fork.

For most application indexers this is invisible. For analytics that simulate gas use (like estimating "what if I batched this differently"), use the actual Linea node.

## Fee model and burn

Linea has a **fee burn** mechanism (similar in spirit to BEP-95 on BSC or EIP-1559 on Ethereum but Linea-specific): a portion of fees collected is burned, contributing to ETH deflationary pressure on Ethereum L1 indirectly {{unsourced: confirm exact mechanism}}.

Indexer impact:
- Receipts and gas accounting are standard.
- "Total fee revenue collected" ≠ "Total fee revenue retained by sequencer/protocol" — burn is the difference.
- For supply / circulating supply tracking, account for the burned portion.

## Where the data lives

Same split as Scroll: `linea-besu` for L2 EL data, L1 contracts for batch lifecycle and bridge events. No separate CL component.

A complete Linea indexer reads L1 + L2 in lockstep (same pattern as every rollup in this guide).
