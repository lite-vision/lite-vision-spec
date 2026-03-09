# LV-SPEC-0304 — RPACK Delta Updates (RPACK-Δ)
Version: 1.0  
Status: Draft (Engineering)  
Conformance Level: Core (Mandatory for streaming / live updates)  
Implements: LV-SPEC-0107, LV-SPEC-0203, LV-SPEC-0301, LV-SPEC-0302, LV-SPEC-0303  
Language Target: Rust  

---

# 1. Purpose

This specification defines RPACK-Δ (RPACK Delta Updates).

It formalizes:

- Patch semantics
- Conflict resolution rules
- Progressive refinement strategies
- Idempotency guarantees
- Ordering constraints

RPACK-Δ enables:

- Live scene updates
- Incremental streaming
- Progressive asset refinement
- Efficient network usage

The network still:

- Anchors only delta_hash
- Never renders pixels
- Never executes renderer logic

---

# 2. Conceptual Model

Given:

Base RPACK: R₀  
Delta: Δ₁  
Result: R₁ = Apply(R₀, Δ₁)

Subsequent deltas:

R₂ = Apply(R₁, Δ₂)

Each delta is content-addressed:

DeltaID = H(canonical_delta_bytes)

Truth Plane MAY anchor:

delta_hash
base_hash
result_hash (optional)

---

# 3. Delta Structure

RPACKDelta {
  version: u32,
  base_rpack_hash: Hash32,
  delta_id: Hash32,
  sequence_number: u64,
  operations: Vec<DeltaOp>,
  metadata: DeltaMetadata
}

Delta MUST specify base_rpack_hash.

Applying delta to wrong base MUST fail.

---

# 4. Delta Operations

DeltaOp {
  op_type: enum,
  target_path: String,
  payload: Option<bytes>,
  expected_hash: Option<Hash32>
}

Supported operations:

1. AddNode
2. RemoveNode
3. UpdateTransform
4. UpdateMaterial
5. ReplaceGeometry
6. AddConstraint
7. RemoveConstraint
8. AddAssetRef
9. RemoveAssetRef
10. MetadataUpdate
11. ChunkReplace
12. ChunkAppend

Operations MUST be deterministic.

---

# 5. Patch Semantics

## 5.1 AddNode

Adds new node with full specification.

Fails if:

- node_id already exists
- parent_id invalid

---

## 5.2 RemoveNode

Removes node and optionally children.

Policy defined by:

RemovePolicy:
- Cascade
- RejectIfChildrenExist

---

## 5.3 UpdateTransform

Replaces full transform.

Partial transform updates NOT allowed in canonical mode.

---

## 5.4 ReplaceGeometry

Replaces geometry_ref with new AssetID.

Requires:

expected_hash of prior geometry.

---

# 6. Conflict Detection

Conflict occurs if:

- expected_hash does not match current target state
- sequence_number out of order
- base_rpack_hash mismatch

Conflict resolution strategies:

1. Reject delta
2. Request rebase
3. Apply in forked branch (optional enterprise mode)

Core mode MUST reject conflicting delta.

---

# 7. Progressive Refinement

RPACK-Δ supports progressive refinement.

Example:

Δ₁: Low-res geometry  
Δ₂: Medium-res geometry  
Δ₃: High-res geometry  

Each delta:

- Replaces geometry_ref
- Preserves node identity
- Improves asset fidelity

Clients MAY render intermediate states.

---

# 8. Progressive Asset Strategy

Progressive strategy MAY include:

- Mesh LOD refinement
- Texture resolution upgrade
- Shader enhancement
- Animation refinement
- Physics detail increase

Refinement MUST:

- Maintain semantic identity
- Not change logical object meaning
- Be reversible via deterministic ordering

---

# 9. Idempotency Rules

A delta is idempotent if:

Apply(R, Δ) twice → same result.

To ensure idempotency:

- sequence_number MUST strictly increase
- DeltaID MUST be unique
- target_path MUST uniquely identify element
- expected_hash MUST guard mutation

If same delta applied twice:

Second application MUST detect prior application and no-op.

---

# 10. Ordering Rules

## 10.1 Strict Ordering

Deltas MUST be applied in:

sequence_number ascending order

If sequence gap detected:

Client MUST:

- Pause application
- Request missing delta

---

## 10.2 Deterministic Ordering

If deterministic mode required:

- Delta ordering MUST be canonical.
- sequence_number must be contiguous.
- No reordering permitted.

---

## 10.3 Soft Mode Ordering

In soft mode:

Out-of-order deltas MAY be buffered.

Application order still must respect sequence_number.

---

# 11. Delta Anchoring

Truth Plane MAY anchor:

DeltaAnchor {
  base_rpack_hash,
  delta_hash,
  result_hash,
  block_height
}

Anchoring delta ensures:

- Integrity
- Ordering guarantee
- Dispute accountability

Truth Plane does NOT validate delta semantics.

---

# 12. Delta Validation Procedure

Client MUST:

1. Verify delta_hash.
2. Verify base_rpack_hash matches local RPACK.
3. Validate sequence_number.
4. For each operation:
   - Validate target exists.
   - Validate expected_hash (if provided).
5. Apply operations in order.
6. Recompute new RPACK_hash.
7. Optionally verify result_hash.

Failure at any step → reject delta.

---

# 13. Streaming Use Case

For live scenes:

- Initial RPACK loads baseline.
- RPACK-Δ streams incremental updates.
- Client applies progressively.

Streaming flag in header MUST indicate delta compatibility.

---

# 14. Branching and Forking (Optional)

Enterprise mode MAY allow:

Parallel branches:

R₀ → Δ₁ → R₁  
R₀ → Δ₁′ → R₁′  

Core mode does NOT support branching.

Fork resolution must be explicit.

---

# 15. Delta Size Constraints

Delta size MUST respect:

JobTicket.max_output_size

Large deltas SHOULD be chunked.

Delta operations MUST not exceed governance-defined max operations per delta.

---

# 16. Security Considerations

Clients MUST:

- Validate hashes before applying.
- Reject invalid base references.
- Limit operation count to prevent DoS.
- Enforce memory bounds.

Deltas are untrusted until verified.

---

# 17. Determinism Constraints

If deterministic mode:

- All operations must be fully specified.
- No partial patch ambiguity.
- Floating-point transforms must be exact.
- Node ordering must be stable.
- Asset replacement must be exact.

Replay applying same deltas must produce identical RPACK_hash.

---

# 18. Rust Implementation Requirements

Implementation MUST:

- Use canonical path resolution.
- Use stable ordering (BTree structures).
- Reject ambiguous target_path.
- Enforce sequence_number monotonicity.
- Prevent double-application of same DeltaID.
- Validate expected_hash before mutation.

Floating-point nondeterminism MUST be avoided in deterministic mode.

---

# 19. Compliance Requirements

To claim LV-SPEC-0304 compliance:

Implementation MUST:

- Implement RPACKDelta structure.
- Support Add/Remove/Update operations.
- Enforce base hash validation.
- Enforce idempotency.
- Enforce strict ordering.
- Support progressive refinement.
- Support hash verification.

Branching support optional in Core.

---

# 20. Design Invariants

1. Delta must reference explicit base.
2. Deltas must be idempotent.
3. Ordering must be deterministic.
4. Conflicts must not silently resolve.
5. Pixel rendering remains client-side.
6. Hash anchoring remains Truth Plane responsibility.

---

# 21. Summary

LV-SPEC-0304 defines RPACK-Δ:

- Deterministic patch semantics
- Conflict detection
- Progressive refinement
- Idempotency guarantees
- Strict ordering rules

It enables:

Live updates.
Efficient streaming.
High-fidelity refinement.
Deterministic replay.

Lite-Vision remains:

Artifact-anchored.
Client-rendered.
GPU-compute focused.
CPU-consensus secured.

---

End of LV-SPEC-0304