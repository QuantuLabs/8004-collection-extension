# 8004 Collection Extension Specification

**Version**: 1.0.0
**Status**: Draft
**Date**: 2026-02-16

## Guidance Scope

This document is a V1 reference profile intended to guide implementations toward interoperable behavior.
Implementations may diverge internally, provided they preserve deterministic collection identity and cross-indexer compatibility.

---

## 1. Overview

This specification defines a compact and indexer-friendly extension for ERC-8004 collections across Solana and EVM.

Design constraints:

1. No change to core 8004 registries.
2. Minimal on-chain footprint.
3. Deterministic indexing across indexers.
4. Compatibility with SDK patterns using metadata key/value writes.

V1 defines a single native flow: **collection pointer on agent metadata**.

Informational note (out of scope for V1): explorer/marketplace implementations may later support creator-signed display overrides for UI fields, without changing canonical identity fields (`collection_key`, `creator_snapshot_caip10`, `cid_norm`) or lock state.

---

## 2. Data Model

An agent participating in extension collections SHOULD store one collection pointer metadata entry:

- `key = "col"`
- `value = "c1:<cid_norm>"`

### 2.1 Pointer Value JSON Schema

Canonical schema URI:

- `https://raw.githubusercontent.com/QuantuLabs/8004-collection-extension/main/collection-pointer-value-v1.schema.json`

Repository reference:

- `./collection-pointer-value-v1.schema.json`

This schema validates the raw metadata value format for `col`.

Where:

- `cid_norm` is the canonical CID string in **CIDv1 base32 lowercase**.

Examples:

- valid: `c1:bafybeigdyrzt5...`
- invalid: `ipfs://bafy...` (must be normalized before storage)
- invalid: `Qm...` (CIDv0 must be converted to CIDv1 base32)

Input rule:

- SDK/indexer SHOULD accept CIDv0 or CIDv1 input.
- SDK/indexer SHOULD normalize to CIDv1 base32 lowercase before storage/comparison.
- Implementations MUST convert the value to lowercase before applying regex or prefix validation, to avoid rejecting valid uppercase Base32 CIDv1 inputs.
- The pointer value (including `c1:` prefix) SHOULD be between 50 and 256 characters after normalization.

### 2.2 CID Target Document JSON Schema

The CID pointed by `col` should resolve to a UTF-8 JSON document.

Canonical schema URI:

- `https://raw.githubusercontent.com/QuantuLabs/8004-collection-extension/main/collection-cid-document-v1.schema.json`

Repository reference:

- `./collection-cid-document-v1.schema.json`

Recommended minimum JSON shape:

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

The collection document is intentionally minimal to avoid duplicating agent-level information already present in `agentURI` data. `version` identifies the document schema profile, `symbol` is optional display metadata, and `socials` is the recommended container for social/account links.

The schema allows additional properties for forward compatibility. UIs MUST only render fields defined in the schema; additional properties MUST NOT be rendered without sanitization to prevent stored XSS via IPFS-hosted documents.

Schema URIs point to the canonical repository at `QuantuLabs/8004-collection-extension`.

### 2.3 Asset Identifier Format

To prevent cross-registry collisions, indexers SHOULD store `asset` in a canonical CAIP-19-style form that includes chain, registry identity, and agent local id.

Recommended shape:

`<chain_id_caip2>/erc8004:<registry_ref>/<agent_local_id>`

Per-chain `agent_local_id` format:

- **EVM**: `uint256` token ID as decimal string (e.g., `241`).
- **Solana**: Base58 string of the Agent Asset Account address (e.g., `7YQq9mVn7f8w4t6J8h2K3L5mN7P9rS2uV4xY6zA1bC3`).

Examples:

- `eip155:8453/erc8004:0x8004a6090Cd10A7288092483047B097295Fb8847/241`
- `solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp/erc8004:8rW9y8Xj3nL4QvGm2Qf3k8uYxKfJ2T9b6hVnP3Lm9Q4/7YQq9mVn7f8w4t6J8h2K3L5mN7P9rS2uV4xY6zA1bC3`

---

## 3. Chain and Creator Identity (CAIP-10)

### 3.1 Chain ID format

Recommended external chain identifier format is **CAIP-2**.
For Solana, the `solana` namespace reference is the first 32 chars of the Base58 string returned by `getGenesisHash` (truncated genesis hash string).
This rule is string-based and deterministic:

1. Read `getGenesisHash` as Base58 text.
2. Take the first 32 characters of that text.
3. Do not reinterpret as bytes and do not re-encode.

This follows the Chain Agnostic Solana namespace profile.

Examples:

- EVM Mainnet: `eip155:1`
- Base Mainnet: `eip155:8453`
- Solana Mainnet: `solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp`
- Solana Devnet: `solana:EtWTRABZaYq6iMfeYKouRu166VU2xqa1`
- Solana Testnet: `solana:4uhcVJyU9pJkvQyS88uRDiswHXSCkY3z`

### 3.2 Creator snapshot format

Recommended creator identifier format is **CAIP-10** account IDs.

Format:

`<caip2_chain_id>:<account_address>`

Examples:

- `eip155:8453:0x742d35Cc6634C0532925a3b844Bc9e7595f8fE43`
- `solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp:HvF3JqhahcX7JfhbDRYYCJ7S3f6nJdrqu5yi9shyTREp`

### 3.3 Creator snapshot source

`creator_snapshot_caip10` should be immutable and derived from the first registration event owner.
It is an indexer-derived field name, not an on-chain event field name.

- Solana: owner from first `AgentRegistered`
- EVM: owner from first `Registered`

Implementations should not use current owner after transfers for creator snapshot derivation.

Factory/minter semantics are explicit in this profile:

1. `creator_snapshot_caip10` reflects the first registration owner emitted by the registry flow.
2. If a factory/launchpad/relayer is the first registration owner, that factory/launchpad/relayer becomes the creator namespace.
3. Projects that need user-scoped creator namespaces SHOULD ensure the end user is the first registration owner (e.g., factory transfers ownership in the same transaction before the `Registered` event, or delegates registration to the end user).
4. Indexers SHOULD treat this field as provenance namespace, not as a verified brand identity signal.
5. UIs MUST display `creator_snapshot_caip10` prominently alongside collection display name. Two collections with the same CID but different creators are distinct collections and MUST be visually distinguishable.

Wallet rotation behavior is intentional in this profile:

1. New agents registered with a different initial owner will produce a different `creator_snapshot_caip10`.
2. Therefore, even with the same `cid_norm`, they will derive a different `collection_key`.
3. Projects that require a single continuous collection namespace SHOULD keep a stable registration wallet.

---

## 4. Collection Key Derivation

Deterministic key:

`collection_key = <creator_snapshot_caip10>|<cid_norm>`

`|` is used as separator because `:` is already used inside CAIP IDs.

Indexer implementations should persist both:

1. human-readable `collection_key`
2. indexed tuple fields (`creator_snapshot_caip10`, `cid_norm`)

Collection identity model: because `collection_key` includes `cid_norm`, **collections are immutable snapshots**. Updating the collection JSON document (name, image, description) produces a new CID and therefore a new `collection_key`. Existing agents remain locked to the original CID; only newly enrolled agents will point to the new CID. This is by design: it prevents silent metadata rug-pulls on existing collection members.

Projects that need to update display metadata without fragmenting their collection SHOULD use an explorer/marketplace display override layer (out of scope for V1) or plan collection versions as intentional "seasons" with documented CID lineage.

---

## 5. Indexer Specification

### 5.1 Event inputs

1. Solana registration event `AgentRegistered` (use field `owner`)
2. EVM registration event `Registered` (use field `owner`)
3. Metadata set events (`MetadataSet`)
4. Metadata delete events (`MetadataDeleted` where available). Not all 8004 registries emit this event. Indexers that do not have access to delete events SHOULD treat the absence as acceptable; lock state is unaffected since deletes are rejected for locked memberships.
5. Agent deactivation/burn signals where available:
   - EVM: NFT transfer to zero address.
   - Solana: chain-equivalent burn/close signal if exposed by the indexing source.

### 5.2 Canonical Ordering and Finality

V1 `col` immutability is an indexer policy. Core on-chain registries/programs remain unchanged and may still emit later metadata updates.

Indexers MUST use a canonical event ordering and MUST NOT rely on websocket/network arrival order.

Solana canonical source and order:

1. Canonical source SHOULD be `getBlock` with `commitment=finalized`.
2. Events from failed transactions (`meta.err != null`) MUST be ignored.
3. Canonical sort key MUST be `(slot, tx_index, log_index)` where:
   - `slot`: block slot.
   - `tx_index`: index in `getBlock().transactions`.
   - `log_index`: zero-based ordinal position of the event's log message within the `meta.logMessages` array, counting all log entries (including program invocation logs, return logs, and inner instruction logs), not only 8004-specific events. This ensures deterministic ordering even when a single transaction emits multiple `MetadataSet` events via CPI (cross-program invocation).

EVM canonical source and order:

1. Canonical source SHOULD be finalized head data.
2. Canonical sort key MUST be `(block_number, transaction_index, log_index)`.
3. `log_index` MUST be the block-scoped log index from RPC logs.

First-write-wins MUST be evaluated using the canonical sort key on canonical chain history.

### 5.3 Reorg and Rollback Policy

If an implementation ingests non-finalized data (for latency), it MUST support rollback.

1. Any event later marked non-canonical MUST be reverted.
2. Reversion MUST include previously locked `col` state derived from orphaned events.
3. After rollback, lock resolution MUST be recomputed from canonical history using canonical sort order.
4. Rollback handling SHOULD be driven by chain-native indicators:
   - EVM: canonical block hash tracking and `removed=true` logs when available.
   - Solana: finalized/root progression, replacing pre-finalized views.
5. Indexers SHOULD compare stored `lock_block_hash` against the canonical chain to detect if a lock event was orphaned. If a mismatch is found, the lock MUST be reverted and recomputed from canonical history.

### 5.4 Ingestion rules

V1 intentionally does not define any unlock, detach, or admin-override flow for `col`.

On metadata set where `key == "col"`:

1. Match key using exact, case-sensitive equality: `key == "col"`.
2. Decode value as UTF-8.
3. Trim leading/trailing ASCII whitespace (bytes `0x09`, `0x0A`, `0x0D`, `0x20`) from the decoded value. Non-ASCII whitespace (e.g., NBSP `0xC2A0`, zero-width space) is not trimmed and will cause CID parsing to fail.
4. Convert the trimmed value to lowercase.
5. Validate prefix `c1:`.
6. Parse and normalize CID -> `cid_norm` (if parsing fails, treat as malformed input).
7. Load `creator_snapshot_caip10` for asset.
8. Compute `collection_key`.
9. If no existing locked membership for `(chain_id_caip2, asset)`, create it and lock `col`.
10. If an existing locked membership has the same `cid_norm`, treat as idempotent (no membership change).
11. If an existing locked membership has a different `cid_norm`, reject mutation and keep the original locked membership.
12. Append history event.

Operator/delegate risk: if the underlying 8004 registry allows operators or delegates (e.g., ERC-721 `setApprovalForAll`, Metaplex delegates) to call `setMetadata`, a temporary operator (marketplace, staking contract) could set `col` on behalf of the owner. Because of first-write-wins, this permanently locks the agent's collection membership. Implementations aware of this risk SHOULD restrict `col` writes to owner-only at the registry level. If this is not possible, SDK/UI MUST warn owners that approving operators grants irrevocable `col` write authority.

Enrollment semantics for pre-existing agents:

1. An already registered agent with no locked `col` MAY join a collection later via its first valid `setMetadata("col", "c1:<cid_norm>")`.
2. Underlying registry ownership rules apply to `setMetadata`; this profile does not add extra write authority.
3. Once this first valid `col` is indexed, membership is immutable in v1.
4. If the agent is no longer controlled by the original creator, the current owner controls whether/when the agent joins a collection.

On metadata set where `key != "col"`:

1. Ignore for collection-extension state.

On metadata delete where `key == "col"`:

1. If no locked membership exists, no-op.
2. If a locked membership exists, reject delete and keep the original locked membership.
3. Append history event.

On malformed value:

1. Non-fatal ingestion.
2. Store invalid reason.
3. Exclude invalid entries from default collection queries.
4. Do not create or replace locked membership.

On burn/deactivation signal:

1. Mark membership `active=false` for `(chain_id_caip2, asset)`.
2. Keep locked identity fields (`creator_snapshot_caip10`, `cid_norm`, `collection_key`) unchanged.
3. Append history event.

### 5.5 Recommended tables

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

`block_timestamp` fields SHOULD be populated from chain block header timestamps, not ingestion wall-clock time.

### 5.6 CID Document Resolution (Non-Blocking)

Collection lock decisions MUST be based on pointer syntax/normalization only (`col` value).

1. Resolving the pointed CID document MUST be asynchronous and MUST NOT block ingestion workers.
2. Failed or slow document fetch MUST NOT prevent lock state updates.
3. Indexers SHOULD enforce bounded fetches (for example: timeout and max response size).
4. Resolution failures SHOULD be stored as diagnostic state for UI/ops, without mutating locked membership identity.
5. Indexers SHOULD cache successfully resolved CID documents locally and serve cached content as fallback when IPFS resolution fails or the document is unpinned from all nodes. The CID is content-addressed, so cached content is guaranteed to match the original if the CID matches.
6. Indexers SHOULD enforce a maximum document size (recommended: 64 KB) and a fetch timeout (recommended: 10 seconds).

---

## 6. Security and Trust Model

V1 uses native open tagging semantics.

Risks:

1. Sybil/spam collections.
2. Brand impersonation via same CID but different creator.
3. No on-chain collection governance.
4. Creator wallet rotation creates a new creator namespace by design (not a protocol flaw).
5. Human error on first valid `col` write is irreversible in v1.
6. Because `collection_key` includes `cid_norm`, changing collection JSON CID creates a new namespace for newly registered agents (collection identity fragmentation).
7. Operator/delegate griefing: a temporary operator can irrevocably lock agents into a garbage collection via first-write-wins (see section 5.4).
8. CID document availability: if unpinned from all IPFS nodes, display metadata is lost (collection identity is preserved but UI degrades).

Mitigations:

1. Display full `collection_key`, not only display name.
2. Apply indexer-level anti-spam controls.
3. Add reputation/trust scoring for creators at API/UI level.
4. Lock `col` on first valid set (first-write-wins) to prevent silent collection drift.
5. Keep immutable history for audit and dispute analysis.
6. SDK/UI SHOULD require explicit confirmation before first `col` write, since this action is irreversible in v1.
7. SDK/UI SHOULD warn owners that approving operators grants irrevocable `col` write authority.
8. Indexers SHOULD cache CID documents locally as availability fallback (see section 5.6).

### 6.1 V2 Migration Path (Informational)

V1 intentionally does not define unlock, detach, or admin-override flows. Future versions MAY address this via:

1. A new metadata key (e.g., `col_v2`) that supersedes `col` with updated semantics.
2. A burn-and-remint flow where the agent NFT is destroyed and re-created, resetting all extension state.
3. An indexer-level migration event where the creator signs an off-chain attestation linking old and new CIDs for the same logical collection.

These paths are sketched here to preserve design space; none are normative in V1.

---

## 7. Validation Guidelines (Baseline)

1. Extension key should be exact `col` (case-sensitive).
2. Value should be UTF-8 decoded and trimmed before validation.
3. Value format should be `c1:<cid_norm>`.
4. CID should decode successfully.
5. Canonical storage should use CIDv1 base32 lowercase.
6. Invalid entries should not stop ingestion.
7. `creator_snapshot_caip10` should be derived from first registration owner event.
8. `col` should be treated as immutable after first valid indexed value.
9. Solana events from failed transactions should be ignored.
10. Canonical sort order should follow section 5.2 before first-write-wins is applied.
11. If non-finalized ingestion is used, rollback handling should follow section 5.3.
12. `asset` should use canonical CAIP-19-style formatting.
13. `block_timestamp` fields should come from chain block headers.
14. CID document resolution should be non-blocking and bounded.
15. Burn/deactivation signals should set `active=false` without mutating locked identity fields.
16. Value should be converted to lowercase before prefix/regex validation (section 5.4 step 4).
17. ASCII whitespace trimming should cover bytes `0x09`, `0x0A`, `0x0D`, `0x20` only. Non-ASCII whitespace should not be trimmed.
18. Pointer value length after normalization should be between 50 and 256 characters.
19. UIs should only render CID document fields defined in the schema; additional properties should not be rendered without sanitization.
20. UIs should display `creator_snapshot_caip10` prominently to distinguish collections with the same CID but different creators.
21. EVM `agent_local_id` should be `uint256` decimal string; Solana `agent_local_id` should be Base58 asset account address.
22. Solana `log_index` should be the zero-based ordinal position in the full `meta.logMessages` array, counting all entries.

---

## 8. Conformance Test Vectors

1. Solana intra-block conflict:
   - Block/slot S has two valid `MetadataSet(col)` on same asset.
   - Event A at `(S, tx_index=0, log_index=10)` sets `col=a`.
   - Event B at `(S, tx_index=1, log_index=3)` sets `col=b`.
   - Expected locked value: `col=a`.

2. Solana failed transaction filter:
   - Event A appears in tx with `meta.err != null`.
   - Event B appears later in successful tx.
   - Expected locked value derived from Event B only.

3. EVM intra-block conflict:
   - Same block N has two `MetadataSet(col)` on same asset.
   - Event A `(N, tx_index=2, log_index=15)` sets `col=a`.
   - Event B `(N, tx_index=7, log_index=40)` sets `col=b`.
   - Expected locked value: `col=a`.

4. EVM reorg rollback:
   - Canonical history temporarily includes block H1 with first valid `col=a`.
   - H1 is replaced by canonical block H2 with first valid `col=b`.
   - Expected final locked value after rollback/replay: `col=b`.

5. Burn/deactivation handling:
   - Agent has active locked membership.
   - A burn/deactivation signal is ingested.
   - Expected state: `active=false`, locked `collection_key` history preserved.

6. CIDv0 normalization:
   - Agent sets `col=c1:QmYwAPJzv5CZsnA625s3Xf2nemtYgPpHdWEz79ojWnPbdG`.
   - SDK/indexer normalizes CIDv0 to CIDv1 base32 lowercase.
   - Expected `cid_norm`: CIDv1 base32 equivalent of the QmY... hash.
   - Expected locked value: `col` with the normalized CIDv1 base32 string.

7. Uppercase Base32 input:
   - Agent sets `col=c1:BAFYBEIGDYRZT5...` (uppercase).
   - Indexer converts to lowercase before validation.
   - Expected: accepted and normalized to `bafybeigdyrzt5...`.

8. Self-overwrite attempt:
   - Agent has locked `col=c1:bafyA...`.
   - Agent sets `col=c1:bafyB...` (different CID).
   - Expected: rejected. History event `SET_REJECTED_LOCKED`. Locked value unchanged.

9. Factory creator snapshot:
   - Factory contract registers an agent. `Registered` event has `owner=FactoryAddress`.
   - Agent is later transferred to `UserAddress`.
   - Expected: `creator_snapshot_caip10` remains `FactoryAddress`, not `UserAddress`.

10. Two agents same CID, same creator:
    - Agent A and Agent B both set `col=c1:bafySame...` and share the same `creator_snapshot_caip10`.
    - Expected: both agents have identical `collection_key` and are in the same collection.

11. Solana intra-transaction CPI conflict:
    - Single transaction calls `setMetadata(col)` twice via CPI on the same asset.
    - Event A at `(S, tx_index=5, log_index=12)` sets `col=a`.
    - Event B at `(S, tx_index=5, log_index=18)` sets `col=b`.
    - Expected locked value: `col=a` (lower `log_index` wins).

---

## 9. Reference Pseudocode

```ts
function normalizeCollectionCid(input: string): string | null {
  try {
    const raw = input.trim();
    const noScheme = raw
      .replace(/^ipfs:\/\//i, "")
      .replace(/^\/ipfs\//i, "");

    const cid = CID.parse(noScheme); // accepts v0 or v1
    return cid.toV1().toString().toLowerCase();
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

For best interoperability, implementations should align on the following:

1. Use `col` as the extension metadata key.
2. Encode pointer values as `c1:<cid_norm>`.
3. Normalize CID input to CIDv1 base32 lowercase.
4. Derive `creator_snapshot_caip10` from first registration owner event.
5. Derive `collection_key` as `creator_snapshot_caip10|cid_norm`.
6. Apply non-fatal handling for malformed collection pointers.
7. Lock `col` after first valid indexed value (`first-write-wins`).
8. Use canonical CAIP-19-style `asset` identifiers in indexer storage.
9. Resolve CID documents asynchronously with bounded fetch policy.
10. Allow post-registration enrollment of agents that do not yet have a locked `col` via first valid `setMetadata("col", ...)`.
