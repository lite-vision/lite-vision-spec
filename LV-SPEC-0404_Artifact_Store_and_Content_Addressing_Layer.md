# LV-SPEC-0404 — Artifact Store and Content Addressing Layer
Version: 1.0  
Status: Draft (Engineering)  
Conformance Level: Core (Mandatory for artifact-serving nodes)  
Implements: LV-SPEC-0107, LV-SPEC-0205, LV-SPEC-0301, LV-SPEC-0303, LV-SPEC-0400, LV-SPEC-0403  
Language Target: Rust  

---

# 1. Purpose

This specification defines the Artifact Store and Content Addressing Layer (ACAL) of Lite-Vision.

It formalizes:

- Artifact lifecycle (ingest → replicate → pin → prune)
- Content addressing model
- Integrity proofs and verification
- Replication policies
- Optional encryption at rest
- Optional access control hooks

Artifacts include:

- RPACK containers
- RPACK-Δ deltas
- Regional memory snapshots (optional)
- Model checkpoints
- Kernel binaries
- Auxiliary structured outputs

The Truth Plane anchors hashes.
The Artifact Store manages bytes.

---

# 2. Content Addressing Model

## 2.1 Artifact ID

ArtifactID = H(canonical_artifact_bytes)

ArtifactID MUST:

- Be 32 bytes
- Use canonical hash function (LV-SPEC-0104)
- Be immutable
- Uniquely identify content

ArtifactID is globally stable.

---

## 2.2 Canonical Bytes Rule

Artifact hash MUST be computed over:

Canonical serialized bytes

Compression and encryption MUST NOT alter canonical hash semantics.

If artifact is stored compressed or encrypted:

Hash MUST represent plaintext canonical bytes.

---

# 3. Artifact Types

ArtifactType enum:

- RPACK
- RPACK_DELTA
- MODEL_CHECKPOINT
- KERNEL_BINARY
- SNAPSHOT
- METADATA
- EXTENSION

ArtifactType MUST be included in metadata.

---

# 4. Artifact Lifecycle

Artifact lifecycle states:

Ingested → Verified → Replicated → Pinned → Prunable → Pruned

Transitions are deterministic.

---

## 4.1 Ingest

Ingest occurs when:

- Operator produces artifact
- Client uploads artifact
- Node receives replicated artifact

Ingest steps:

1. Receive bytes.
2. Compute ArtifactID.
3. Validate against declared ID.
4. Store temporarily.
5. Mark Verified if hash matches.

---

## 4.2 Verified

Artifact is verified if:

H(bytes) == ArtifactID

Unverified artifacts MUST NOT be served.

---

## 4.3 Replicated

Replication policies determine:

- Number of replicas
- Geographic distribution
- Partition-aware placement

Replication MUST:

- Preserve canonical bytes
- Preserve ArtifactID
- Avoid modification

---

## 4.4 Pin

Pin prevents artifact from pruning.

Pin may be triggered by:

- Active job reference
- Governance rule
- User request
- Partition migration
- Snapshot anchoring

Pinned artifact MUST NOT be pruned.

---

## 4.5 Prune

Artifact eligible for pruning if:

- reference_count == 0
- retention window expired
- not pinned
- not required for audit

Pruning MUST NOT break:

- Proof reconstruction
- Active delta chains
- Anchored commitments

---

# 5. Reference Tracking

Each artifact MUST track:

ArtifactRef {
  artifact_id,
  reference_count,
  pin_count,
  retention_expiry_block
}

Reference increments when:

- RPACK references asset
- Delta references base
- Snapshot references artifact
- Job references output

Reference decrements when reference removed.

Pruning only allowed when:

reference_count == 0 AND pin_count == 0

---

# 6. Replication Policies

Replication MAY be:

- Fixed replication factor (N replicas)
- Region-aware replication
- Cost-aware replication
- Governance-controlled replication

Replication MUST NOT:

- Modify artifact bytes
- Change ArtifactID
- Violate content addressing

---

## 6.1 Minimum Replication Guarantee

Governance MAY define:

MinReplicationFactor

If replication falls below threshold:

- Node SHOULD replicate artifact
- Network MAY incentivize replication

---

# 7. Integrity Proofs

Artifact integrity proven via:

ArtifactProof {
  artifact_id,
  inclusion_proof,
  storage_signature
}

Proof types:

- Merkle inclusion proof
- Storage provider signature
- Snapshot inclusion proof

ArtifactProof MUST allow:

Verification of integrity without downloading full artifact (if chunked).

---

# 8. Chunked Artifacts

Large artifacts MAY be chunked.

ChunkID = H(chunk_bytes)

Artifact may be represented as:

Merkle tree of chunks.

Root hash = ArtifactID.

Client MAY:

- Verify chunk hash
- Verify Merkle proof
- Stream partial artifact

Chunking MUST preserve canonical semantics.

---

# 9. Encryption at Rest (Optional)

Encryption at rest MAY be applied.

Encryption requirements:

- ArtifactID computed on plaintext.
- Ciphertext stored separately.
- KeyID associated with artifact.

EncryptionMetadata {
  encryption_algo: u16,
  key_id: Hash32,
  nonce: bytes
}

Encryption MUST:

- Not alter ArtifactID
- Allow deterministic decryption
- Support key rotation (new ciphertext, same ArtifactID)

Encryption is optional in Core.

---

# 10. Access Control (Optional)

Artifact access control MAY be enforced at storage layer.

AccessControlPolicy {
  read_roles: Vec<Role>,
  write_roles: Vec<Role>,
  time_constraints: Option<BlockRange>
}

Truth Plane does NOT enforce access control.

Access control enforced by storage nodes.

Access control MUST NOT affect:

Artifact integrity.
ArtifactID.

---

# 11. Garbage Collection Safety

GC MUST NOT prune artifact if:

- Referenced by committed memory
- Within dispute window
- Required for deterministic replay
- Part of delta chain
- Pinned by governance

GC must validate reference_count and pin_count.

---

# 12. Artifact Mutation Prohibition

Artifacts are immutable.

Mutation requires:

- New artifact
- New ArtifactID
- New anchor (if committed)

In-place mutation forbidden.

---

# 13. Cross-Partition Artifact Availability

If partition deleted:

Artifacts referenced by partition snapshot MUST remain accessible.

Cold storage MUST retain:

- ArtifactID
- Proof metadata

Artifacts may migrate across partitions without changing ID.

---

# 14. Failure Handling

If artifact corrupted:

- Mark invalid
- Remove corrupted copy
- Fetch from replica
- Re-verify

Node MUST NOT serve corrupted artifact.

---

# 15. Rust Implementation Requirements

Implementation MUST:

- Use canonical hashing for ArtifactID
- Validate hash before serving
- Implement reference tracking
- Support chunked storage
- Prevent in-place mutation
- Use atomic writes for ingest
- Avoid floating-point arithmetic in storage logic

Storage index must be deterministic.

---

# 16. Compliance Requirements

To claim LV-SPEC-0404 compliance:

Implementation MUST:

- Implement ArtifactID content addressing
- Support artifact lifecycle transitions
- Implement replication policy
- Implement reference tracking
- Enforce pruning safety
- Support optional encryption metadata
- Support optional access control hooks
- Preserve hash invariants

---

# 17. Design Invariants

1. ArtifactID is immutable.
2. Artifact bytes must match ArtifactID.
3. Pruning must not break commitments.
4. Encryption must not alter ArtifactID.
5. Access control must not affect integrity.
6. Delta chains must remain reconstructable.
7. No artifact mutation allowed.

---

# 18. Summary

LV-SPEC-0404 defines the Artifact Store:

- Content-addressed storage
- Lifecycle management
- Replication and pinning
- Safe pruning
- Integrity proofs
- Optional encryption and access control

It ensures:

Scalable artifact distribution.
Deterministic verification.
Proof-preserving storage.
Immutable content semantics.
Safe lifecycle evolution.

Lite-Vision maintains:

Hash-anchored truth.
GPU-powered intelligence.
Client-rendered artifacts.
Immutable artifact integrity.

---

End of LV-SPEC-0404