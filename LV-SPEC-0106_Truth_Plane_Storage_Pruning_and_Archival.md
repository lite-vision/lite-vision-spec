# LV-SPEC-0106 — Truth Plane Storage, Pruning, and Archival
Version: 1.0  
Status: Draft (Engineering)  
Conformance Level: Core (Mandatory)  
Implements: LV-SPEC-0100, LV-SPEC-0101, LV-SPEC-0103  
Language Target: Rust  

---

# 1. Purpose

This specification defines:

- The Truth Plane state partitioning and sharding model
- Snapshotting and compaction procedures
- Pruning rules and invariants
- Archive node behavior
- Historical state proof mechanisms

The Truth Plane MUST support long-term sustainability through controlled storage lifecycle management while preserving full verifiability.

---

# 2. Storage Model Overview

Truth Plane storage consists of:

1. Block storage
2. State storage
3. Index storage
4. Snapshot storage
5. Archive storage (optional but supported)

State at height h:

T_h

State root:

state_root_h = MerkleRoot(T_h)

All pruning MUST preserve the ability to verify:

- Block hashes
- State roots
- Transaction inclusion proofs
- Receipt inclusion proofs

---

# 3. State Partitioning and Sharding

## 3.1 Logical State Domains

Truth Plane state is logically partitioned:

T = {
  Accounts,
  Validators,
  Stake,
  Governance,
  Operators,
  Receipts,
  Partitions
}

Each domain maintains its own Merkle sub-root.

Global state_root = H(concat(sub_roots))

---

## 3.2 Sharding Model (Optional Advanced Mode)

Truth Plane MAY implement horizontal sharding:

T = ⋃ T_k

Where each shard T_k:

- Maintains local state subset
- Maintains local Merkle root
- Commits root into global root

Cross-shard transactions MUST:

- Be atomically committed
- Include cryptographic proof references
- Preserve global determinism

Shard addition/removal MUST occur at epoch boundary.

---

# 4. Snapshotting

## 4.1 Snapshot Definition

Snapshot S_h is:

- Complete state T_h
- At block height h
- Represented as canonical serialized state tree

Snapshots MUST include:

- state_root_h
- block_hash_h
- validator_set_hash_h

---

## 4.2 Snapshot Frequency

Snapshot interval:

Every K blocks OR every epoch boundary.

Governance MAY adjust interval.

---

## 4.3 Snapshot Integrity

Snapshot validity requires:

H(snapshot_data) matches state_root_h

Snapshot MUST be verifiable independently of live node.

---

# 5. Compaction

Compaction reduces storage overhead without altering state root.

## 5.1 Compaction Targets

- Transaction index
- Intermediate trie nodes
- Receipt cache
- Historical intermediate states

Compaction MUST NOT:

- Alter block hashes
- Alter state roots
- Remove required proof paths

---

## 5.2 Determinism

Compaction MUST:

- Be deterministic
- Produce identical resulting state on replay
- Preserve Merkle integrity

---

# 6. Pruning

Pruning removes obsolete historical state while preserving proof capability.

---

## 6.1 Prunable Data

Nodes MAY prune:

- Transaction bodies older than retention window
- Intermediate state snapshots
- Execution traces
- Historical receipt details (if anchored)

Nodes MUST NOT prune:

- Block headers
- State roots
- Validator set history
- Slashing proofs
- Governance decisions

---

## 6.2 Retention Policy

Retention window:

R_blocks OR R_epochs

Governance MAY adjust retention bounds.

Nodes MAY choose longer retention but MUST NOT choose shorter than protocol minimum.

---

# 7. Pruning Invariants

Pruning MUST preserve:

1. Block hash chain integrity
2. State root reproducibility
3. Transaction inclusion proofs
4. Receipt inclusion proofs
5. Slashing proof verification
6. Governance audit trail

Formally:

For any pruned block b < h:

It MUST still be possible to verify:

- block_hash_b
- state_root_b
- Merkle proof for tx_b

Either via:

- Snapshot
- Archive node
- Historical proof package

---

# 8. Archive Nodes

Archive nodes store:

- Full block bodies
- Full transaction history
- All historical state
- All receipts

Archive nodes provide:

- Historical proof service
- Forensic replay
- Governance audits
- Compliance exports

Archive nodes are NOT required for consensus participation.

---

# 9. Historical State Proof

To prove account state at height h:

Provide:

1. Block header at height h
2. state_root_h
3. Merkle proof from state_root_h to account leaf

Verification MUST require:

- Only block header
- Merkle proof
- Hash function

No access to full historical state required.

---

# 10. Receipt and Artifact Proofs

Receipt proof requires:

- Block header
- receipt_root
- Merkle proof to receipt leaf

Artifact verification requires:

- receipt entry
- artifact_hash
- content hash check off-plane

Truth Plane does NOT store artifact bytes.

---

# 11. Light Client Support

Light clients MUST be able to:

- Verify block headers
- Verify QC signatures
- Verify Merkle proofs for transactions and receipts
- Verify validator set transitions

Light clients MUST NOT require:

- Full state storage
- Full transaction history

---

# 12. State Reconstruction

Given:

- Genesis block
- Ordered blocks
- Deterministic δ

Full node MUST be able to reconstruct state_root_h exactly.

Snapshot MAY accelerate reconstruction.

---

# 13. Disk Layout Recommendations (Rust)

Implementations SHOULD:

- Separate hot and cold storage
- Use append-only block files
- Store Merkle nodes in key-value store
- Use ordered key structures (e.g., BTreeMap)
- Maintain canonical serialization boundaries

Unsafe optimizations MUST NOT break determinism.

---

# 14. Partition Deletion

If a partition is removed via governance:

- Final state snapshot MUST be stored
- Partition Merkle root MUST remain verifiable
- Removal MUST not invalidate prior state_root

Partition deletion MUST:

- Be epoch-bound
- Preserve historical proofs

---

# 15. Storage Tiering

Truth Plane nodes MAY implement:

Tier 1 — Active State  
Tier 2 — Snapshot Cache  
Tier 3 — Archive  

Movement between tiers MUST preserve:

- Hash integrity
- Proof verifiability
- Replay determinism

---

# 16. Safety Guarantees

Storage and pruning MUST NOT:

- Change historical block hashes
- Modify finalized state roots
- Invalidate slashing proofs
- Break light client verification
- Introduce non-determinism

---

# 17. Compliance Requirements

To claim LV-SPEC-0106 compliance, implementation MUST:

- Support state snapshotting
- Support safe pruning
- Preserve Merkle root integrity
- Provide proof-of-inclusion verification
- Support archive mode
- Preserve partition boundaries
- Maintain deterministic state reconstruction

---

# 18. Design Invariants

1. Pruning must never alter consensus history.
2. State roots are immutable once finalized.
3. Snapshots must match canonical state root.
4. Partition removal must preserve auditability.
5. Archive nodes extend, not replace, consensus nodes.
6. Storage lifecycle management must remain CPU-friendly.

---

# 19. Summary

LV-SPEC-0106 defines the Truth Plane storage lifecycle:

- Partitioned state
- Snapshotting
- Deterministic compaction
- Safe pruning
- Archive preservation
- Historical proof guarantees

It ensures that Lite-Vision remains:

Sustainable.
Auditable.
Deterministic.
CPU-secured.

---

End of LV-SPEC-0106