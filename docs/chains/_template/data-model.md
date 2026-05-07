# Data Model

## Block

<Header fields, body structure, size limits. Anything an indexer extracts.>

## Transaction

<Tx types, signature scheme, fee model. **List every tx type** — many chains add custom ones that break naive RLP parsers. EIP-2718 typed-tx envelopes, blob txs, custom L2 deposit/system txs, etc.>

## Receipt / Execution result

<Per-tx output: status, gas used, logs, return data, side effects, state diffs if exposed.>

## Logs / Events

<Event format, indexed fields, topic structure. For non-EVM: how do programs emit events and how does an indexer subscribe?>

## State

<Storage layout. For EVM: account → storage trie. For others: describe the analog (objects, resources, accounts-with-data).>

## Special data

<Blobs (EIP-4844), compressed calldata, off-chain commitments, blob KZG proofs, sequencer batches in calldata vs blobs, anything that requires non-RPC retrieval.>
