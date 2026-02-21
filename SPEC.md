# 8004 Collection Extension Specification

**Version**: 1.0.0
**Status**: Draft
**Date**: 2026-02-16

## Guidance Scope

This document is a V1 reference profile intended to guide implementations toward interoperable behavior. Implementations may diverge internally, provided they preserve deterministic collection identity and cross-indexer compatibility.

---

## 1. Overview

This specification defines a compact and indexer-friendly extension for ERC-8004 collections across Solana and EVM chains. The design aims to satisfy four constraints: no change to core 8004 registries, minimal on-chain footprint, deterministic indexing across independent indexers, and compatibility with existing SDK patterns that use metadata key/value writes.

V1 defines a single native flow: **collection pointer on agent metadata**. An agent joins a collection by writing a single metadata entry that references a content-addressed document on IPFS. The indexer enforces immutability on this pointer after the first valid write.

> **Informational note (out of scope for V1):** explorer and marketplace implementations may later support creator-signed display overrides for UI fields, without changing canonical identity fields (`collection_key`, `creator_snapshot_caip10`, `cid_norm`) or lock state.

---

## 2. Data Model

An agent participating in extension collections SHOULD store one collection pointer metadata entry with key `"col"` and a value in the format `"c1:<cid_norm>"`, where `cid_norm` is the canonical CID string in CIDv1 base32 lowercase.

### 2.1 Pointer Value JSON Schema

The canonical schema that validates the raw metadata value format for `col` is published at:

- `https://raw.githubusercontent.com/QuantuLabs/8004-collection-extension/main/collection-pointer-value-v1.schema.json`
- Repository path: `./collection-pointer-value-v1.schema.json`

A valid pointer value looks like `c1:bafybeigdyrzt5...`. Values such as `ipfs://bafy...` or raw CIDv0 strings like `Qm...` are not valid storage formats and must be normalized before being written on-chain.

SDKs and indexers SHOULD accept both CIDv0 (base58) and CIDv1 input and normalize to CIDv1 base32 lowercase before storage or comparison. Implementations MUST perform this normalization after CID parsing, not by lowercasing the raw input string. CIDv0 uses base58 encoding, which is case-sensitive; lowercasing before parsing would destroy CIDv0 input and produce an incorrect result.

The pointer value (including the `c1:` prefix) SHOULD be between 62 and 256 characters after normalization. A valid CIDv1 base32 string with sha2-256 is at least 59 characters long; with the `c1:` prefix the minimum total length is 62.

### 2.2 CID Target Document JSON Schema

The CID referenced by the `col` pointer should resolve to a UTF-8 JSON document that describes the collection. The canonical schema for this document is published at:

- `https://raw.githubusercontent.com/QuantuLabs/8004-collection-extension/main/collection-cid-document-v1.schema.json`
- Repository path: `./collection-cid-document-v1.schema.json`

The recommended minimum document shape is:

```json
{
  "version": "1.0.0",
  "name": "My Collection",
  "symbol": "COLL",
  "description": "Optional description",
  "image": "ipfs://bafy...",
  "banner_image": "ipfs://bafy...",
  "socials": {
    "website": "https://example.com",
    "x": "@mycollection",
    "discord": "https://discord.gg/mycollection"
  }
}
```

The collection document is intentionally minimal to avoid duplicating agent-level information already present in `agentURI` data. The `version` field identifies the document schema profile, `symbol` is optional display metadata, and `socials` is the recommended container for social and account links.

The schema allows additional properties for forward compatibility. However, UIs MUST sanitize ALL fields from CID documents before rendering, including the defined schema fields (`name`, `symbol`, `description`, and `socials` values). Raw HTML rendering of any field is prohibited. URI-like fields (`image`, `banner_image`, `socials.website`) MUST be validated against a safe scheme allowlist: only `ipfs://`, `https://`, and `ar://` are permitted. Schemes such as `javascript:`, `data:`, `vbscript:`, and `blob:` MUST be rejected. Any additional properties beyond the defined schema MUST NOT be rendered without sanitization, to prevent stored XSS via IPFS-hosted documents.

### 2.3 Asset Identifier Format

To prevent cross-registry collisions, indexers SHOULD store the `asset` field in a canonical CAIP-19-style format that includes the chain identifier, registry identity, and agent local ID. The recommended format is:

```
<chain_id_caip2>/erc8004:<registry_ref>/<agent_local_id>
```

On EVM chains, the `agent_local_id` is the `uint256` token ID expressed as a decimal string (e.g., `241`). On Solana, it is the Base58 string of the Agent Asset Account address (e.g., `7YQq9mVn7f8w4t6J8h2K3L5mN7P9rS2uV4xY6zA1bC3`).

Full examples:

- `eip155:8453/erc8004:0x8004a6090Cd10A7288092483047B097295Fb8847/241`
- `solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp/erc8004:8rW9y8Xj3nL4QvGm2Qf3k8uYxKfJ2T9b6hVnP3Lm9Q4/7YQq9mVn7f8w4t6J8h2K3L5mN7P9rS2uV4xY6zA1bC3`

---

## 3. Chain and Creator Identity (CAIP-10)

### 3.1 Chain ID Format

The recommended external chain identifier format is **CAIP-2**. For EVM chains, this takes the familiar `eip155:<chain_id>` form (e.g., `eip155:1` for Ethereum mainnet, `eip155:8453` for Base mainnet).

For Solana, the CAIP-2 namespace reference is derived from the genesis hash. Specifically, it is the first 32 characters of the Base58 string returned by `getGenesisHash`. This rule is string-based and deterministic: read the genesis hash as Base58 text, take the first 32 characters, and do not reinterpret the bytes or re-encode them. This follows the Chain Agnostic Solana namespace profile.

Reference values:

| Chain | CAIP-2 Identifier |
|-------|-------------------|
| EVM Mainnet | `eip155:1` |
| Base Mainnet | `eip155:8453` |
| Solana Mainnet | `solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp` |
| Solana Devnet | `solana:EtWTRABZaYq6iMfeYKouRu166VU2xqa1` |
| Solana Testnet | `solana:4uhcVJyU9pJkvQyS88uRDiswHXSCkY3z` |

### 3.2 Creator Snapshot Format

The recommended creator identifier format is **CAIP-10** account IDs, which combine the CAIP-2 chain identifier with the account address:

```
<caip2_chain_id>:<account_address>
```

To ensure deterministic `collection_key` derivation across indexers, EVM account addresses in `creator_snapshot_caip10` and `asset` fields MUST be stored using EIP-55 mixed-case checksum encoding. Indexers MUST apply EIP-55 checksumming to all EVM addresses before storage and comparison.

Examples:

- `eip155:8453:0x742d35Cc6634C0532925a3b844Bc9e7595f8fE43`
- `solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp:HvF3JqhahcX7JfhbDRYYCJ7S3f6nJdrqu5yi9shyTREp`

### 3.3 Creator Snapshot Source

The `creator_snapshot_caip10` is an indexer-derived field (not an on-chain event field) that captures the identity of the address that first registered the agent. It should be treated as immutable once set. On Solana, it is derived from the `owner` field of the first `AgentRegistered` event. On EVM, it comes from the `owner` field of the first `Registered` event. Implementations should never use the current owner after transfers for creator snapshot derivation.

Factory and minter semantics are handled explicitly in this profile:

1. The `creator_snapshot_caip10` always reflects the first registration owner emitted by the registry flow. If a factory, launchpad, or relayer is the first registration owner, that address becomes the creator namespace.
2. Projects that need user-scoped creator namespaces SHOULD ensure the end user is the first registration owner. This can be achieved by having the factory transfer ownership in the same transaction before the `Registered` event, or by delegating registration to the end user directly.
3. Indexers SHOULD treat this field as a provenance namespace, not as a verified brand identity signal.
4. UIs MUST display `creator_snapshot_caip10` prominently alongside the collection display name. Two collections with the same CID but different creators are distinct collections and MUST be visually distinguishable.

Wallet rotation behavior is intentional in this profile. If a project registers new agents from a different wallet, those agents will have a different `creator_snapshot_caip10`, and therefore a different `collection_key` even when pointing to the same CID. Projects that require a single continuous collection namespace SHOULD keep a stable registration wallet.

---

## 4. Collection Key Derivation

The collection key is derived deterministically from the creator's identity and the normalized CID:

```
collection_key = <creator_snapshot_caip10>|<cid_norm>
```

The pipe character `|` is used as separator because the colon `:` is already used inside CAIP identifiers.

Indexer implementations should persist both the human-readable `collection_key` and the indexed tuple fields (`creator_snapshot_caip10`, `cid_norm`) for efficient querying.

Because the `collection_key` includes `cid_norm`, **collections are immutable snapshots**. Updating the collection JSON document (name, image, description) produces a new CID and therefore a new `collection_key`. Existing agents remain locked to the original CID; only newly enrolled agents will point to the updated document. This is by design: it prevents silent metadata rug-pulls on existing collection members.

Projects that need to update display metadata without fragmenting their collection SHOULD use an explorer or marketplace display override layer (out of scope for V1) or plan collection versions as intentional "seasons" with documented CID lineage.

---

## 5. Indexer Specification

### 5.1 Event Inputs

Indexers should consume the following event types:

1. **Registration events**: `AgentRegistered` on Solana and `Registered` on EVM, using the `owner` field to derive the creator snapshot.
2. **Metadata set events**: `MetadataSet`, which carry the key/value pair written by the agent owner.
3. **Metadata delete events**: `MetadataDeleted`, where available. Not all 8004 registries emit this event. Indexers that do not have access to delete events SHOULD treat their absence as acceptable, since lock state is unaffected (deletes are rejected for locked memberships).
4. **Deactivation and burn signals**: on EVM, an NFT transfer to the zero address; on Solana, the chain-equivalent burn or close signal if exposed by the indexing source.

### 5.2 Canonical Ordering and Finality

V1 collection immutability is an indexer policy. The core on-chain registries and programs remain unchanged and may still emit later metadata updates. Indexers MUST use a canonical event ordering and MUST NOT rely on websocket or network arrival order.

**Solana canonical source and order:**

The canonical source SHOULD be `getBlock` with `commitment=finalized`. Events from failed transactions (`meta.err != null`) MUST be ignored. The canonical sort key MUST be `(slot, tx_index, log_index)` where `slot` is the block slot, `tx_index` is the transaction's position within the block's `transactions` array, and `log_index` is the zero-based ordinal position of the event's log message within the full `meta.logMessages` array. This count includes all log entries (program invocation logs, return logs, and inner instruction logs), not only 8004-specific events. Using the full array ordinal ensures deterministic ordering even when a single transaction emits multiple `MetadataSet` events via cross-program invocation (CPI).

The flat `log_index` approach is preferred over instruction-based tracking (`instruction_index`, `inner_instruction_index`) because a single CPI instruction can emit multiple events, and reconstructing the CPI call stack from log strings is fragile. The flat array index from finalized `getBlock` data is immutable and deterministic.

**Solana runtime log truncation:** the Solana runtime enforces a per-transaction log byte limit. If a transaction exceeds this limit, the runtime silently truncates `meta.logMessages` and appends a `"Log truncated"` entry. Events emitted after the truncation point will not appear in the log array. Indexers relying on log-based event parsing SHOULD detect the `"Log truncated"` sentinel and flag affected transactions for manual review or fallback to account state diff analysis.

**EVM canonical source and order:**

The canonical source SHOULD be finalized head data. The canonical sort key MUST be `(block_number, transaction_index, log_index)`, where `log_index` is the block-scoped log index from RPC logs.

First-write-wins MUST be evaluated using the canonical sort key on canonical chain history.

### 5.3 Reorg and Rollback Policy

If an implementation ingests non-finalized data for lower latency, it MUST support rollback. Any event later marked non-canonical MUST be reverted, including any previously locked `col` state derived from orphaned events. After rollback, lock resolution MUST be recomputed from canonical history using canonical sort order.

Rollback handling SHOULD be driven by chain-native indicators: on EVM, canonical block hash tracking and `removed=true` logs when available; on Solana, finalized/root progression replacing pre-finalized views. Indexers SHOULD compare the stored `lock_block_hash` against the canonical chain to detect if a lock event was orphaned. If a mismatch is found, the lock MUST be reverted and recomputed from canonical history.

### 5.4 Ingestion Rules

V1 intentionally does not define any unlock, detach, or admin-override flow for `col`.

**On metadata set where `key == "col"`:**

1. Match the key using exact, case-sensitive equality: `key == "col"`.
2. Decode the value as UTF-8.
3. Trim leading and trailing ASCII whitespace (bytes `0x09`, `0x0A`, `0x0D`, `0x20`) from the decoded value. Non-ASCII whitespace (e.g., NBSP `0xC2A0`, zero-width space) is not trimmed and will cause CID parsing to fail, which is the intended behavior.
4. Validate that the value starts with the prefix `c1:` (case-insensitive: accept `c1:`, `C1:`, etc.).
5. Extract the CID portion (everything after `c1:`).
6. Parse the CID string. The parser should accept both CIDv0 (base58) and CIDv1 in any base or case. If parsing fails, treat the value as malformed input.
7. Normalize the parsed CID to CIDv1 base32 lowercase to produce `cid_norm`. This step handles CIDv0-to-v1 conversion and uppercase-to-lowercase normalization without destroying case-sensitive encodings like base58.
8. Load the `creator_snapshot_caip10` for this asset.
9. Compute the `collection_key` from `creator_snapshot_caip10` and `cid_norm`.
10. If no existing locked membership exists for `(chain_id_caip2, asset)`, create the membership and lock it.
11. If an existing locked membership has the same `cid_norm`, treat as idempotent (no membership change, event type `SET_NOOP`).
12. If an existing locked membership has a different `cid_norm`, reject the mutation and keep the original locked membership (event type `SET_REJECTED_LOCKED`).
13. Append a history event.

**Operator and delegate risk:** if the underlying 8004 registry allows operators or delegates (e.g., ERC-721 `setApprovalForAll`, Metaplex delegates) to call `setMetadata`, a temporary operator such as a marketplace or staking contract could set `col` on behalf of the owner. Because of first-write-wins, this permanently locks the agent's collection membership. Implementations aware of this risk SHOULD restrict `col` writes to owner-only at the registry level. If this is not possible, SDKs and UIs MUST warn owners that approving operators grants irrevocable `col` write authority.

**Enrollment semantics for pre-existing agents:**

An already-registered agent with no locked `col` may join a collection at any time via its first valid `setMetadata("col", "c1:<cid_norm>")`. The underlying registry's ownership rules apply to `setMetadata`; this profile does not add extra write authority. Once the first valid `col` is indexed, membership becomes immutable in V1. If the agent is no longer controlled by the original creator, the current owner controls whether and when the agent joins a collection.

**On metadata set where `key != "col"`:**

These events are ignored for collection-extension state.

**On metadata delete where `key == "col"`:**

If no locked membership exists, this is a no-op. If a locked membership exists, the delete is rejected and the original locked membership is preserved. A history event with type `DELETE_REJECTED_LOCKED` is appended.

**On malformed value:**

Malformed values are handled non-fatally. The indexer stores an `invalid_reason` and excludes the entry from default collection queries, but does not create or replace any locked membership. The agent remains eligible to set a valid `col` in a future transaction.

**On burn or deactivation signal:**

The membership is marked `active=false` for the given `(chain_id_caip2, asset)`. The locked identity fields (`creator_snapshot_caip10`, `cid_norm`, `collection_key`) remain unchanged. A history event is appended.

### 5.5 Recommended Tables

```sql
CREATE TABLE extension_collection_memberships (
  chain_id_caip2 TEXT NOT NULL,
  asset TEXT NOT NULL, -- canonical CAIP-19-style asset id
  creator_snapshot_caip10 TEXT NOT NULL,
  cid_norm TEXT NOT NULL,
  collection_key TEXT NOT NULL,
  col_locked BOOLEAN NOT NULL DEFAULT TRUE,
  lock_tx_hash TEXT,
  lock_block_number BIGINT,
  lock_block_hash TEXT,
  lock_block_timestamp TIMESTAMPTZ,
  lock_tx_index INTEGER,
  lock_log_index INTEGER,
  lock_slot BIGINT,
  lock_created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  active BOOLEAN NOT NULL DEFAULT TRUE,
  invalid_reason TEXT,
  first_seen_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  PRIMARY KEY (chain_id_caip2, asset)
);

CREATE TABLE extension_collection_membership_history (
  id BIGSERIAL PRIMARY KEY,
  chain_id_caip2 TEXT NOT NULL,
  asset TEXT NOT NULL, -- canonical CAIP-19-style asset id
  creator_snapshot_caip10 TEXT NOT NULL,
  cid_norm TEXT,
  collection_key TEXT,
  event_type TEXT NOT NULL, -- SET_LOCKED | SET_NOOP | SET_REJECTED_LOCKED | DELETE_REJECTED_LOCKED | INVALID | DEACTIVATE
  tx_hash TEXT,
  block_number BIGINT,
  block_hash TEXT,
  block_timestamp TIMESTAMPTZ,
  tx_index INTEGER,
  log_index INTEGER,
  slot BIGINT,
  removed BOOLEAN NOT NULL DEFAULT FALSE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_ext_collection_key
  ON extension_collection_memberships(collection_key)
  WHERE active = TRUE AND invalid_reason IS NULL;

CREATE INDEX idx_ext_collection_creator_cid
  ON extension_collection_memberships(creator_snapshot_caip10, cid_norm)
  WHERE active = TRUE AND invalid_reason IS NULL;

CREATE INDEX idx_ext_collection_cid
  ON extension_collection_memberships(cid_norm)
  WHERE active = TRUE AND invalid_reason IS NULL;
```

The `block_timestamp` fields SHOULD be populated from chain block header timestamps, not from ingestion wall-clock time.

### 5.6 CID Document Resolution (Non-Blocking)

Collection lock decisions MUST be based solely on pointer syntax and normalization (the `col` value). Resolving the pointed CID document to retrieve collection metadata MUST be performed asynchronously and MUST NOT block ingestion workers.

A failed or slow document fetch MUST NOT prevent lock state updates. Indexers SHOULD enforce bounded fetches with a recommended maximum document size of 64 KB and a fetch timeout of 10 seconds. Resolution failures SHOULD be stored as diagnostic state for UI and operational tooling, without mutating any locked membership identity.

Indexers SHOULD cache successfully resolved CID documents locally and serve cached content as a fallback when IPFS resolution fails or when the document is no longer pinned on any node. Because the CID is content-addressed, cached content is guaranteed to match the original as long as the CID matches.

---

## 6. Security and Trust Model

V1 uses native open tagging semantics, meaning any agent owner can set any CID as their collection pointer. This openness introduces several risks, each with corresponding mitigations:

| # | Risk | Mitigation |
|---|------|------------|
| 1 | **Sybil and spam collections**: anyone can create collections by publishing a CID. | Display the full `collection_key` (not only the display name) and apply indexer-level anti-spam controls. |
| 2 | **Brand impersonation**: two different creators can point to the same CID, creating two visually similar but distinct collections. | UIs MUST display `creator_snapshot_caip10` prominently alongside the collection name so users can distinguish them. |
| 3 | **No on-chain governance**: collection rules exist only at the indexer layer. | Add reputation and trust scoring for creators at the API and UI level. |
| 4 | **Creator wallet rotation**: registering agents from a new wallet creates a new creator namespace, fragmenting the collection. | Projects SHOULD keep a stable registration wallet. This is by design, not a protocol flaw. |
| 5 | **Irreversible first write**: human error on the first valid `col` write cannot be undone in V1. | SDKs and UIs SHOULD require explicit confirmation before the first `col` write. |
| 6 | **Collection identity fragmentation**: changing the collection JSON document produces a new CID and a new `collection_key`, splitting the collection across old and new CIDs. | Treat collection versions as intentional "seasons" with documented CID lineage, or use a display override layer. |
| 7 | **Operator and delegate griefing**: a temporary operator can irrevocably lock agents into a garbage collection via first-write-wins. | SDKs and UIs SHOULD warn owners that approving operators grants irrevocable `col` write authority. |
| 8 | **CID document availability**: if the document is unpinned from all IPFS nodes, display metadata is lost (collection identity is preserved but the UI degrades). | Indexers SHOULD cache CID documents locally as an availability fallback (see section 5.6). |
| 9 | **Mempool front-running**: on EVM chains, pending `col` transactions are visible in the public mempool, enabling attackers with operator or delegate permissions to front-run with a malicious collection pointer. On Solana, analogous attacks are possible via MEV and priority fees. | SDKs SHOULD use private transaction submission (e.g., Flashbots Protect, MEV-resistant RPCs) for `col` set operations when available. On Solana, SDKs SHOULD use priority fees for `col` transactions. |

An overarching mitigation is to keep immutable history for audit and dispute analysis (see the history table in section 5.5).

### 6.1 V2 Migration Path (Informational)

V1 intentionally does not define unlock, detach, or admin-override flows. Future versions may address this via several possible approaches: a new metadata key (e.g., `col_v2`) that supersedes `col` with updated semantics; a burn-and-remint flow where the agent NFT is destroyed and re-created to reset all extension state; or an indexer-level migration event where the creator signs an off-chain attestation linking old and new CIDs for the same logical collection. These paths are sketched here to preserve design space; none are normative in V1.

---

## 7. Validation Guidelines

The following guidelines represent the baseline checks that implementations should perform. They are numbered for reference but do not imply a required execution order.

1. The extension metadata key must be exactly `"col"`, matched with case-sensitive equality.
2. The metadata value must be decoded as UTF-8 and trimmed of ASCII whitespace before validation.
3. The value format must follow the `c1:<cid_norm>` pattern.
4. The CID portion must decode successfully using a standard CID parser.
5. The canonical storage form must use CIDv1 base32 lowercase.
6. Invalid entries must not halt ingestion; they should be stored with a reason and excluded from default queries.
7. The `creator_snapshot_caip10` must be derived from the first registration owner event, not from the current owner.
8. The `col` key must be treated as immutable after the first valid indexed value (first-write-wins).
9. On Solana, events from failed transactions (`meta.err != null`) must be ignored.
10. Canonical sort order (section 5.2) must be applied before evaluating first-write-wins.
11. If non-finalized ingestion is used, rollback handling must follow section 5.3.
12. The `asset` field must use canonical CAIP-19-style formatting.
13. The `block_timestamp` fields must come from chain block headers, not ingestion wall-clock time.
14. CID document resolution must be non-blocking and bounded (section 5.6).
15. Burn and deactivation signals must set `active=false` without mutating the locked identity fields.
16. CID normalization must happen after CID parsing, not by lowercasing the raw input. CIDv0 uses base58, which is case-sensitive and must be parsed before any case conversion (section 5.4, steps 5-7).
17. ASCII whitespace trimming must cover only bytes `0x09`, `0x0A`, `0x0D`, and `0x20`. Non-ASCII whitespace must not be trimmed.
18. The pointer value length after normalization must be between 62 and 256 characters.
19. UIs must sanitize all fields from CID documents before rendering. URI-like fields must only allow `ipfs://`, `https://`, and `ar://` schemes.
20. UIs must display `creator_snapshot_caip10` prominently to distinguish collections with the same CID but different creators.
21. On EVM, the `agent_local_id` must be a `uint256` decimal string. On Solana, it must be the Base58 asset account address.
22. On Solana, the `log_index` must be the zero-based ordinal position in the full `meta.logMessages` array, counting all entries.
23. EVM account addresses in CAIP-10 identifiers must use EIP-55 mixed-case checksum encoding for deterministic cross-indexer comparison.
24. Solana indexers should detect the `"Log truncated"` sentinel in `meta.logMessages` and flag affected transactions for review.

---

## 8. Conformance Test Vectors

The following test vectors cover the key behaviors that implementations must handle correctly. Each vector describes a scenario and its expected outcome.

**1. Solana intra-block conflict.** A block at slot S contains two valid `MetadataSet(col)` events on the same asset. Event A at `(S, tx_index=0, log_index=10)` sets `col=a`, and event B at `(S, tx_index=1, log_index=3)` sets `col=b`. The expected locked value is `col=a`, because event A has a lower `tx_index`.

**2. Solana failed transaction filter.** Event A appears in a transaction with `meta.err != null`, and event B appears later in a successful transaction. The expected locked value is derived from event B only, since failed transactions must be ignored.

**3. EVM intra-block conflict.** Block N contains two `MetadataSet(col)` events on the same asset. Event A at `(N, tx_index=2, log_index=15)` sets `col=a`, and event B at `(N, tx_index=7, log_index=40)` sets `col=b`. The expected locked value is `col=a`.

**4. EVM reorg rollback.** Canonical history temporarily includes block H1 with the first valid `col=a`. Block H1 is later replaced by canonical block H2 with the first valid `col=b`. After rollback and replay, the expected final locked value is `col=b`.

**5. Burn and deactivation handling.** An agent has an active locked membership. A burn or deactivation signal is ingested. The expected state is `active=false` with the locked `collection_key` and identity fields preserved.

**6. CIDv0 normalization.** An agent sets `col=c1:QmYwAPJzv5CZsnA625s3Xf2nemtYgPpHdWEz79ojWnPbdG`. The SDK or indexer parses this as CIDv0 and normalizes it to CIDv1 base32 lowercase. The expected `cid_norm` is the CIDv1 base32 equivalent of the original QmY... hash.

**7. Uppercase base32 input.** An agent sets `col=c1:BAFYBEIGDYRZT5...` (uppercase). The indexer normalizes to lowercase before storage. The expected result is acceptance with a normalized value of `bafybeigdyrzt5...`.

**8. Self-overwrite attempt.** An agent has a locked `col=c1:bafyA...` and attempts to set `col=c1:bafyB...` (a different CID). The expected result is rejection with a `SET_REJECTED_LOCKED` history event. The locked value remains unchanged.

**9. Factory creator snapshot.** A factory contract registers an agent. The `Registered` event has `owner=FactoryAddress`. The agent is later transferred to `UserAddress`. The expected `creator_snapshot_caip10` remains the factory address, not the user address.

**10. Two agents, same CID, same creator.** Agent A and Agent B both set `col=c1:bafySame...` and share the same `creator_snapshot_caip10`. Both agents should have identical `collection_key` values and belong to the same collection.

**11. Solana intra-transaction CPI conflict.** A single transaction calls `setMetadata(col)` twice via CPI on the same asset. Event A at `(S, tx_index=5, log_index=12)` sets `col=a`, and event B at `(S, tx_index=5, log_index=18)` sets `col=b`. The expected locked value is `col=a`, because it has the lower `log_index`.

**12. Idempotent re-set (same CID).** An agent has a locked `col=c1:bafyA...` and sets the same value again. The expected result is no state change with a `SET_NOOP` history event.

**13. Delete attempt on locked membership.** An agent has a locked `col=c1:bafyA...`. A `MetadataDeleted` event with `key == "col"` is ingested. The expected result is rejection with a `DELETE_REJECTED_LOCKED` history event. The locked value remains unchanged.

**14. Malformed value handling.** An agent sets `col=c2:bafybeig...` (wrong prefix). The expected result is that the value is treated as malformed, an `invalid_reason` is stored, no locked membership is created, and the agent can still set a valid `col` later.

**15. Two agents, same CID, different creator.** Agent A (creator `0xAlice`) and Agent B (creator `0xBob`) both set `col=c1:bafySame...`. The expected result is two different `collection_key` values, producing two distinct collections despite sharing the same CID.

**16. ASCII whitespace trimming.** An agent sets `col="\t  c1:bafybeigdyrzt5...\n"` (with tabs, spaces, and a newline). The expected result is that the value is trimmed to `c1:bafybeigdyrzt5...`, accepted, and locked.

**17. Non-ASCII whitespace rejection.** An agent sets `col="\u00A0c1:bafybeigdyrzt5..."` (with a leading NBSP character). The expected result is that the value is treated as malformed, because NBSP is not trimmed by ASCII whitespace rules and causes CID parsing to fail.

---

## 9. Reference Pseudocode

```ts
function trimAsciiWhitespace(s: string): string {
  return s.replace(/^[\x09\x0a\x0d\x20]+|[\x09\x0a\x0d\x20]+$/g, "");
}

function normalizeCollectionCid(input: string): string | null {
  try {
    const raw = trimAsciiWhitespace(input);
    const noScheme = raw
      .replace(/^ipfs:\/\//i, "")
      .replace(/^\/ipfs\//i, "");

    const cid = CID.parse(noScheme); // accepts v0 or v1, any case
    return cid.toV1().toString(); // CIDv1 base32lower is already lowercase
  } catch {
    return null;
  }
}

function encodeCollectionPointer(inputCid: string): string | null {
  const cidNorm = normalizeCollectionCid(inputCid);
  if (!cidNorm) return null;
  return `c1:${cidNorm}`;
}

function deriveCollectionKey(
  creatorSnapshotCaip10: string,
  cidNorm: string
): string {
  return `${creatorSnapshotCaip10}|${cidNorm}`;
}

function applyColSet(
  currentLockedCidNorm: string | null,
  incomingCidNorm: string
): "SET_LOCKED" | "SET_NOOP" | "SET_REJECTED_LOCKED" {
  if (!currentLockedCidNorm) return "SET_LOCKED";
  if (currentLockedCidNorm === incomingCidNorm) return "SET_NOOP";
  return "SET_REJECTED_LOCKED";
}

function compareEvmEventOrder(a: {
  blockNumber: bigint;
  transactionIndex: number;
  logIndex: number;
}, b: {
  blockNumber: bigint;
  transactionIndex: number;
  logIndex: number;
}): number {
  if (a.blockNumber !== b.blockNumber) return a.blockNumber < b.blockNumber ? -1 : 1;
  if (a.transactionIndex !== b.transactionIndex) return a.transactionIndex - b.transactionIndex;
  return a.logIndex - b.logIndex;
}

function compareSolanaEventOrder(a: {
  slot: bigint;
  txIndex: number;
  logIndex: number;
}, b: {
  slot: bigint;
  txIndex: number;
  logIndex: number;
}): number {
  if (a.slot !== b.slot) return a.slot < b.slot ? -1 : 1;
  if (a.txIndex !== b.txIndex) return a.txIndex - b.txIndex;
  return a.logIndex - b.logIndex;
}
```

---

## 10. Implementation Baseline

For best interoperability, implementations should align on the following practices:

1. Use `"col"` as the extension metadata key.
2. Encode pointer values in the `c1:<cid_norm>` format.
3. Normalize all CID input to CIDv1 base32 lowercase after parsing.
4. Derive `creator_snapshot_caip10` from the first registration owner event.
5. Derive `collection_key` as `creator_snapshot_caip10|cid_norm`.
6. Handle malformed collection pointers non-fatally.
7. Lock `col` after the first valid indexed value (first-write-wins).
8. Use canonical CAIP-19-style `asset` identifiers in indexer storage.
9. Resolve CID documents asynchronously with a bounded fetch policy.
10. Allow post-registration enrollment of agents that do not yet have a locked `col`, via the first valid `setMetadata("col", ...)` call.
