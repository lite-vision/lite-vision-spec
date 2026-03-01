# LV-SPEC-0403 — Storage Tiering, Compaction, and Garbage Collection
Version: 1.0  
Status: Draft (Engineering)  
Conformance Level: Core (Mandatory for persistent nodes)  
Implements: LV-SPEC-0106, LV-SPEC-0205, LV-SPEC-0301, LV-SPEC-0400, LV-SPEC-0401, LV-SPEC-0402  
Language Target: Rust  

---

# 1. Purpose

This specification defines Storage Tiering, Compaction, and Garbage Collection (GC) for Lite-Vision.

It formalizes:

- Hot / Warm / Cold storage tiers
- Data movement policy between tiers
- Compaction algorithms and triggers
- GC safety invariants
- Proof preservation guarantees

All storage operations MUST preserve:

- Hash commitments
- Historical verifiability
- Deterministic replay
- Cross-partition proof validity

Storage optimization MUST NEVER break commitments.

---

# 2. Storage Scope

This specification applies to:

Truth Plane:
- Blocks
- State trie
- Receipt logs
- Partition snapshots

Intelligence Plane:
- RPACK artifacts
- Regional memory snapshots
- Asset caches
- CRDT state logs

---

# 3. Storage Tiers

Lite-Vision defines three storage tiers.

---

## 3.1 Hot Tier

Characteristics:

- Low latency
- Frequently accessed
- Mutable or recently committed
- SSD or memory-backed

Examples:

- Latest blocks
- Active partition state
- Pending receipts
- Active RPACK-Δ streams
- Regional CRDT active state

Retention window configurable.

---

## 3.2 Warm Tier

Characteristics:

- Moderate latency
- Less frequently accessed
- Immutable historical data
- Optimized for storage efficiency

Examples:

- Historical blocks (within retention window)
- Settled receipts
- Archived RPACK artifacts
- Regional memory snapshots

---

## 3.3 Cold Tier

Characteristics:

- High latency
- Long-term archival
- Immutable
- Possibly external storage

Examples:

- Old partition snapshots
- Historical state roots
- Tombstones
- Audit records

Cold tier MUST preserve proof verifiability.

---

# 4. Tier Movement Policy

Data movement MUST follow deterministic policy.

---

## 4.1 Hot → Warm Transition

Trigger conditions:

- Block height older than threshold
- Receipt settlement window expired
- No active dispute
- No active cross-partition reference

Transition MUST:

- Preserve Merkle roots
- Preserve inclusion proofs
- Maintain index for retrieval

---

## 4.2 Warm → Cold Transition

Trigger conditions:

- Governance-defined retention threshold
- Snapshot archived
- No pending audit window

Cold movement MUST:

- Preserve state_root
- Preserve proof reconstruction capability
- Maintain block header availability

---

# 5. Compaction Model

Compaction reduces storage footprint while preserving hash integrity.

Compaction MUST NOT:

- Modify committed data semantics
- Alter block ordering
- Break Merkle proof reconstruction

---

# 6. Truth Plane Compaction

## 6.1 State Trie Compaction

Allowed operations:

- Remove unreachable nodes
- Deduplicate identical subtrees
- Repack key-value storage

Invariant:

state_root MUST remain identical.

If compaction alters hash → invalid.

---

## 6.2 Block Log Compaction

Block bodies MAY be pruned if:

- Block header retained
- Receipt_root retained
- Inclusion proof reconstructable
- Archival nodes store full body

Compaction MUST preserve:

BlockHeaderHash.

---

## 6.3 Receipt Log Compaction

Receipts MAY be compacted to:

- receipt_hash
- resource_hash
- output_hash

Full receipt bytes MAY be pruned after window expires.

Inclusion proof MUST remain valid.

---

# 7. Intelligence Plane Compaction

---

## 7.1 RPACK Compaction

RPACK artifacts MAY:

- Deduplicate identical chunks
- Remove unused chunk references
- Recompress with approved algorithm

Constraint:

RPACK_hash MUST remain unchanged if canonical RPACK unchanged.

If compaction changes canonical bytes → new artifact required.

---

## 7.2 CRDT Log Compaction

Regional CRDT logs MAY:

- Collapse operations into snapshot
- Remove obsolete tombstones (safe GC condition)
- Compress operation history

Snapshot hash MUST be deterministic.

---

# 8. Garbage Collection (GC)

GC MUST be conservative.

GC Safety Rule:

No object may be garbage-collected if referenced by:

- Active block
- Unexpired dispute window
- Snapshot root
- Cross-partition proof
- Anchored committed memory

---

# 9. GC Safety Invariants

GC MUST NOT:

1. Delete data required to reconstruct state_root.
2. Delete receipt referenced by fraud window.
3. Delete RPACK referenced by active job.
4. Delete tombstone before safe threshold.
5. Delete partition tombstone.
6. Delete snapshot used for migration proof.

---

# 10. Reference Counting Model

Objects MAY be tracked via:

ReferenceCount {
  object_hash,
  reference_count
}

References include:

- Block inclusion
- Partition snapshot inclusion
- RPACK-Δ base linkage
- CRDT snapshot linkage

GC may delete only when:

reference_count == 0
AND
retention_window_expired

---

# 11. Snapshot-Based Compaction

State snapshot at block height H:

snapshot_root = H(canonical_state)

After snapshot:

- Pre-snapshot deltas MAY be pruned.
- Reconstructability MUST remain intact.
- Snapshot MUST include full state.

Snapshot MUST be verifiable.

---

# 12. Tombstone Retention

CRDT tombstones MUST remain until:

- All replicas have acknowledged removal.
- Safe GC condition satisfied.
- Snapshot includes removal state.

Premature tombstone removal is forbidden.

---

# 13. Partition GC

If partition deleted:

- Final snapshot retained.
- Tombstone retained.
- Partition ID never reused.

Cold storage MUST preserve:

Final state_root and tombstone.

---

# 14. RPACK-Δ GC

Delta chain:

R₀ → Δ₁ → Δ₂ → Δ₃

If snapshot at R₃ created:

Earlier deltas MAY be removed if:

- Snapshot anchored
- Delta application reconstructable
- No active replay requirement

---

# 15. Audit Window Constraints

GC MUST respect:

- verification_window_blocks
- appeal_window_blocks
- archival_retention_policy

Data under active audit MUST NOT be pruned.

---

# 16. Storage Cost Model

Governance MAY define:

- Storage cost per byte
- Cold storage incentives
- Archival rewards
- Snapshot compression rewards

Storage cost MUST not incentivize unsafe pruning.

---

# 17. Failure Handling

If GC removes required data erroneously:

Node must:

- Mark itself invalid
- Resync from archival peer
- Not participate in consensus until repaired

GC failure MUST NOT propagate corrupted state.

---

# 18. Rust Implementation Requirements

Implementation MUST:

- Use canonical hashing before and after compaction
- Validate state_root unchanged
- Maintain reference tracking
- Use atomic file replacement
- Prevent partial compaction corruption
- Avoid floating-point arithmetic in storage logic

Compaction must be deterministic.

---

# 19. Compliance Requirements

To claim LV-SPEC-0403 compliance:

Implementation MUST:

- Implement three-tier storage
- Implement deterministic compaction
- Preserve hash invariants
- Enforce GC safety rules
- Respect retention windows
- Preserve partition tombstones
- Support snapshot-based compaction

---

# 20. Design Invariants

1. Hash commitments must never change.
2. Compaction must preserve state_root.
3. GC must not break proof reconstruction.
4. Tombstones must not be prematurely deleted.
5. Snapshot must be canonical and reproducible.
6. Cold storage must preserve audit capability.
7. Pixel buffers remain excluded from committed storage.

---

# 21. Summary

LV-SPEC-0403 defines storage lifecycle:

- Hot / Warm / Cold tiers
- Deterministic compaction
- Conservative garbage collection
- Proof-preserving pruning
- Snapshot-driven optimization

It ensures:

Scalable storage.
Long-term verifiability.
Efficient compaction.
Safe pruning.
Immutable commitments.

Lite-Vision maintains:

CPU-secured truth.
GPU-powered intelligence.
Hash-anchored state.
Client-rendered outputs.
Safe storage evolution.

---

End of LV-SPEC-0403