# Architecture

## Consensus

<Algorithm. Validator/sequencer set composition. Slot/epoch structure if any. Slashing conditions if relevant.>

## Block production

<Who proposes blocks, on what cadence, with what guarantees. Single sequencer? Rotating proposer? PoW miner?>

## Fork choice

<How the canonical chain is selected. LMD-GHOST, longest-chain, single-sequencer, fork-choice-by-finality-gadget, etc.>

## Finality model

<Probabilistic | single-slot | sequencer-confirmed | L1-finalized | hybrid. Be precise about the difference between "tx is in a block" and "tx will never be reorged". Cross-link to [`docs/concepts/finality-models.md`](../../concepts/finality-models.md).>

## State model

<Account-based | UTXO | object-centric | resource. How state is committed, proven, and pruned.>

## Where the data physically lives

<Single chain, or split between L1 and L2? Data availability layer separate from execution? Off-chain components an indexer must talk to (sequencer feed, batcher, prover)?>
