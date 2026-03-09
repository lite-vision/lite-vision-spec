# LV-SPEC-0101 — BFT Consensus Protocol Specification
Version: 1.0  
Status: Draft (Engineering)  
Conformance Level: Core (Mandatory)  
Implements: LV-SPEC-0100  
Language Target: Rust  

---

# 1. Purpose

This document specifies the Byzantine Fault Tolerant (BFT) consensus protocol used by the Lite-Vision Truth Plane.

The protocol:

- Is CPU-secured
- Provides deterministic finality
- Tolerates < 1/3 Byzantine validators
- Is partially synchronous
- Is leader-based
- Produces linear block finality

Lite-Vision adopts a **HotStuff-like 3-phase BFT protocol** with deterministic finality and quorum certificates.

---

# 2. Model and Assumptions

## 2.1 System Model

Let:

- N = total validators
- f = Byzantine validators

Assumption:

f < N / 3

Network model:

- Partially synchronous
- Messages eventually delivered after Global Stabilization Time (GST)

Cryptographic assumptions:

- Signatures unforgeable
- Hash functions collision-resistant

---

# 3. Validator Set

Validator set V_t is defined by the Truth Plane state.

Each validator:

- Has public key pk_i
- Has stake weight w_i

Voting power MAY be stake-weighted.

Total voting power:

W = Σ w_i

Quorum threshold:

Q = 2/3 W

---

# 4. Consensus Overview

Lite-Vision uses a **3-phase commit protocol**:

1. Propose
2. Vote (Prepare)
3. Commit

Finality achieved once a block receives a valid quorum certificate in commit phase.

Each block contains:

- Block header
- Parent hash
- Height
- View number
- Transaction list
- Quorum certificate (for parent)

---

# 5. Data Structures

## 5.1 Block

Block B_h:

- height
- parent_hash
- view
- proposer_id
- tx_list
- parent_qc

Block hash:

H(B_h)

---

## 5.2 Vote

Vote contains:

- block_hash
- height
- view
- voter_id
- signature

---

## 5.3 Quorum Certificate (QC)

QC contains:

- block_hash
- height
- view
- aggregated signatures
- validator set reference

Validity:

Σ w_i (signers) ≥ Q

---

# 6. Protocol Phases

Each view v has a designated leader L_v.

Leader selection MAY be round-robin or stake-weighted deterministic function.

---

## 6.1 Phase 1: Propose

Leader L_v:

- Selects highest known QC
- Builds new block B
- Broadcasts Proposal(B)

Validators:

- Verify:
  - Valid parent QC
  - Correct height
  - Valid transactions
  - Deterministic state transition valid

If valid → proceed to vote.

---

## 6.2 Phase 2: Prepare (Vote)

Validator sends:

Vote(block_hash, height, view)

To leader.

Leader collects votes.

If quorum reached:

QC_prepare created.

---

## 6.3 Phase 3: Commit

Leader broadcasts QC_prepare.

Validators:

- Lock on block
- Send commit vote

When quorum commit votes collected:

QC_commit formed.

Block finalized.

Finalization rule:

If block B_h has valid QC_commit, then:

- B_h is finalized.
- All ancestors are finalized.

---

# 7. Locking and Safety Rule

Each validator maintains:

- locked_block
- locked_view

A validator MUST NOT vote for a conflicting block unless:

- New block extends locked_block
- Or has higher QC that justifies unlocking

This ensures:

No two conflicting blocks can both reach commit QC.

---

# 8. Safety Proof Sketch

Assume two conflicting blocks B and B'.

Both cannot obtain QC_commit because:

- Each requires ≥ 2/3 voting power
- Intersection of any two ≥2/3 sets is ≥1/3
- At least one honest validator would have to vote twice
- Honest validators do not violate locking rules

Thus safety holds if f < N/3.

---

# 9. Liveness Conditions

Liveness requires:

- Eventually synchronous network
- Honest leader eventually selected

Timeout mechanism:

Each validator maintains local timeout.

If no proposal received in time:

- Trigger view change
- Move to next view

Eventually an honest leader will propose.

---

# 10. View Change Protocol

When timeout occurs:

Validator sends:

NewView(view+1, highest_qc)

New leader collects:

- 2/3 NewView messages
- Highest QC among them

Leader proposes block extending highest QC.

View number strictly increases.

---

# 11. Timeout and Pacemaker

Each validator runs a pacemaker:

- Maintains current view
- Triggers timeouts
- Escalates view number

Timeout duration MAY increase exponentially under repeated failures.

---

# 12. Finality

Finality is deterministic.

Once QC_commit is formed:

- Block is final.
- Cannot be reverted unless ≥1/3 validators violate protocol.

No probabilistic confirmation model.

---

# 13. Equivocation Handling

If validator signs conflicting votes for same height/view:

- Proof of equivocation is submitted
- Slashing enforced via LV-SPEC-0102

Equivocation proof:

Two signatures over different block_hash with same (height, view).

---

# 14. Validator Rotation

Validator set changes:

- Governed by transactions
- Applied at epoch boundaries

Epoch transition:

- Final block of epoch contains next validator set
- Next block references new set

Safety preserved across epoch transitions.

---

# 15. Determinism Requirements

Consensus logic MUST:

- Use deterministic Rust code
- Use canonical serialization
- Avoid floating-point
- Avoid system clock dependency (logical clocks only)
- Use stable iteration order

Block hash MUST be stable across platforms.

---

# 16. Performance Characteristics

Message complexity per view:

O(N)

Signature aggregation reduces:

- QC size
- Broadcast bandwidth

Finality latency:

≈ 2–3 network round trips under normal conditions.

---

# 17. Fault Model Summary

Tolerates:

- Crash faults
- Byzantine faults
- Network delays (eventual synchrony)

Does not tolerate:

- ≥ 1/3 Byzantine validators
- Cryptographic break
- Persistent network partition

---

# 18. Coupling Constraints

Consensus layer MUST NOT:

- Inspect intelligence data
- Validate RPACK contents
- Execute GPU workloads

Consensus validates:

- Transaction format
- Signatures
- Deterministic state transitions
- Receipt anchoring integrity

---

# 19. Compliance Requirements

To claim LV-SPEC-0101 compliance, an implementation MUST:

- Implement 3-phase HotStuff-like protocol
- Enforce quorum threshold ≥ 2/3 voting power
- Enforce locking rule
- Implement view change
- Enforce equivocation slashing proofs
- Provide deterministic finality

---

# 20. Summary

Lite-Vision Truth Plane consensus is:

- HotStuff-like
- Leader-based
- 3-phase
- Deterministically final
- CPU-secured
- Stake-weighted (optional)

Safety: guaranteed if f < N/3  
Liveness: guaranteed under partial synchrony  

This protocol forms the foundation of Lite-Vision economic security.

---

End of LV-SPEC-0101