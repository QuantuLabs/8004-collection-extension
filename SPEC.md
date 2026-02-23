# 8004 Collection Extension Specification

**Version**: 1.0.0
**Status**: Draft
**Date**: 2026-02-16

---

## Guidance Scope

This document is a reference profile intended to guide implementations toward interoperable behavior.
Implementations may diverge internally, provided they preserve deterministic collection identity and cross-indexer compatibility.

---

## 1. Overview

This specification defines a compact and indexer-friendly extension for ERC-8004 collections across Solana and EVM chains.

The design aims to satisfy four constraints:

- No change to core 8004 registries.
- Minimal on-chain footprint.
- Deterministic indexing across independent indexers.
- Compatibility with existing SDK patterns that use metadata key/value writes.

This specification defines a single native flow: **collection pointer on agent metadata**. An agent joins a collection by writing a single metadata entry that references a content-addressed document on IPFS. The indexer enforces immutability on this pointer after the first valid write.

> **Note (out of scope):** explorer and marketplace implementations may later support creator-signed display overrides for UI fields, without changing canonical identity fields (`collection_key`, `creator_snapshot_caip10`, `cid_norm`) or lock state.

---

## 2. Data Model

An agent participating in extension collections SHOULD store one collection pointer metadata entry:

- **Key**: `"col"`
- **Value**: `"c1:<cid_norm>"`

The `cid_norm` portion is the canonical CID string in CIDv1 base32 lowercase.

<br>

### 2.1 Pointer Value JSON Schema

The canonical schema that validates the **normalized** (post-ingestion) metadata value format for `col` is published at:

```
https://raw.githubusercontent.com/QuantuLabs/8004-collection-extension/main/collection-pointer-value-v1.schema.json
```

Repository path: `./collection-pointer-value-v1.schema.json`

<br>

A valid pointer value looks like `c1:bafybeigdyrzt5...`.

Values such as `ipfs://bafy...` or raw CIDv0 strings like `Qm...` are not valid storage formats and must be normalized before being written on-chain.

<br>

**CID input normalization:**

SDKs and indexers SHOULD accept both CIDv0 (base58) and CIDv1 input and normalize to CIDv1 base32 lowercase before storage or comparison.

Implementations MUST perform this normalization after CID parsing, not by lowercasing the raw input string. CIDv0 uses base58 encoding, which is case-sensitive; lowercasing before parsing would destroy CIDv0 input and produce an incorrect result.

<br>

**Length constraints:**

The pointer value (including the `c1:` prefix) MUST be between 62 and 256 characters after normalization. A valid CIDv1 base32 string with sha2-256 is at least 59 characters long; with the `c1:` prefix the minimum total length is 62.

<br>

### 2.2 CID Target Document JSON Schema

The CID referenced by the `col` pointer should resolve to a UTF-8 JSON document that describes the collection. The canonical schema for this document is published at:

```
https://raw.githubusercontent.com/QuantuLabs/8004-collection-extension/main/collection-cid-document-v1.schema.json
```

Repository path: `./collection-cid-document-v1.schema.json`

<br>

**Recommended minimum document shape:**

```json
{
  "version": "1.0.0",
  "name": "My Collection",
  "symbol": "COLL",
  "description": "Optional description",
  "image": "ipfs://bafy...",
  "banner_image": "ipfs://bafy...",
  "parent": "bafybeigparentcid...",
  "socials": {
    "website": "https://example.com",
    "x": "@mycollection",
    "discord": "https://discord.gg/mycollection"
  }
}
```

The collection document is intentionally minimal to avoid duplicating agent-level information already present in `agentURI` data. The `version` field identifies the document schema profile, `symbol` is optional display metadata, `parent` is an optional CID reference to a parent collection (see section 4.1), and `socials` is the recommended container for social and account links.

<br>

**Sanitization requirements:**

The schema allows additional properties for forward compatibility. However, UIs MUST sanitize ALL fields from CID documents before rendering, including the defined schema fields (`name`, `symbol`, `description`, and `socials` values).

Raw HTML rendering of any field is prohibited.

URI-like fields (`image`, `banner_image`, `socials.website`) MUST be validated against a safe scheme allowlist: only `ipfs://`, `https://`, and `ar://` are permitted. Schemes such as `javascript:`, `data:`, `vbscript:`, and `blob:` MUST be rejected.

Any additional properties beyond the defined schema MUST NOT be rendered without sanitization, to prevent stored XSS via IPFS-hosted documents.

<br>

### 2.3 Asset Identifier Format

To prevent cross-registry collisions, indexers SHOULD store the `asset` field in a canonical format that includes the chain identifier, registry address, and agent local ID:

```
<chain_id_caip2>/<registry_ref>/<agent_local_id>
```

On EVM chains, the `agent_local_id` is the `uint256` token ID expressed as a decimal string (e.g., `241`).

On Solana, it is the Base58 string of the Agent Asset Account address (e.g., `7YQq9mVn7f8w4t6J8h2K3L5mN7P9rS2uV4xY6zA1bC3`).

<br>

**Full examples:**

- `eip155:8453/0x8004a6090cd10a7288092483047b097295fb8847/241`
- `solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp/8rW9y8Xj3nL4QvGm2Qf3k8uYxKfJ2T9b6hVnP3Lm9Q4/7YQq9mVn7f8w4t6J8h2K3L5mN7P9rS2uV4xY6zA1bC3`

---

## 3. Chain and Creator Identity (CAIP-10)

### 3.1 Chain ID Format

The recommended external chain identifier format is **CAIP-2**.

For EVM chains, this takes the familiar `eip155:<chain_id>` form (e.g., `eip155:1` for Ethereum mainnet, `eip155:8453` for Base mainnet).

For Solana, the CAIP-2 namespace reference is derived from the genesis hash. Specifically, it is the first 32 characters of the Base58 string returned by `getGenesisHash`. This rule is string-based and deterministic: read the genesis hash as Base58 text, take the first 32 characters, and do not reinterpret the bytes or re-encode them. This follows the Chain Agnostic Solana namespace profile.

<br>

**Reference values:**

| Chain            | CAIP-2 Identifier                              |
|------------------|-------------------------------------------------|
| EVM Mainnet      | `eip155:1`                                      |
| Base Mainnet     | `eip155:8453`                                   |
| Solana Mainnet   | `solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp`       |
| Solana Devnet    | `solana:EtWTRABZaYq6iMfeYKouRu166VU2xqa1`       |
| Solana Testnet   | `solana:4uhcVJyU9pJkvQyS88uRDiswHXSCkY3z`       |

<br>

### 3.2 Creator Snapshot Format

The recommended creator identifier format is **CAIP-10** account IDs, which combine the CAIP-2 chain identifier with the account address:

```
<caip2_chain_id>:<account_address>
```

To ensure deterministic `collection_key` derivation across indexers, EVM account addresses in `creator_snapshot_caip10` and `asset` fields MUST be stored in lowercase, per the CAIP-10 specification for `eip155` namespaces. EIP-55 mixed-case encoding is reserved for display only.

<br>

**Examples:**

- `eip155:8453:0x742d35cc6634c0532925a3b844bc9e7595f8fe43`
- `solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp:HvF3JqhahcX7JfhbDRYYCJ7S3f6nJdrqu5yi9shyTREp`

<br>

### 3.3 Creator Snapshot Source

The `creator_snapshot_caip10` is an indexer-derived field (not an on-chain event field) that captures the identity of the address that first registered the agent. It should be treated as immutable once set.

On Solana, it is derived from the `owner` field of the first `AgentRegistered` event.
On EVM, it comes from the `owner` field of the first `Registered` event.

Implementations should never use the current owner after transfers for creator snapshot derivation.

<br>

**Factory and minter semantics:**

1. The `creator_snapshot_caip10` always reflects the first registration owner emitted by the registry flow. If a factory, launchpad, or relayer is the first registration owner, that address becomes the creator namespace.

2. Projects that need user-scoped creator namespaces SHOULD ensure the end user is the first registration owner. This can be achieved by having the factory transfer ownership in the same transaction before the `Registered` event, or by delegating registration to the end user directly.

3. Indexers SHOULD treat this field as a provenance namespace, not as a verified brand identity signal.

4. UIs MUST display `creator_snapshot_caip10` prominently alongside the collection display name. Two collections with the same CID but different creators are distinct collections and MUST be visually distinguishable.

<br>

**Wallet rotation:**

If a project registers new agents from a different wallet, those agents will have a different `creator_snapshot_caip10`, and therefore a different `collection_key` even when pointing to the same CID. Projects that require a single continuous collection namespace SHOULD keep a stable registration wallet.

---

## 4. Collection Key Derivation

The collection key is derived deterministically from the creator's identity and the normalized CID:

```
collection_key = <creator_snapshot_caip10>|<cid_norm>
```

The pipe character `|` is used as separator because the colon `:` is already used inside CAIP identifiers.

Indexer implementations should persist both the human-readable `collection_key` and the indexed tuple fields (`creator_snapshot_caip10`, `cid_norm`) for efficient querying.

<br>

**Immutable snapshot model:**

Because the `collection_key` includes `cid_norm`, **collections are immutable snapshots**. Updating the collection JSON document (name, image, description) produces a new CID and therefore a new `collection_key`.

Existing agents remain locked to the original CID; only newly enrolled agents will point to the updated document. This is by design: it prevents silent metadata rug-pulls on existing collection members.

Projects that need to update display metadata without fragmenting their collection SHOULD use an explorer or marketplace display override layer (out of scope) or plan collection versions as intentional "seasons" with documented CID lineage.

---

## 4.1 Sub-Collections

A collection can declare a parent by including a `parent` field in its CID document (section 2.2). The value is the CIDv1 base32 lowercase string of the parent collection's CID document.

<br>

**Authorization model:**

Sub-collection authorization is implicit via the `collection_key` namespace. Because `collection_key = creator_snapshot_caip10|cid_norm`, a sub-collection relationship is only valid when the parent collection shares the same `creator_snapshot_caip10`. The indexer computes `parent_collection_key = creator_snapshot_caip10|parent_cid_norm` and verifies that at least one locked membership with this `collection_key` exists (regardless of `active` status). Cross-creator sub-collections are not possible: if Alice creates a parent and Bob creates a child referencing Alice's CID, the lookup `bob|parent_cid` will not match Alice's collection `alice|parent_cid`.

<br>

**Resolution:**

The parent relationship is resolved during asynchronous CID document resolution (section 5.6), not at ingestion time. Lock state and membership identity are unaffected by the parent field. This means:

1. The `col` pointer is locked immediately based on pointer syntax (first-write-wins).
2. When the CID document is resolved, the indexer extracts the `parent` field, normalizes the parent CID, and validates the parent relationship.
3. If the parent collection exists in the index, the indexer stores `parent_cid_norm`, `parent_collection_key`, and `depth` on the membership row.
4. If the parent collection does not exist, the parent relationship is stored as unresolved. The membership remains valid; only the hierarchy metadata is incomplete.

<br>

**Depth limit:**

The maximum sub-collection nesting depth is **8 levels**. A root collection (no `parent` field) has `depth = 0`. Depth MUST only be finalized when the full ancestry chain to a root is resolved; partial chains store `depth = NULL`. Once resolved, `depth = parent_depth + 1`. If the computed depth exceeds 8, the parent relationship MUST be rejected and logged as a diagnostic event, but the membership itself remains valid.

<br>

**Immutability:**

Because the `parent` field is inside the CID document and the CID is content-addressed, the parent relationship is immutable. Changing the parent requires publishing a new CID document, which produces a new `cid_norm` and therefore a new `collection_key`. This is consistent with the immutable snapshot model (section 4).

---

## 5. Indexer Specification

### 5.1 Event Inputs

Indexers should consume the following event types:

1. **Registration events** — `AgentRegistered` on Solana and `Registered` on EVM, using the `owner` field to derive the creator snapshot.

   Registration events MUST also initialize the ownership table entry for the agent, setting `current_owner` to the registration `owner`. This ensures ownership is known even when `MetadataSet(col)` occurs before the ERC-721 `Transfer` event within the same transaction (e.g., during mint flows).

   **Re-mint handling:** If a Registration event is processed for an asset that already has a row in the membership table (e.g., a re-minted EVM token ID), the indexer MUST delete the existing membership row, then delete and re-initialize the ownership entry from the new Registration event (new `creator_snapshot_caip10`, new `current_owner`). The membership row remains absent until a new valid `MetadataSet(col)` creates it via step 13. A `RE_REGISTERED` history event SHOULD be appended for audit.

2. **Metadata set events** — `MetadataSet`, which carry the key/value pair written by the agent owner.

3. **Metadata delete events** — `MetadataDeleted`, where available. Not all 8004 registries emit this event. Indexers that do not have access to delete events SHOULD treat their absence as acceptable, since lock state is unaffected (deletes are rejected for locked memberships).

4. **Deactivation and burn signals** — On EVM, an NFT transfer to the zero address. On Solana, the chain-equivalent burn or close signal if exposed by the indexing source.

5. **Transfer events** — On EVM, the ERC-721 `Transfer(address indexed from, address indexed to, uint256 indexed tokenId)` event from the registry contract. On Solana, Metaplex Core transfer instruction logs or owner field changes detected from finalized block data.

   Implementations SHOULD prefer log-based transfer detection (which provides a `log_index` for deterministic interleaving) over account state diffs (which lack a `log_index`). When ownership is derived from account state diffs, the ownership update MUST be applied at the transaction boundary after all log-based events in that transaction have been processed.

   These events maintain the local ownership state required for creator verification (see section 5.7).

<br>

### 5.2 Canonical Ordering and Finality

Collection immutability is an indexer policy. The core on-chain registries and programs remain unchanged and may still emit later metadata updates.

Indexers MUST use a canonical event ordering and MUST NOT rely on websocket or network arrival order.

<br>

#### Solana

The canonical source SHOULD be `getBlock` with `commitment=finalized`. Events from failed transactions (`meta.err != null`) MUST be ignored.

The canonical sort key MUST be `(slot, tx_index, log_index)` where:

- **slot**: the block slot.
- **tx_index**: the transaction's position within the block's `transactions` array.
- **log_index**: the zero-based ordinal position of the event's log message within the full `meta.logMessages` array, counting all log entries (program invocation logs, return logs, and inner instruction logs), not only 8004-specific events.

Using the full array ordinal ensures deterministic ordering even when a single transaction emits multiple `MetadataSet` events via cross-program invocation (CPI).

The flat `log_index` approach is preferred over instruction-based tracking (`instruction_index`, `inner_instruction_index`) because a single CPI instruction can emit multiple events, and reconstructing the CPI call stack from log strings is fragile. The flat array index from finalized `getBlock` data is immutable and deterministic.

<br>

**Runtime log truncation:**

The Solana runtime enforces a per-transaction log byte limit. If a transaction exceeds this limit, the runtime silently truncates `meta.logMessages` and appends a `"Log truncated"` entry. Events emitted after the truncation point will not appear in the log array.

Indexers relying on log-based event parsing MUST scan for the `"Log truncated"` sentinel before processing any events from a transaction.

All events from that transaction (Registration, Transfer, MetadataSet, Burn) MUST be deferred and queued for recovery via archive replay or state diff analysis, since truncated event payloads (including asset identifiers) may be lost. Events visible before the truncation point MUST also be deferred, because later events in the same transaction may have been truncated.

On recovery, indexers MUST replay all events for affected assets from the recovered transaction's canonical position forward (including events from subsequent transactions that were already processed), to ensure ownership and lock state are consistent.

If recovery is impossible (no archive source available), the transaction MUST remain permanently quarantined and its events never processed. For assets known to be affected by the quarantined transaction, the `SET_UNVERIFIABLE` blocking rule (section 5.7) applies: their `col` processing remains blocked until archive recovery is eventually performed. For assets whose involvement cannot be determined (truncated payloads hide the asset identifier), subsequent transactions proceed normally for first-write-wins evaluation.

Note: when truncated payloads prevent identification of affected assets, temporary cross-indexer divergence is accepted until archive recovery resolves the quarantined transaction.

<br>

#### EVM

The canonical source SHOULD be finalized head data.

The canonical sort key MUST be `(block_number, transaction_index, log_index)`, where `log_index` is the block-scoped log index from RPC logs.

<br>

First-write-wins MUST be evaluated using the canonical sort key on canonical chain history.

All event types (Registration, Transfer, MetadataSet, MetadataDeleted, Burn) MUST be interleaved into a single processing queue ordered by the canonical sort key. Implementations MUST NOT batch events by type (e.g., processing all Transfers before all MetadataSet events within a block).

Implementations sourcing events from separate streams MUST buffer `MetadataSet(col)` events until all prior events (up to the same canonical sort key) are ingested.

<br>

### 5.3 Reorg and Rollback Policy

If an implementation ingests non-finalized data for lower latency, it MUST support rollback.

Any event later marked non-canonical MUST be reverted, including any previously locked `col` state and any ownership state updates derived from orphaned events. History entries derived from orphaned events MUST be marked with `removed = TRUE`. After rollback, lock resolution, ownership state, and parent/depth relationships MUST be recomputed from canonical history using canonical sort order.

Rollback handling SHOULD be driven by chain-native indicators:

- **EVM**: canonical block hash tracking and `removed=true` logs when available.
- **Solana**: finalized/root progression replacing pre-finalized views.

Indexers SHOULD compare the stored `lock_block_hash` against the canonical chain to detect if a lock event was orphaned. If a mismatch is found, the lock MUST be reverted and recomputed from canonical history.

<br>

### 5.4 Ingestion Rules

This specification intentionally does not define any unlock, detach, or admin-override flow for `col`.

<br>

#### On metadata set where `key == "col"`

> **Control flow**: Any step that produces a malformed/`INVALID` outcome (steps 2, 4, 6, 8) immediately exits this sequence and hands off to the "On malformed value" handler below. Steps that produce a terminal event (`SET_UNVERIFIABLE`, `SET_REJECTED_NOT_CREATOR`, `SET_NOOP`, `SET_REJECTED_LOCKED`) skip to step 14 as noted inline.

1. Match the key using exact, case-sensitive equality: `key == "col"`.

2. Decode the value as strict UTF-8 (invalid byte sequences MUST produce an `INVALID` event, not replacement characters).

3. Trim leading and trailing ASCII whitespace (bytes `0x09`, `0x0A`, `0x0D`, `0x20`) from the decoded value. Non-ASCII whitespace (e.g., NBSP `0xC2A0`, zero-width space) is not trimmed and will cause CID parsing to fail, which is the intended behavior.

4. Validate that the value starts with the prefix `c1:` (case-insensitive: accept `c1:`, `C1:`, etc.). If it does not, treat as malformed input. The canonical stored prefix is always lowercase `c1:` regardless of input case.

5. Extract the CID portion (everything after `c1:`).

6. Parse the CID string. The parser should accept both CIDv0 (base58) and CIDv1 in any base or case. If parsing fails, treat the value as malformed input.

7. Normalize the parsed CID to CIDv1 base32 lowercase to produce `cid_norm`. This step handles CIDv0-to-v1 conversion and uppercase-to-lowercase normalization without destroying case-sensitive encodings like base58.

8. Validate that the full pointer value (`c1:` + `cid_norm`) is between 62 and 256 characters. If out of bounds, treat as malformed.

9. If an existing locked membership exists for `(chain_id_caip2, asset)`: if it has the same `cid_norm`, treat as idempotent (`SET_NOOP`); if different, reject (`SET_REJECTED_LOCKED`). In either case, skip to step 14. This short-circuit avoids unnecessary ownership checks and false deferrals for already-locked assets.

10. Load the `creator_snapshot_caip10` for this asset. If unavailable (e.g., Registration event not yet indexed), treat as `SET_UNVERIFIABLE` and skip to step 14.

11. Verify that the current tracked owner of this asset matches `creator_snapshot_caip10` (see section 5.7). If the agent has been transferred and the current owner differs from the original creator, reject the write (event type `SET_REJECTED_NOT_CREATOR`) and skip to step 14. The agent's `col` slot remains open for a future valid write if ownership returns to the original creator.

12. Compute the `collection_key` from `creator_snapshot_caip10` and `cid_norm`.

13. Create the membership and lock it (event type `SET_LOCKED`).

14. Append a history event.

<br>

#### Operator and delegate risk

If the underlying 8004 registry allows operators or delegates (e.g., ERC-721 `setApprovalForAll`, Metaplex delegates) to call `setMetadata`, a temporary operator such as a marketplace or staking contract could set `col` on behalf of the owner.

Because of first-write-wins, this permanently locks the agent's collection membership. Implementations aware of this risk SHOULD restrict `col` writes to owner-only at the registry level. If this is not possible, SDKs and UIs MUST warn owners that approving operators grants irrevocable `col` write authority.

<br>

#### Enrollment of pre-existing agents

An already-registered agent with no locked `col` may join a collection at any time via its first valid `setMetadata("col", "c1:<cid_norm>")`.

The underlying registry's ownership rules apply to `setMetadata`; this profile does not add extra write authority. Once the first valid `col` is indexed, membership becomes immutable.

If the agent is no longer controlled by the original creator, the `col` write will be rejected by the ownership check until ownership returns to the original creator (section 5.7).

<br>

#### On metadata set where `key != "col"`

These events are ignored for collection-extension state.

<br>

#### On metadata delete where `key == "col"`

If no locked membership exists, this is a no-op. If a locked membership exists, the delete is rejected and the original locked membership is preserved. A history event with type `DELETE_REJECTED_LOCKED` is appended.

<br>

#### On malformed value

Malformed values are handled non-fatally. The indexer records the malformed event in the history table with event type `INVALID` and an `invalid_reason` field. No row is inserted or updated in the membership table. This avoids conflicts with NOT NULL constraints on `cid_norm` and `collection_key` in the membership table.

The agent remains eligible to set a valid `col` in a future transaction.

<br>

#### On burn or deactivation signal

The membership is marked `active=false` for the given `(chain_id_caip2, asset)`. The locked identity fields (`creator_snapshot_caip10`, `cid_norm`, `collection_key`) remain unchanged. A history event is appended. If a burn and `MetadataSet(col)` occur in the same transaction, they are processed in `log_index` order per the unified queue; the final `active` state reflects whichever event was last.

<br>

### 5.5 Recommended Tables

```sql
CREATE TABLE extension_collection_memberships (
  chain_id_caip2          TEXT        NOT NULL,
  asset                   TEXT        NOT NULL,
  creator_snapshot_caip10 TEXT        NOT NULL,
  cid_norm                TEXT        NOT NULL,
  collection_key          TEXT        NOT NULL,
  col_locked              BOOLEAN     NOT NULL DEFAULT TRUE,
  lock_tx_hash            TEXT,
  lock_block_number       BIGINT,
  lock_block_hash         TEXT,
  lock_block_timestamp    TIMESTAMPTZ,
  lock_tx_index           INTEGER,
  lock_log_index          INTEGER,
  lock_slot               BIGINT,
  parent_cid_norm         TEXT,
  parent_collection_key   TEXT,
  depth                   INTEGER,
  lock_created_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  active                  BOOLEAN     NOT NULL DEFAULT TRUE,
  first_seen_at           TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at              TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  PRIMARY KEY (chain_id_caip2, asset)
);
```

```sql
CREATE TABLE extension_agent_ownership (
  chain_id_caip2          TEXT        NOT NULL,
  asset                   TEXT        NOT NULL,
  creator_snapshot_caip10 TEXT        NOT NULL,
  current_owner           TEXT        NOT NULL,
  previous_owner          TEXT,
  block_number     BIGINT,
  block_hash       TEXT,
  slot             BIGINT,
  tx_hash          TEXT,
  tx_index         INTEGER,
  log_index        INTEGER,
  updated_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  PRIMARY KEY (chain_id_caip2, asset)
);
```

```sql
CREATE TABLE extension_collection_membership_history (
  id                      BIGSERIAL   PRIMARY KEY,
  chain_id_caip2          TEXT        NOT NULL,
  asset                   TEXT        NOT NULL,
  creator_snapshot_caip10 TEXT,
  cid_norm                TEXT,
  collection_key          TEXT,
  event_type              TEXT        NOT NULL,
    -- SET_LOCKED | SET_NOOP | SET_REJECTED_LOCKED
    -- SET_REJECTED_NOT_CREATOR | SET_UNVERIFIABLE
    -- DELETE_REJECTED_LOCKED | INVALID | DEACTIVATE
    -- RE_REGISTERED
  tx_hash                 TEXT,
  block_number            BIGINT,
  block_hash              TEXT,
  block_timestamp         TIMESTAMPTZ,
  tx_index                INTEGER,
  log_index               INTEGER,
  slot                    BIGINT,
  invalid_reason          TEXT,
  removed                 BOOLEAN     NOT NULL DEFAULT FALSE,
  created_at              TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

```sql
CREATE INDEX idx_ext_collection_key
  ON extension_collection_memberships(collection_key)
  WHERE active = TRUE;

CREATE INDEX idx_ext_collection_creator_cid
  ON extension_collection_memberships(creator_snapshot_caip10, cid_norm)
  WHERE active = TRUE;

CREATE INDEX idx_ext_collection_cid
  ON extension_collection_memberships(cid_norm)
  WHERE active = TRUE;

CREATE INDEX idx_ext_collection_parent
  ON extension_collection_memberships(parent_collection_key)
  WHERE parent_collection_key IS NOT NULL;

CREATE INDEX idx_ext_collection_key_all
  ON extension_collection_memberships(collection_key);
  -- Unfiltered: used for parent existence checks (regardless of active status)
```

The `block_timestamp` fields SHOULD be populated from chain block header timestamps, not from ingestion wall-clock time.

<br>

### 5.6 CID Document Resolution (Non-Blocking)

Collection lock decisions MUST be based solely on pointer syntax and normalization (the `col` value). Resolving the pointed CID document to retrieve collection metadata MUST be performed asynchronously and MUST NOT block ingestion workers.

A failed or slow document fetch MUST NOT prevent lock state updates.

Indexers SHOULD enforce bounded fetches with a recommended maximum document size of 64 KB and a fetch timeout of 10 seconds. Resolution failures SHOULD be stored as diagnostic state for UI and operational tooling, without mutating any locked membership identity.

Before applying resolution results (parent, depth), indexers MUST verify that the membership's `cid_norm` and `creator_snapshot_caip10` have not changed since the resolution was initiated (e.g., due to reorg or re-mint). Stale results MUST be discarded.

Indexers SHOULD cache successfully resolved CID documents locally and serve cached content as a fallback when IPFS resolution fails or when the document is no longer pinned on any node. Because the CID is content-addressed, cached content is guaranteed to match the original as long as the CID matches.

When the resolved document does not contain a `parent` field, the indexer MUST set `depth = 0` (root collection).

When the resolved document contains a `parent` field, the indexer MUST attempt to parse and normalize the parent CID to CIDv1 base32 lowercase.

- **Parse failure:** If the value matches the schema regex but is not a valid CID, the indexer MUST ignore the parent field: set `parent_cid_norm = NULL`, `parent_collection_key = NULL`, `depth = 0` (treat as root), and log a diagnostic event.
- **Parse success:** The indexer MUST compute `parent_collection_key = creator_snapshot_caip10|parent_cid_norm` and verify that at least one locked membership with this `collection_key` exists (regardless of `active` status, since collection identity is immutable) (see section 4.1). The indexer MUST always store `parent_cid_norm` and `parent_collection_key`, regardless of whether the parent collection exists yet.

**Depth finalization:**

- If the parent's depth is finalized and `parent_depth < 8`, set `depth = parent_depth + 1`.
- If the parent's depth is finalized and `parent_depth >= 8`, reject the parent relationship and log a diagnostic event; keep `depth = NULL`.
- If the parent's depth is not yet finalized, set `depth = NULL` (unresolved).

**Depth immutability:** Once depth transitions from NULL to a finalized integer, it MUST NOT revert to NULL during normal operation, with two exceptions:

1. During reorg replay (section 5.3), depth is recomputed from canonical history and may change, including reverting to NULL if ancestry cannot be resolved.
2. When a re-mint (section 5.1 item 1) deletes the last membership row for a parent `collection_key`, child collections referencing that parent MUST have their depth reverted to NULL (parent no longer exists).

**Depth cascade:** Indexers MUST cascade depth updates: when a collection's depth changes (NULL to finalized, finalized to NULL due to re-mint or reorg, or between finalized values after reorg replay), all children sharing its `parent_collection_key` MUST be re-evaluated.

<br>

### 5.7 Ownership Tracking

Indexers MUST maintain a local ownership state for each agent, updated from Transfer events (section 5.1, item 5). This ownership table maps each `(chain_id_caip2, asset)` to its `current_owner` in CAIP-10 format.

<br>

#### Purpose

The `col` metadata key binds agents to a creator's collection namespace. Because `creator_snapshot_caip10` is derived from the first registration owner and never changes, an agent that has been transferred to a different owner still carries the original creator's namespace. Without ownership verification, the new owner could write `col` and place the agent into the original creator's collection without authorization.

Ownership tracking prevents this: on every `MetadataSet(col)` event, the indexer compares the current tracked owner to `creator_snapshot_caip10`. If they differ, the write is rejected.

<br>

#### Implementation

On EVM, indexers ingest `Transfer(from, to, tokenId)` events from the registry contract and update the local ownership table. On Solana, indexers track Metaplex Core transfer instructions or detect owner field changes from finalized block data.

The ownership table is a materialized cache: `(chain_id_caip2, asset) → current_owner`. It is not the source of truth; the canonical event stream is. On chain reorganizations, the ownership table MUST be rebuilt by replaying canonical events from the last finalized checkpoint (see section 5.3).

The `previous_owner` column is an optimization for single-step rollbacks only; if multiple Transfer events for the same asset occur within a single block, or for multi-block reorgs, full replay from the last finalized checkpoint is required.

Storage overhead is approximately 80–150 bytes per agent. For a registry of 10 million agents, total storage is under 1 GB.

<br>

#### Fallback for cold start

When an indexer starts fresh or encounters an agent whose ownership is not yet tracked, it MAY bootstrap ownership state using the following chain-specific methods:

- **EVM**: replay Transfer events from archive logs/traces to reconstruct ownership at the exact event position. A single `ownerOf` RPC call returns end-of-block state and MUST NOT be used when additional Transfer events for the same asset exist later in the same block, as those would change the result. `ownerOf(blockNumber)` is acceptable only when the indexer has confirmed no later same-block transfers exist for that asset. Querying `latest` state is unsafe because the current owner may differ from the owner at the historical event's block.
- **Solana**: replay `getBlock` data from the agent's registration slot forward, since Solana's `getAccountInfo` RPC does not support historical slot queries. Alternatively, use a specialized indexing service that provides historical account state.

If historical ownership cannot be determined, the `col` write MUST be deferred with event type `SET_UNVERIFIABLE` rather than evaluated against present-day state.

Deferred events MUST block further `col` processing for that specific asset until ownership is resolved, to preserve first-write-wins ordering. Regardless of the deferral reason (unknown ownership or log truncation), all `SET_UNVERIFIABLE` events MUST block asset-scoped `col` processing and MUST be re-evaluated in canonical order when the deferral condition is resolved.

When replaying deferred `col` events, the ownership check MUST use the historical owner at each event's original canonical sort key, not the current owner. Implementations SHOULD reconstruct the asset's full event sequence (including Transfers) from the earliest deferred point to ensure correct ownership state at each evaluation.

This fallback SHOULD only be used during initial sync or gap repair, not during steady-state ingestion.

<br>

#### Same-transaction ordering

When a Transfer event and a MetadataSet(col) event occur within the same transaction, the unified processing queue (section 5.2) ensures they are evaluated in `log_index` order. If the Transfer is emitted before the MetadataSet, the ownership table is updated before the `col` write is evaluated. If the MetadataSet is emitted before the Transfer, the `col` write is evaluated against the pre-transfer owner. Implementations MUST NOT reorder events within a transaction.

For log-based indexers with `log_index` ordering, the standard launchpad/factory flow (Register → SetCol → Transfer in one transaction) is handled correctly by normal `log_index` order: the `MetadataSet(col)` is processed while the creator is still the owner.

On Solana, when ownership is derived from account state diffs (which lack a `log_index`), the ownership update is applied at the transaction boundary after all log-based events in that transaction.

When a state-diff ownership change and a `MetadataSet(col)` co-occur in the same transaction and intra-transaction log ordering is unavailable, the `MetadataSet(col)` MUST be deferred as `SET_UNVERIFIABLE` and resolved via archive log replay to reconstruct the actual `log_index` ordering. This applies to all same-transaction cases, including those co-occurring with Registration events.

Implementations SHOULD prefer log-based transfer detection to avoid deferrals entirely.

<br>

#### Meta-transactions and account abstraction

Ownership tracking is robust against meta-transaction relayers (ERC-2771), account abstraction bundlers (EIP-4337), and contract wallets (Safe, Squads). Because the check compares the tracked NFT owner (from Transfer events) to the creator snapshot, the identity of the transaction sender is irrelevant. This is the primary reason ownership tracking is preferred over transaction-sender-based verification.

---

## 6. Security and Trust Model

This specification uses native open tagging semantics, meaning any agent owner can set any CID as their collection pointer. This openness introduces several risks, each with corresponding mitigations.

<br>

| # | Risk | Mitigation |
|---|------|------------|
| 1 | **Sybil and spam collections**: anyone can create collections by publishing a CID. | Display the full `collection_key` (not only the display name) and apply indexer-level anti-spam controls. |
| 2 | **Brand impersonation**: two different creators can point to the same CID, creating two visually similar but distinct collections. | UIs MUST display `creator_snapshot_caip10` prominently alongside the collection name so users can distinguish them. |
| 3 | **No on-chain governance**: collection rules exist only at the indexer layer. | Add reputation and trust scoring for creators at the API and UI level. |
| 4 | **Creator wallet rotation**: registering agents from a new wallet creates a new creator namespace, fragmenting the collection. | Projects SHOULD keep a stable registration wallet. This is by design, not a protocol flaw. |
| 5 | **Irreversible first write**: human error on the first valid `col` write cannot be undone. | SDKs and UIs SHOULD require explicit confirmation before the first `col` write. |
| 6 | **Collection identity fragmentation**: changing the collection JSON document produces a new CID and a new `collection_key`, splitting the collection across old and new CIDs. | Treat collection versions as intentional "seasons" with documented CID lineage, or use a display override layer. |
| 7 | **Operator and delegate griefing**: a temporary operator can irrevocably lock agents into a garbage collection via first-write-wins. | SDKs and UIs SHOULD warn owners that approving operators grants irrevocable `col` write authority. |
| 8 | **CID document availability**: if the document is unpinned from all IPFS nodes, display metadata is lost (collection identity is preserved but the UI degrades). | Indexers SHOULD cache CID documents locally as an availability fallback (see section 5.6). |
| 9 | **Mempool front-running**: on EVM chains, pending `col` transactions are visible in the public mempool, enabling attackers with operator or delegate permissions to front-run with a malicious collection pointer. On Solana, analogous attacks are possible via MEV and priority fees. | SDKs SHOULD use private transaction submission (e.g., Flashbots Protect, MEV-resistant RPCs) for `col` set operations when available. On Solana, SDKs SHOULD use priority fees for `col` transactions. |
| 10 | **Namespace pollution via transferred agents**: if an agent is transferred after registration, the new owner inherits the original `creator_snapshot_caip10` namespace. Without ownership verification, the new owner could write `col` and place agents into the original creator's collection. | Indexers MUST track Transfer events and verify that the current owner matches `creator_snapshot_caip10` before accepting a `col` write (see section 5.7). |
| 11 | **Dangling sub-collection parents**: a CID document may reference a `parent` CID for which no collection exists in the index, creating an unresolved hierarchy. | Indexers store the parent CID and mark the relationship as unresolved. Depth cascades automatically when matching parent collections are created or finalized (section 5.6). |

<br>

An overarching mitigation is to keep immutable history for audit and dispute analysis (see the history table in section 5.5).

---

## 7. Validation Guidelines

The following guidelines represent the baseline checks that implementations should perform. They are numbered for reference but do not imply a required execution order.

<br>

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

12. The `asset` field must use the canonical `<chain_id_caip2>/<registry_ref>/<agent_local_id>` format.

13. The `block_timestamp` fields must come from chain block headers, not ingestion wall-clock time.

14. CID document resolution must be non-blocking and bounded (section 5.6).

15. Burn and deactivation signals must set `active=false` without mutating the locked identity fields.

16. CID normalization must happen after CID parsing, not by lowercasing the raw input. CIDv0 uses base58, which is case-sensitive and must be parsed before any case conversion (section 5.4, steps 5-7).

17. ASCII whitespace trimming must cover only bytes `0x09`, `0x0A`, `0x0D`, and `0x20`. Non-ASCII whitespace must not be trimmed.

18. The pointer value length after normalization MUST be between 62 and 256 characters.

19. UIs must sanitize all fields from CID documents before rendering. URI-like fields must only allow `ipfs://`, `https://`, and `ar://` schemes.

20. UIs must display `creator_snapshot_caip10` prominently to distinguish collections with the same CID but different creators.

21. On EVM, the `agent_local_id` must be a `uint256` decimal string. On Solana, it must be the Base58 asset account address.

22. On Solana, the `log_index` must be the zero-based ordinal position in the full `meta.logMessages` array, counting all entries.

23. EVM account addresses in CAIP-10 identifiers must be stored in lowercase per CAIP-10. EIP-55 encoding is for display only.

24. Solana indexers MUST detect the `"Log truncated"` sentinel in `meta.logMessages` and flag affected transactions as incomplete.

25. Indexers must track Transfer events and maintain a local ownership table mapping each agent to its current owner (section 5.7).

26. On every `MetadataSet(col)` event, indexers must verify that the current tracked owner matches `creator_snapshot_caip10` before processing the collection write. Writes from non-creator owners must be rejected with `SET_REJECTED_NOT_CREATOR`.

27. All event types (Registration, Transfer, MetadataSet, MetadataDeleted, Burn) must be processed in a single unified queue ordered by canonical sort key. Type-batched processing is prohibited.

28. Ownership state rollback must be symmetric with membership state rollback on chain reorganizations.

29. Cold-start ownership fallback queries must target the specific historical block height (EVM: archive node, Solana: `getBlock` replay). Queries against `latest` state are unsafe for historical event evaluation.

30. `SET_UNVERIFIABLE` events must block further `col` processing for that specific asset until ownership is resolved, to preserve first-write-wins ordering.

31. Registration events must initialize the ownership table entry for the agent, ensuring ownership is known before any same-transaction MetadataSet events.

32. When a resolved CID document contains a `parent` field, indexers must attempt to parse and normalize the parent CID. On success, always store `parent_cid_norm` and `parent_collection_key`, and validate parent existence for depth finalization (section 5.6). On parse failure, ignore the parent field and treat as root (depth 0).

33. Sub-collection depth must not exceed 8 levels. Depth is finalized only when the full ancestry to a root is resolved (`NULL` while pending).

34. Cross-creator sub-collections are not valid. The parent lookup uses the child's `creator_snapshot_caip10`, ensuring only the same creator namespace can form a hierarchy.

---

## 8. Conformance Test Vectors

The following test vectors cover the key behaviors that implementations must handle correctly.

<br>

**1. Solana intra-block conflict.**
A block at slot S contains two valid `MetadataSet(col)` events on the same asset. Event A at `(S, tx_index=0, log_index=10)` sets `col=a`, and event B at `(S, tx_index=1, log_index=3)` sets `col=b`.
Expected: locked value is `col=a`, because event A has a lower `tx_index`.

<br>

**2. Solana failed transaction filter.**
Event A appears in a transaction with `meta.err != null`, and event B appears later in a successful transaction.
Expected: locked value is derived from event B only, since failed transactions must be ignored.

<br>

**3. EVM intra-block conflict.**
Block N contains two `MetadataSet(col)` events on the same asset. Event A at `(N, tx_index=2, log_index=15)` sets `col=a`, and event B at `(N, tx_index=7, log_index=40)` sets `col=b`.
Expected: locked value is `col=a`.

<br>

**4. EVM reorg rollback.**
Canonical history temporarily includes block H1 with the first valid `col=a`. Block H1 is later replaced by canonical block H2 with the first valid `col=b`.
Expected: after rollback and replay, locked value is `col=b`.

<br>

**5. Burn and deactivation handling.**
An agent has an active locked membership. A burn or deactivation signal is ingested.
Expected: `active=false` with the locked `collection_key` and identity fields preserved.

<br>

**6. CIDv0 normalization.**
An agent sets `col=c1:QmYwAPJzv5CZsnA625s3Xf2nemtYgPpHdWEz79ojWnPbdG`. The SDK or indexer parses this as CIDv0 and normalizes it to CIDv1 base32 lowercase.
Expected: `cid_norm` is the CIDv1 base32 equivalent of the original QmY... hash.

<br>

**7. Uppercase base32 input.**
An agent sets `col=c1:BAFYBEIGDYRZT5...` (uppercase). The indexer normalizes to lowercase before storage.
Expected: accepted with a normalized value of `bafybeigdyrzt5...`.

<br>

**8. Self-overwrite attempt.**
An agent has a locked `col=c1:bafyA...` and attempts to set `col=c1:bafyB...` (a different CID).
Expected: rejection with a `SET_REJECTED_LOCKED` history event. Locked value unchanged.

<br>

**9. Factory creator snapshot.**
A factory contract registers an agent. The `Registered` event has `owner=FactoryAddress`. The agent is later transferred to `UserAddress`.
Expected: `creator_snapshot_caip10` remains the factory address, not the user address.

<br>

**10. Two agents, same CID, same creator.**
Agent A and Agent B both set `col=c1:bafySame...` and share the same `creator_snapshot_caip10`.
Expected: identical `collection_key` values; both agents belong to the same collection.

<br>

**11. Solana intra-transaction CPI conflict.**
A single transaction calls `setMetadata(col)` twice via CPI on the same asset. Event A at `(S, tx_index=5, log_index=12)` sets `col=a`, and event B at `(S, tx_index=5, log_index=18)` sets `col=b`.
Expected: locked value is `col=a`, because it has the lower `log_index`.

<br>

**12. Idempotent re-set (same CID).**
An agent has a locked `col=c1:bafyA...` and sets the same value again.
Expected: no state change, `SET_NOOP` history event.

<br>

**13. Delete attempt on locked membership.**
An agent has a locked `col=c1:bafyA...`. A `MetadataDeleted` event with `key == "col"` is ingested.
Expected: rejection with a `DELETE_REJECTED_LOCKED` history event. Locked value unchanged.

<br>

**14. Malformed value handling.**
An agent sets `col=c2:bafybeig...` (wrong prefix).
Expected: treated as malformed, `invalid_reason` stored, no locked membership created. The agent can still set a valid `col` later.

<br>

**15. Two agents, same CID, different creator.**
Agent A (creator `0xAlice`) and Agent B (creator `0xBob`) both set `col=c1:bafySame...`.
Expected: two different `collection_key` values, producing two distinct collections despite sharing the same CID.

<br>

**16. ASCII whitespace trimming.**
An agent sets `col="\t  c1:bafybeigdyrzt5...\n"` (with tabs, spaces, and a newline).
Expected: trimmed to `c1:bafybeigdyrzt5...`, accepted and locked.

<br>

**17. Non-ASCII whitespace rejection.**
An agent sets `col="\u00A0c1:bafybeigdyrzt5..."` (with a leading NBSP character).
Expected: treated as malformed, because NBSP is not trimmed by ASCII whitespace rules and causes CID parsing to fail.

<br>

**18. Transferred agent col write rejection.**
Agent A is registered by `0xAlice` (creator snapshot = `0xAlice`). Agent A is later transferred to `0xBob`. Bob attempts `setMetadata("col", "c1:bafyValid...")`.
Expected: rejected with `SET_REJECTED_NOT_CREATOR`. The agent's `col` slot remains open. If ownership returns to Alice, she can set `col` normally.

<br>

**19. Ownership returned after transfer.**
Agent A is registered by `0xAlice`, transferred to `0xBob`, then transferred back to `0xAlice`. Alice calls `setMetadata("col", "c1:bafyValid...")`.
Expected: accepted and locked normally, because the current owner matches `creator_snapshot_caip10`.

<br>

**20. Same-transaction transfer then col write.**
A single transaction emits `Transfer(0xAlice, 0xBob, agentId)` at `log_index=5`, then `MetadataSet(col)` at `log_index=8`. The agent's `creator_snapshot_caip10` is `0xAlice`.
Expected: ownership table updates to `0xBob` at log_index=5. The col write at log_index=8 is rejected with `SET_REJECTED_NOT_CREATOR` because current_owner (`0xBob`) differs from creator (`0xAlice`).

<br>

**21. Reorg rollback of ownership.**
Block N contains a `Transfer(0xAlice, 0xBob, agentId)`. Block N is orphaned. Block N' (replacement) contains no Transfer for this agent.
Expected: ownership state rolls back to `0xAlice`. Any `col` writes evaluated against the orphaned ownership state must be reverted and reprocessed.

<br>

**22. Sub-collection with valid parent.**
Agent A is registered by `0xAlice` and sets `col=c1:bafyChild...`. The CID document at `bafyChild` contains `"parent": "bafyParent..."`. Alice already has an active locked collection with `collection_key = 0xAlice|bafyParent...` at depth 0.
Expected: after CID document resolution, Agent A's membership is updated with `parent_cid_norm = bafyParent...`, `parent_collection_key = 0xAlice|bafyParent...`, `depth = 1`.

<br>

**23. Sub-collection with non-existent parent.**
Agent A is registered by `0xAlice` and sets `col=c1:bafyChild...`. The CID document contains `"parent": "bafyOrphan..."`. No collection with `collection_key = 0xAlice|bafyOrphan...` exists.
Expected: membership is locked normally (lock state is unaffected). Parent relationship is stored as unresolved: `parent_cid_norm = bafyOrphan...`, `parent_collection_key = 0xAlice|bafyOrphan...`, `depth = NULL`.

<br>

**24. Cross-creator sub-collection attempt.**
Agent A is registered by `0xBob` and sets `col=c1:bafyChild...`. The CID document contains `"parent": "bafyParent..."`. A collection `0xAlice|bafyParent...` exists, but no collection `0xBob|bafyParent...` exists.
Expected: parent relationship is unresolved because the lookup uses Bob's creator namespace, not Alice's.

<br>

**25. Sub-collection max depth exceeded.**
A chain of sub-collections exists: depth 0 → 1 → 2 → ... → 8. Agent A tries to create a sub-collection at depth 9.
Expected: parent relationship is rejected and logged as a diagnostic event. Membership remains valid with `depth = NULL`.

<br>

**26. Same-transaction registration, col write, then transfer (factory flow).**
A factory contract emits `Registration(agentId, factory)` at `log_index=2`, `MetadataSet(col)` at `log_index=5`, and `Transfer(factory, buyer, agentId)` at `log_index=8` in a single transaction. The `creator_snapshot_caip10` is the factory address.
Expected: the `col` write at `log_index=5` is accepted (`SET_LOCKED`) because the owner is still the factory at that point. The ownership table is updated to `buyer` at `log_index=8` after the lock is already in place.

<br>

**27. Same-transaction registration, transfer, then post-transfer col write (hijack prevention).**
A factory emits `Registration(agentId, factory)` at `log_index=2`, `Transfer(factory, buyer, agentId)` at `log_index=5`. The buyer's contract calls `setMetadata(col)` via `onERC721Received` at `log_index=8`.
Expected: the `col` write at `log_index=8` is rejected with `SET_REJECTED_NOT_CREATOR` because the current owner is `buyer`, not the factory. No registration exception overrides this.

---

## 9. Reference Pseudocode

```ts
function trimAsciiWhitespace(s: string): string {
  return s.replace(/^[\x09\x0a\x0d\x20]+|[\x09\x0a\x0d\x20]+$/g, "");
}

// SDK-facing helper — not part of the indexer ingestion pipeline.
function normalizeCollectionCid(input: string): string | null {
  try {
    const raw = trimAsciiWhitespace(input);
    const cid = CID.parse(raw);      // accepts v0 or v1, any case
    return cid.toV1().toString();     // CIDv1 base32lower — already lowercase
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

function verifyOwnership(
  chainIdCaip2: string,
  asset: string,
  creatorSnapshotCaip10: string
): "creator" | "not_creator" | "unknown" {
  const currentOwner = ownershipTable.get(chainIdCaip2, asset);
  if (!currentOwner) return "unknown";
  return currentOwner === creatorSnapshotCaip10 ? "creator" : "not_creator";
}

// NOTE: Callers MUST check for pending SET_UNVERIFIABLE deferrals on this
// asset before calling applyColSet. If a prior event is deferred, further
// col processing MUST be blocked (section 5.7) to preserve first-write-wins.
function applyColSet(
  currentLockedCidNorm: string | null,
  incomingCidNorm: string,
  ownershipStatus: "creator" | "not_creator" | "unknown"
): "SET_LOCKED" | "SET_NOOP" | "SET_REJECTED_LOCKED"
   | "SET_REJECTED_NOT_CREATOR" | "SET_UNVERIFIABLE" {
  // Lock check first: already-locked assets need no ownership verification
  if (currentLockedCidNorm) {
    return currentLockedCidNorm === incomingCidNorm ? "SET_NOOP" : "SET_REJECTED_LOCKED";
  }
  if (ownershipStatus === "unknown") return "SET_UNVERIFIABLE";
  if (ownershipStatus === "not_creator") return "SET_REJECTED_NOT_CREATOR";
  return "SET_LOCKED";
}
```

<br>

**Event ordering comparators:**

```ts
function compareEvmEventOrder(
  a: { blockNumber: bigint; transactionIndex: number; logIndex: number },
  b: { blockNumber: bigint; transactionIndex: number; logIndex: number }
): number {
  if (a.blockNumber !== b.blockNumber)
    return a.blockNumber < b.blockNumber ? -1 : 1;
  if (a.transactionIndex !== b.transactionIndex)
    return a.transactionIndex - b.transactionIndex;
  return a.logIndex - b.logIndex;
}

function compareSolanaEventOrder(
  a: { slot: bigint; txIndex: number; logIndex: number },
  b: { slot: bigint; txIndex: number; logIndex: number }
): number {
  if (a.slot !== b.slot)
    return a.slot < b.slot ? -1 : 1;
  if (a.txIndex !== b.txIndex)
    return a.txIndex - b.txIndex;
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

8. Use canonical `<chain_id_caip2>/<registry_ref>/<agent_local_id>` asset identifiers in indexer storage.

9. Resolve CID documents asynchronously with a bounded fetch policy.

10. Allow post-registration enrollment of agents that do not yet have a locked `col`, via the first valid `setMetadata("col", ...)` call.

11. Track Transfer events and verify agent ownership before accepting `col` writes.

12. Resolve sub-collection parent relationships from CID documents asynchronously, validating same-creator namespace and depth constraints (section 4.1).
