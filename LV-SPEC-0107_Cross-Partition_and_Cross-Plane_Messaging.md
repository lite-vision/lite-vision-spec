# LV-SPEC-0107 — Cross-Partition and Cross-Plane Messaging
Version: 1.0  
Status: Draft (Engineering)  
Conformance Level: Core (Mandatory)  
Implements: LV-SPEC-0100, 0101, 0103, 0106  
Language Target: Rust  

---

# 1. Purpose

This specification defines:

- Cross-partition (cross-shard) proof format
- Cross-plane (Truth ↔ Intelligence) commit/verify flows
- Anti-replay protections
- Ordering constraints where required

This document governs how:

- Truth Plane partitions communicate safely
- Intelligence artifacts are committed and verified
- Messages are proven without breaking determinism
- Replay attacks are prevented

All cross-boundary communication MUST preserve:

- Deterministic verification
- BFT safety
- Hash-stable commitments

---

# 2. Cross-Partition (Cross-Shard) Messaging

## 2.1 Partition Model

Truth Plane may be partitioned:

T = ⋃ T_k

Each partition T_k maintains:

- Local state
- Local Merkle root
- Block height
- Validator subset (optional advanced mode)

Global state_root includes all partition roots.

---

## 2.2 Cross-Partition Message Definition

A CrossPartitionMessage (CPM) is:

- source_partition_id
- target_partition_id
- source_block_height
- payload_hash
- Merkle_proof_to_state_root
- signature(s)

The payload itself MAY reside off-partition but MUST be hash-committed.

---

## 2.3 Cross-Shard Proof Format

Cross-shard proof MUST include:

1. Source block header
2. Source partition root
3. Merkle proof from partition root to message leaf
4. Proof that partition root is included in global state_root
5. QC for source block

Verification steps:

1. Verify QC validity.
2. Verify block header integrity.
3. Verify partition root inclusion in state_root.
4. Verify Merkle proof to payload_hash.
5. Verify anti-replay constraints.

If all pass → message valid.

---

## 2.4 Atomic Cross-Shard Transaction Flow

To perform atomic transfer between partitions A and B:

Phase 1:
- Lock asset in A.
- Emit cross-partition message with proof.

Phase 2:
- Verify proof in B.
- Mint/unlock asset in B.

Completion requires:

- Deterministic commit in B
- Replay protection (see Section 6)

---

# 3. Cross-Plane Messaging (Truth ↔ Intelligence)

Cross-plane messaging is asymmetric:

- Intelligence Plane submits commitments to Truth Plane.
- Truth Plane enforces economic constraints.
- Truth Plane does NOT execute intelligence code.

---

# 4. Intelligence Artifact Commit Flow

## 4.1 Artifact Commit Structure

ArtifactCommit:

- operator_id
- job_id
- input_hash
- output_hash (RPACK hash)
- resource_hash
- receipt_signature

Truth Plane verifies:

- Operator registered
- Stake bonded
- Escrow exists
- Signature valid

Truth Plane stores:

- output_hash
- operator_id
- block_height

It does NOT store artifact bytes.

---

## 4.2 Artifact Verification Flow

If dispute triggered:

1. Retrieve anchored output_hash.
2. Retrieve artifact bytes off-plane.
3. Verify H(artifact_bytes) == output_hash.
4. Execute verification protocol (LV-SPEC-0206).
5. Submit proof of fraud (if any).

Truth Plane only validates:

- Proof structure
- Slashing eligibility
- Deterministic outcome

---

# 5. Cross-Plane Commit Invariants

Truth Plane MUST:

- Treat output_hash as opaque.
- Not parse RPACK structure.
- Not require GPU validation.
- Anchor artifact deterministically.

Intelligence Plane MUST:

- Include input_hash and output_hash in receipt.
- Use canonical encoding.
- Respect domain separation.

---

# 6. Anti-Replay Protection

Replay protection mechanisms apply to both cross-partition and cross-plane flows.

---

## 6.1 Cross-Partition Replay Protection

Each CrossPartitionMessage MUST include:

- Unique message_id
- Source height
- Source partition
- Nonce

Target partition MUST maintain:

ConsumedMessageRegistry

Message rejected if:

- message_id already consumed
- Source height older than pruning window
- Merkle proof invalid

---

## 6.2 Cross-Plane Replay Protection

Each ReceiptAnchor MUST include:

- job_id
- unique execution_nonce

Truth Plane MUST reject:

- Duplicate receipt for same (job_id, operator_id)
- Receipt referencing closed job
- Receipt with reused execution_nonce

---

# 7. Ordering Constraints

## 7.1 Truth Plane Ordering

Truth Plane guarantees total order of:

- Transactions
- Receipt anchors
- Cross-partition commits

No partial ordering ambiguity.

---

## 7.2 Intelligence Plane Ordering

Intelligence Plane is asynchronous.

Ordering only required when:

- Committing to Truth Plane
- Resolving disputes
- Updating escrow state

Cross-plane ordering constraint:

Truth Plane block height defines authoritative order.

---

# 8. Cross-Partition Finality

A cross-partition message is final when:

- Source block finalized
- QC validated
- Proof verified in target
- Target block committed

Rollback only possible if ≥1/3 validators Byzantine.

---

# 9. Message Serialization

All cross-boundary messages MUST:

- Use canonical serialization
- Include version identifier
- Include domain-separated hash
- Use fixed field ordering

Serialization MUST be stable across Rust versions.

---

# 10. Partition Addition and Removal

When adding partition:

- Assign partition_id
- Initialize Merkle root
- Update global state_root structure

When removing partition:

- Commit final partition root
- Preserve proof access
- Prevent further message emission

Removal MUST be epoch-bound.

---

# 11. Light Client Verification

Light client verifying cross-partition message MUST require:

- Block header
- QC
- Partition inclusion proof
- Merkle proof
- Hash function

No full state required.

---

# 12. Security Invariants

1. No cross-shard message valid without QC.
2. No artifact accepted without receipt anchor.
3. No replayable cross-partition message.
4. No GPU required for verification.
5. Cross-plane commits are hash-only.
6. Partition removal must preserve proof access.

---

# 13. Failure Handling

If proof invalid:

- Reject message.
- Do not alter state.
- Optionally record misbehavior.

If target partition unavailable:

- Message pending until target active.

If Intelligence artifact unavailable:

- Dispute may trigger slashing.

---

# 14. Compliance Requirements

To claim LV-SPEC-0107 compliance:

Implementation MUST:

- Support Merkle-based cross-shard proof validation
- Implement consumed message registry
- Anchor artifact hashes deterministically
- Enforce anti-replay rules
- Maintain deterministic ordering
- Support partition lifecycle transitions

---

# 15. Summary

LV-SPEC-0107 defines the boundaries between:

- Truth partitions
- Truth and Intelligence planes

It ensures:

- Safe cross-shard transfers
- Verifiable artifact commitments
- Replay-resistant messaging
- Deterministic verification
- GPU-independent enforcement

Lite-Vision maintains:

Partition safety.
Economic integrity.
Cryptographic verifiability.

Without coupling consensus to intelligence execution.

---

End of LV-SPEC-0107