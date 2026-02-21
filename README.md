# 8004 Collection Extension

Compact, indexer-friendly collection extension for the [ERC-8004](https://eips.ethereum.org/EIPS/eip-8004) Agent Registry across Solana and EVM chains.

## Overview

This specification enables agents registered in 8004 registries to join extension collections via a single metadata key (`col`). Collections are identified by a deterministic key derived from the creator's CAIP-10 address and a CID pointing to collection metadata on IPFS.

Key properties:

- **No on-chain changes**: uses existing `setMetadata` on 8004 registries
- **First-write-wins**: `col` is immutable after first valid set (prevents collection drift)
- **Deterministic identity**: `collection_key = creator_snapshot_caip10|cid_norm`
- **Cross-chain**: Solana and EVM with CAIP-2/CAIP-10/CAIP-19 standards
- **Indexer-enforced**: immutability is an indexer policy, not an on-chain constraint

## How It Works

1. Creator publishes a collection JSON document to IPFS (name, image, socials)
2. Agent owner calls `setMetadata("col", "c1:<cidv1_base32>")` on the 8004 registry
3. Indexer ingests the event, normalizes the CID, derives `collection_key`, and locks membership
4. Once locked, the agent's collection membership is immutable in V1

## Files

| File | Description |
|------|-------------|
| [SPEC.md](./SPEC.md) | Full specification (V1) |
| [collection-pointer-value-v1.schema.json](./collection-pointer-value-v1.schema.json) | JSON Schema for `col` metadata value |
| [collection-cid-document-v1.schema.json](./collection-cid-document-v1.schema.json) | JSON Schema for collection CID document |

## Example

```
# Agent metadata write
key:   "col"
value: "c1:bafybeigdyrzt5sfp7udm7hu76uh7y26nf3efuylqabf3oclgtqy55fbzdi"

# Derived collection key
creator: solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp:HvF3Jqhah...
collection_key: solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp:HvF3Jqhah...|bafybeigdyrzt5sfp7udm7hu76uh7y26nf3efuylqabf3oclgtqy55fbzdi
```

## Related Repositories

- [8004-solana](https://github.com/QuantuLabs/8004-solana) — Solana registry programs (Anchor)
- [8004-solana-ts](https://github.com/QuantuLabs/8004-solana-ts) — TypeScript SDK ([npm](https://www.npmjs.com/package/8004-solana))
- [8004-solana-indexer](https://github.com/QuantuLabs/8004-solana-indexer) — Solana indexer service

## Community

- [Twitter/X](https://x.com/Quantu_AI)
- [Telegram](https://t.me/sol8004)
- [EIP-8004 Standard](https://eips.ethereum.org/EIPS/eip-8004)

## License

MIT
