# LV-SPEC-0207 — Challenge / Dispute Protocol (Truth-Enforced)
Version: 1.0  
Status: Draft (Engineering)  
Conformance Level: Core (Mandatory)  
Implements: LV-SPEC-0102, LV-SPEC-0103, LV-SPEC-0107, LV-SPEC-0205, LV-SPEC-0206  
Language Target: Rust  

---

# 1. Purpose

This specification defines the Challenge / Dispute Protocol of Lite-Vision.

It formalizes:

- Dispute window mechanics
- Challenger bonding requirements
- Adjudication workflow
- Slashing distribution rules
- Appeal constraints

Disputes are enforced by the Truth Plane.

The Truth Plane:

- Anchors receipts
- Freezes escrow
- Enforces slashing
- Does NOT execute GPU workloads

---

# 2. Dispute Overview

A dispute may be initiated when:

- A receipt output_hash is suspected fraudulent
- Deterministic replay mismatch detected
- Redundant execution disagreement occurs
- Resource claims are materially false

Dispute affects:

- Job settlement
- Escrow release
- Operator stake

---

# 3. Dispute Window

Each JobTicket defines:

verification_window_blocks

Dispute MUST be initiated before:

current_block_height > receipt_block_height + verification_window_blocks

After window closes:

- Job settled
- Fraud claim invalid
- Escrow finalized

Deterministic-critical jobs MAY have extended windows.

---

# 4. Dispute Initiation

## 4.1 Challenger Requirements

Challenger MUST:

- Be registered account
- Post ChallengerBond ≥ B_min
- Submit FraudProof structure

FraudProof MUST include:

- receipt_hash
- operator_id
- recomputed_output_hash
- original_output_hash
- kernel_id
- input_hash
- deterministic_seed (if applicable)
- evidence_bundle
- challenger_signature

---

## 4.2 Dispute Transaction

Truth Plane transaction:

DisputeInitiate {
  job_id,
  receipt_hash,
  fraud_proof_hash,
  challenger_bond,
  signature
}

On inclusion:

- Job enters DisputePending
- Escrow frozen
- Operator stake partially frozen

---

# 5. Adjudication Workflow

Dispute resolution follows deterministic steps.

---

## 5.1 Phase 1: Validation

Truth Plane verifies:

- Receipt exists
- Challenger bond posted
- FraudProof well-formed
- Dispute within window
- Execution nonce valid

If invalid → reject dispute.

---

## 5.2 Phase 2: Verification Escalation

Verification level escalates to:

Level 3 or 4 (see LV-SPEC-0206)

Steps:

1. Deterministic replay by independent verifier(s)
2. Compare recomputed output_hash
3. Collect verification signatures

Truth Plane does not compute replay.

Verification result submitted as:

VerificationResult {
  job_id,
  receipt_hash,
  recomputed_output_hash,
  verification_signatures
}

---

## 5.3 Phase 3: Resolution

Case A — Fraud Confirmed:

recomputed_output_hash ≠ original_output_hash

→ Operator guilty.

Case B — Fraud Rejected:

recomputed_output_hash == original_output_hash

→ Challenger guilty.

Case C — Inconclusive:

- Escalate redundancy
- Governance review (rare)
- Refund partial bond

Resolution MUST be deterministic once evidence collected.

---

# 6. Slashing Rules

## 6.1 Operator Slashing

If fraud confirmed:

Operator stake reduced by:

S_slash = max(
  governance_min_slash,
  α × job_budget,
  β × verification_cost
)

Slashed amount distributed:

- γ to Challenger
- δ to Verifiers
- ε burned or to treasury

γ + δ + ε = 1

Distribution ratios governance-controlled.

Operator MAY be:

- Suspended
- Jailed
- Permanently banned (repeat offenses)

---

## 6.2 Challenger Slashing

If fraud rejected:

ChallengerBond forfeited:

- Portion to operator (compensation)
- Portion to verifiers
- Portion burned

Prevents spam disputes.

---

# 7. Escrow Handling

During dispute:

- Escrow frozen
- No operator payout
- No client refund

After resolution:

If operator guilty:

- Client refunded
- Operator slashed
- Verification costs deducted

If challenger guilty:

- Operator paid
- Challenger bond slashed

If inconclusive:

- Escrow partially refunded
- Governance-defined fallback

---

# 8. Appeals

Appeals are limited.

Appeal may occur if:

- Procedural error
- Evidence corruption
- Governance-triggered review

Appeal window:

appeal_window_blocks

Appeal MUST include:

- New evidence
- Appeal bond

Appeal does NOT:

- Revert finalized blocks
- Rewrite state history

Appeals may only affect:

- Slashing distribution
- Suspension duration

---

# 9. Time Constraints

Dispute timing constraints:

- Must initiate within verification window
- Must resolve within adjudication window
- Appeal must occur within appeal window

All windows measured in block height.

---

# 10. Multiple Disputes

If multiple challengers dispute same receipt:

- First valid dispute locks job
- Additional challengers MAY join as co-challengers
- Duplicate disputes rejected

---

# 11. Cross-Partition Disputes

If job executed in different partition:

Dispute MUST include:

- Cross-partition proof
- Receipt inclusion proof
- Artifact inclusion proof

Truth Plane verifies inclusion proofs before adjudication.

---

# 12. Evidence Integrity

EvidenceBundle MAY include:

- Kernel binary hash
- Input data snapshot
- Deterministic seed
- Execution logs
- Resource metrics

Truth Plane verifies structure and hash integrity.

Content validation occurs off-plane.

---

# 13. Economic Safety Condition

For dispute system to be secure:

ChallengerBond > cost_of_false_challenge

OperatorStake > fraud_gain

VerificationProbability × OperatorStake > fraud_gain

Governance MUST monitor economic margins.

---

# 14. Abuse Mitigation

To prevent griefing:

- Rate limit disputes per challenger
- Reputation penalty for failed disputes
- Escalating bond requirement for repeat challengers
- Temporary suspension for malicious dispute spam

---

# 15. Rust Implementation Requirements

Implementation MUST:

- Canonically serialize FraudProof
- Enforce deterministic dispute windows
- Freeze escrow during dispute
- Prevent double-resolution
- Ensure slashing arithmetic uses fixed-width integers
- Maintain ordered dispute registry

No floating-point arithmetic allowed in slashing logic.

---

# 16. Compliance Requirements

To claim LV-SPEC-0207 compliance:

Implementation MUST:

- Support dispute initiation
- Enforce challenger bond
- Support adjudication workflow
- Enforce slashing distribution
- Enforce verification window
- Support appeal constraints
- Preserve deterministic finality

---

# 17. Design Invariants

1. Fraud must be slashable.
2. False challenges must be penalized.
3. Disputes must be time-bound.
4. Truth Plane must not execute GPU workloads.
5. Escrow must freeze during dispute.
6. Historical state must remain immutable.

---

# 18. Summary

LV-SPEC-0207 defines the Truth-enforced dispute mechanism:

- Time-bound challenge windows
- Challenger bonding
- Deterministic adjudication
- Slashing distribution
- Limited appeals

It ensures:

Economic accountability.
Fraud deterrence.
Finality preservation.
GPU-independent enforcement.

Lite-Vision maintains:

CPU-secured settlement.
GPU-powered cognition.
Economically enforced integrity.

---

End of LV-SPEC-0207