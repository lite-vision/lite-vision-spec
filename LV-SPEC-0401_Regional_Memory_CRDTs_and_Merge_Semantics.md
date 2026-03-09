# LV-SPEC-0401 — Regional Memory CRDTs and Merge Semantics
Version: 1.0  
Status: Draft (Engineering)  
Conformance Level: Core (Mandatory for multi-operator regions)  
Implements: LV-SPEC-0107, LV-SPEC-0204, LV-SPEC-0400  
Language Target: Rust  

---

# 1. Purpose

This specification defines the CRDT framework for Regional Memory.

It formalizes:

- CRDT types used in Lite-Vision
- Merge laws and convergence guarantees
- Conflict resolution policies
- Tombstone handling
- Partition rebalancing
- Anti-entropy synchronization

Regional memory is:

- Mutable
- Eventually consistent
- Not directly anchored in Truth Plane
- Safe from consensus corruption

CRDT design must guarantee convergence without central coordination.

---

# 2. Design Goals

Regional memory MUST:

1. Converge deterministically across replicas.
2. Tolerate partitions and re-merges.
3. Avoid global coordination.
4. Avoid consensus-level locking.
5. Remain independent of Truth Plane safety.

Regional memory must never break consensus invariants.

---

# 3. CRDT Model

Each CRDT MUST satisfy:

Commutativity  
Associativity  
Idempotency  

Merge law:

merge(A, B) = merge(B, A)  
merge(A, merge(B, C)) = merge(merge(A, B), C)  
merge(A, A) = A  

All CRDTs must obey these properties.

---

# 4. Supported CRDT Types

Lite-Vision supports the following CRDT types:

---

## 4.1 G-Counter (Grow-Only Counter)

Definition:

- Monotonically increasing integer.
- Represented as vector indexed by replica_id.

Merge rule:

Take element-wise maximum.

Use cases:

- Request counts
- Uptime tracking
- Non-decreasing metrics

---

## 4.2 PN-Counter (Positive-Negative Counter)

Definition:

- Pair of G-Counters: P and N.
- Value = sum(P) - sum(N)

Merge rule:

Merge P and N independently.

Use cases:

- Resource accounting
- Adaptive scoring

---

## 4.3 OR-Set (Observed-Remove Set)

Definition:

- Set with add/remove support.
- Each add operation tagged with unique ID.

Remove removes specific tagged entries.

Merge rule:

Union of adds minus tombstoned tags.

Use cases:

- Active operator list
- Scene object registry
- Active agent memory entries

---

## 4.4 LWW-Register (Last-Write-Wins Register)

Definition:

- Value with timestamp and replica_id.

Merge rule:

Keep value with highest (timestamp, replica_id).

Used only where semantic correctness allows last-write semantics.

Use cases:

- Soft configuration parameters
- Non-critical annotations

LWW MUST NOT be used for economically sensitive data.

---

## 4.5 OR-Map

Definition:

Map<Key, CRDT>

Each value is itself a CRDT.

Merge rule:

Key union; merge values recursively.

Use cases:

- Agent memory
- Scene semantic memory
- Routing metrics

---

## 4.6 CRDT Graph (Add-Wins Directed Graph)

Definition:

Nodes: OR-Set  
Edges: OR-Set  

Edge removal tracked with tombstones.

Merge rule:

Union nodes and edges minus tombstones.

Used for:

- Semantic scene graph in regional refinement
- Agent relationship graph

---

# 5. Operation Metadata

Each CRDT operation MUST include:

OperationID {
  replica_id: u64,
  logical_clock: u64
}

Logical clock MUST be monotonic per replica.

Replica_id MUST be globally unique within region.

---

# 6. Merge Semantics

Given two replicas R1 and R2:

R_new = merge(R1, R2)

Merge MUST:

- Preserve all causally valid adds
- Respect tombstones
- Avoid resurrection of removed elements
- Remain deterministic

---

# 7. Conflict Resolution Policies

CRDTs avoid conflicts by design.

However, semantic conflicts may arise.

Policies:

- Add-wins (default for sets)
- Remove-wins (if explicitly declared)
- LWW only for soft state

Economically sensitive memory MUST NOT use LWW.

---

# 8. Tombstones

Tombstones represent removed elements.

Tombstone record:

Tombstone {
  operation_id,
  target_id,
  removal_clock
}

Tombstones MUST be:

- Preserved until safe GC threshold
- Propagated during anti-entropy

Premature deletion may cause state resurrection.

---

# 9. Tombstone Garbage Collection

Tombstones MAY be removed if:

- All replicas acknowledge removal.
- Removal clock older than retention window.
- Snapshot includes removal state.

Safe GC condition:

∀ replica r:
r.version ≥ tombstone.removal_clock

GC must be deterministic.

---

# 10. Partition Handling

When region partitions:

Replicas evolve independently.

Upon reconnection:

- Merge invoked.
- All CRDT laws guarantee convergence.

No global coordinator required.

---

# 11. Partition Rebalancing

If operators join/leave region:

Rebalancing MUST:

- Assign new replica_id
- Maintain unique identity
- Reconcile CRDT state via merge

Replica removal MUST:

- Freeze replica state
- Merge before removal
- Ensure no orphan tombstones

---

# 12. Anti-Entropy Protocol

Anti-entropy ensures eventual consistency.

Protocol options:

1. Periodic full state sync
2. Version-vector delta sync
3. Merkle-tree diff sync

Recommended approach:

Merkle-tree diff of CRDT state.

Procedure:

1. Exchange root hash.
2. If mismatch → descend tree.
3. Exchange missing operations.

Anti-entropy MUST be bandwidth-efficient.

---

# 13. Version Vectors

Each replica maintains:

VersionVector {
  replica_id → logical_clock
}

Merge of version vectors:

Element-wise max.

Used to detect missing updates.

---

# 14. Determinism Constraints

Regional memory is eventually consistent.

However, if regional snapshot is committed:

snapshot_hash = H(canonical_CRDT_state)

Canonical ordering MUST be:

- Keys sorted lexicographically
- Replica IDs sorted ascending
- Tombstones sorted deterministically

Snapshot must be reproducible.

---

# 15. Reconciliation With Committed Memory

Regional memory MAY be snapshotted and anchored.

When snapshot anchored:

- Snapshot becomes immutable reference.
- Future regional memory evolves independently.
- Snapshot must not mutate.

Snapshot anchoring optional.

---

# 16. Cross-Region Merges

If regions merge permanently:

- Replica ID namespace must be reconciled.
- ID collisions must be remapped deterministically.
- Merge must follow CRDT laws.

Governance MAY restrict cross-region merges.

---

# 17. Security Considerations

Regional memory must assume:

- Byzantine operators may submit malformed CRDT ops.

Validation rules:

- Reject malformed operation_id.
- Reject non-monotonic logical clocks.
- Reject invalid tombstone references.
- Enforce operation signature if required.

CRDT operations MAY be signed by replica.

---

# 18. Rust Implementation Requirements

Implementation MUST:

- Use canonical ordering (BTreeMap preferred).
- Use fixed-width integers for clocks.
- Implement merge as pure function.
- Avoid floating-point arithmetic in CRDT state.
- Validate logical clock monotonicity.
- Support snapshot hashing.

CRDT operations must be deterministic under identical input.

---

# 19. Compliance Requirements

To claim LV-SPEC-0401 compliance:

Implementation MUST:

- Support G-Counter
- Support PN-Counter
- Support OR-Set
- Support OR-Map
- Support CRDT Graph
- Implement tombstone handling
- Implement version vectors
- Implement anti-entropy protocol
- Preserve merge laws

---

# 20. Design Invariants

1. CRDT merges must be commutative.
2. CRDT merges must be associative.
3. CRDT merges must be idempotent.
4. Tombstones must prevent resurrection.
5. Regional memory must never affect consensus safety.
6. Snapshot hashing must be deterministic.
7. No floating-point nondeterminism allowed in CRDT state.

---

# 21. Summary

LV-SPEC-0401 defines Regional Memory CRDTs:

- Supported CRDT types
- Merge laws
- Conflict resolution policies
- Tombstone semantics
- Partition handling
- Anti-entropy synchronization

It enables:

Scalable regional intelligence.
Eventually consistent collaboration.
Partition tolerance.
Deterministic snapshot anchoring.
Consensus independence.

Lite-Vision maintains:

Mutable regional cognition.
Immutable committed memory.
CPU-secured truth.
GPU-powered intelligence.

---

End of LV-SPEC-0401