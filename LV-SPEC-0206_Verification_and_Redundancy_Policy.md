# LV-SPEC-0206 — Verification and Redundancy Policy
Version: 1.0  
Status: Draft (Engineering)  
Conformance Level: Core (Mandatory)  
Implements: LV-SPEC-0102, LV-SPEC-0107, LV-SPEC-0203, LV-SPEC-0205  
Language Target: Rust  

---

# 1. Purpose

This specification defines:

- Primary and secondary execution patterns
- Verifier sampling strategies
- Redundancy schedules
- Disagreement detection
- Escalation and dispute triggers

Verification in Lite-Vision is:

- Probabilistic by default
- Deterministic when required
- Economically enforced
- Asynchronous
- GPU-executed off-consensus

Truth Plane enforces slashing but does not execute verification.

---

# 2. Verification Model Overview

Given:

- Operator O executes job J
- Receipt R submitted
- output_hash anchored

Verification determines whether:

H(recomputed_output) == R.output_hash

If mismatch → fraud proof possible.

Verification modes depend on:

- Job execution_mode
- QoS class
- verification_policy
- redundancy_factor

---

# 3. Execution Patterns

## 3.1 Primary-Only Execution (k=1)

- Single operator executes
- Verification may occur probabilistically
- Suitable for low-value jobs

Fraud deterrence relies on:

p × slash_penalty > fraud_gain

---

## 3.2 Primary + Secondary Redundant Execution (k≥2)

- Multiple operators execute same job
- Outputs compared
- Used for high-assurance jobs

Patterns:

1. Parallel redundant
2. Sequential fallback
3. Majority consensus
4. Deterministic cross-check

Redundancy schedule defined in JobTicket.

---

## 3.3 Deterministic Replay Mode

If execution_mode == Deterministic:

Verification MUST be capable of:

- Full re-execution
- Seed reuse
- Bitwise output comparison

Replay may occur:

- Immediately
- After random delay
- Only upon challenge

---

# 4. Verification Sampling

## 4.1 Sampling Probability

VerificationProbability p ∈ [0, 1]

Defined by:

- QoS class
- Budget
- Governance parameters
- Operator reputation
- Job value

Higher-value jobs → higher p.

---

## 4.2 Adaptive Sampling

Sampling MAY increase if:

- Operator reputation decreases
- Network fraud detected
- High dispute rate observed

Adaptive changes MUST NOT violate determinism for deterministic jobs.

---

# 5. Verifier Selection

Verifier selection MUST follow:

- Eligibility filtering
- Capability match
- No same-operator self-verification (unless redundancy rule allows)
- Random or score-based selection

Deterministic mode MUST use seeded selection:

seed = H(job_id || block_height_reference)

Verifier selection must be reproducible.

---

# 6. Redundancy Schedules

RedundancyFactor r defined in JobTicket.

Possible values:

r = 1 → no redundancy  
r = 2 → dual execution  
r ≥ 3 → majority comparison  

Redundancy MAY be:

- Fixed per job
- Dynamic based on job class
- Escalated upon suspicion

Escalation MUST be deterministic once triggered.

---

# 7. Disagreement Handling

## 7.1 Output Mismatch

If two operators produce:

output_hash_A ≠ output_hash_B

Then:

- Job enters DisputePending
- Escrow frozen
- Verification escalated

---

## 7.2 Escalation Ladder

Level 0: No verification  
Level 1: Random sampling  
Level 2: Deterministic replay  
Level 3: Multi-verifier replay  
Level 4: Formal dispute

Escalation level determined by:

- QoS
- Fraud suspicion
- Governance rule
- Verification policy

---

# 8. Fraud Proof Structure

FraudProof {
  receipt_hash,
  recomputed_output_hash,
  original_output_hash,
  kernel_id,
  input_hash,
  deterministic_seed (if applicable),
  evidence_bundle,
  challenger_signature
}

Truth Plane verifies:

- Receipt exists
- Challenger stake posted
- Evidence well-formed

Verification of correctness occurs off-plane.

If fraud proven:

Operator slashed per LV-SPEC-0102.

---

# 9. False Challenge Handling

If fraud proof invalid:

- Challenger bond slashed
- Operator cleared
- Reputation of challenger reduced

Prevents griefing.

---

# 10. Verification Windows

Each job defines:

verification_window_blocks

Fraud proof must be submitted before window closes.

After window:

- Job settled
- Escrow released
- Fraud claims invalid

Deterministic jobs MAY have longer windows.

---

# 11. Economic Model

Fraud deterrence condition:

p × L > G

Where:

p = verification probability  
L = slash penalty  
G = fraud gain  

Governance MUST maintain L and p such that inequality holds.

---

# 12. Majority Resolution (r ≥ 3)

If r ≥ 3:

Outputs compared.

If majority agree:

- Majority output accepted.
- Minority operators penalized (optional).

If no majority:

- Escalate to deterministic replay.
- Freeze escrow.

Majority rule MUST be explicitly declared in JobTicket.

---

# 13. Partial Verification

For soft jobs:

Verifier MAY:

- Recompute subset of output
- Validate resource metrics
- Validate schema correctness

Partial verification reduces cost but lowers fraud detection certainty.

---

# 14. Cross-Partition Considerations

If job spans partitions:

Verification MUST:

- Use cross-partition proof format
- Validate artifact hash inclusion
- Respect cross-plane anti-replay rules

Verification MUST NOT violate partition determinism.

---

# 15. Deterministic Mode Invariants

If execution_mode == Deterministic:

- Output MUST be reproducible.
- Resource caps MUST match.
- Verifier MUST use identical kernel version.
- Floating-point nondeterminism forbidden.

Mismatch → automatic fraud.

---

# 16. Reputation Interaction

Verification results update reputation:

- Successful verified jobs → reputation increase.
- Fraud detected → reputation drop.
- False challenge → challenger reputation drop.

Reputation MUST be bounded and decaying.

---

# 17. Retry and Verification Interaction

If job retried:

- Only final successful receipt eligible for settlement.
- Prior failed attempts MAY be audited.
- Fraud window applies per attempt.

Retry MUST not reset fraud accountability.

---

# 18. Rust Implementation Requirements

Implementation MUST:

- Support verification selection logic
- Support deterministic replay mode
- Support fraud proof structure
- Enforce verification window
- Prevent self-verification (unless allowed)
- Avoid floating-point nondeterminism in deterministic mode
- Maintain reproducible seeded randomness

---

# 19. Compliance Requirements

To claim LV-SPEC-0206 compliance:

Implementation MUST:

- Support primary-only execution
- Support redundant execution
- Support probabilistic verification
- Support deterministic replay
- Support fraud proof submission
- Support slashing integration
- Enforce verification windows
- Handle disagreement escalation

---

# 20. Design Invariants

1. Verification must be economically meaningful.
2. Deterministic jobs must be replayable.
3. Fraud must be slashable.
4. False challenges must be penalized.
5. Truth Plane must never execute GPU validation.
6. Verification must scale with sparse activation.

---

# 21. Summary

LV-SPEC-0206 defines:

- Redundant execution patterns
- Verification sampling strategies
- Escalation ladder
- Fraud proof mechanics
- Economic deterrence rules

It enables:

Scalable asynchronous execution.
GPU-powered verification.
CPU-secured settlement.
Economic enforcement without full replication.

Lite-Vision achieves:

Security through incentives.
Scalability through sparsity.
Determinism where required.
Flexibility where allowed.

---

End of LV-SPEC-0206