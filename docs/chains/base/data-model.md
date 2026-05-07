# Data Model

**Identical to Optimism.** Same block structure, same tx types (including deposit `0x7E`), same receipt fields, same predeploys at the same `0x4200…` addresses.

See [optimism/data-model.md](../optimism/data-model.md).

## What's different on Base

| Aspect | Base | Optimism |
|---|---|---|
| Chain ID (in tx signing domain) | 8453 | 10 |
| L1-aliasing offset | Same | Same |
| Predeploy bytecode at fork activations | Tracks Base's fork timeline (see [forks-changelog.md](forks-changelog.md)) | Tracks OP's fork timeline |
| Pre-genesis data | None | OVM 1.0/2.0 |

## L1 origin

L1 origin block hashes referenced from Base's L2 blocks point to Ethereum mainnet (same L1 as OP). Indexers tracking L1 origin per L2 block use the same code as Optimism's; only the L1 contract addresses to monitor differ.

## Genesis block

Base genesis is a Bedrock-format block at the chain's launch. There is no `genesis.l2.l1` discontinuity to handle — unlike OP Mainnet, the chain has been Bedrock from block 0 (or 1, depending on how you count the genesis allocation).

Pull the actual genesis hash from the superchain-registry's Base config rather than hardcoding.

## Tx types active

Same set as Optimism for any given timestamp. Base's fork timeline (see [forks-changelog.md](forks-changelog.md)) determines when EIP-7702 (`0x04`) becomes available — likely later than Optimism due to Base typically activating upgrades a few days behind OP Mainnet.
