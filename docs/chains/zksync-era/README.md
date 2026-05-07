# zkSync Era

**Status:** stub
**Family:** ZK rollup — non-bytecode-equivalent EVM (custom VM)
**Mainnet launch:** 2023-03-24
**Native node clients:** zksync-era (Matter Labs)

## Anchor facts (verify in deep dive)

- Native account abstraction — there are no EOAs in the Ethereum sense.
- Custom transaction types (EIP-712-style) that break naive ETH transaction parsers.
- Paymasters are first-class — fee-payer ≠ sender is the common case, not the exception.
- Bytecode is *not* identical to EVM — Solidity compiles via a `zksolc` toolchain. Some opcodes behave differently.
- Hardest EVM-adjacent chain to index correctly — most indexers built only for Ethereum mis-decode tx types.

## To populate

Use [template](../_template/). Tx-type and AA decoding is the headline section.
