# LV-SPEC-0203 — Task / Job Model and Execution Semantics
Version: 1.0  
Status: Draft (Engineering)  
Conformance Level: Core (Mandatory)  
Implements: LV-SPEC-0103, LV-SPEC-0107, LV-SPEC-0200, LV-SPEC-0202  
Language Target: Rust  

---

# 1. Purpose

This specification defines:

- The Job Ticket format
- Budgeting and escrow rules
- Deadlines and QoS policies
- Deterministic vs soft execution modes
- Retry semantics
- Partial completion rules
- Cancellation behavior

Jobs are the fundamental execution unit of the Intelligence Plane.

Truth Plane governs job escrow and settlement.
Intelligence Plane executes jobs asynchronously.

---

# 2. Job Overview

A Job represents:

- A kernel execution request
- Bound input artifacts
- Defined resource budget
- Explicit execution mode
- Defined verification policy

Each job has a unique:

JobID = H(job_ticket)

---

# 3. Job Ticket Format

JobTicket structure:

- job_id
- client_id
- kernel_id
- input_hash
- execution_mode
- budget
- deadline
- qos_class
- verification_policy
- max_retries
- partial_allowed (bool)
- cancellation_policy
- creation_block_height
- signature

All fields MUST be canonically serialized.

---

# 4. Budget Model

## 4.1 Budget Fields

Budget includes:

- max_total_fee
- max_gpu_cycles
- max_cpu_cycles
- max_memory_bytes
- max_output_size
- verifier_reserve

Budget MUST be escrowed in Truth Plane.

Escrow locked at job creation.

---

## 4.2 Fee Accounting

Upon successful completion:

Operator receives:

operator_fee = execution_cost

Unused budget refunded to client.

If fraud detected:

Operator stake may be slashed.
Client escrow partially refunded depending on policy.

---

# 5. Deadline Semantics

JobTicket.deadline defines:

- Absolute block height OR
- Relative block offset

If job not completed before deadline:

Job enters Expired state.

Expired job:

- Cannot be settled normally
- Escrow partially refunded (policy-defined)
- Operator may be penalized if accepted but not executed

---

# 6. QoS Classes

QoS defines routing priority and verification level.

Defined QoS classes:

1. Low-Latency
2. Balanced
3. High-Assurance
4. Deterministic-Critical

QoS influences:

- Operator selection
- Verification probability
- Redundant execution count
- Routing preference

QoS does NOT alter Truth Plane consensus behavior.

---

# 7. Execution Modes

## 7.1 Soft Mode (Default)

- Non-deterministic allowed
- Partial verification
- Faster execution
- Suitable for creative or probabilistic outputs

Output reproducibility not guaranteed.

Verification via sampling.

---

## 7.2 Deterministic Mode

- Deterministic seed required
- No non-seeded randomness
- Replayable
- Suitable for dispute-sensitive jobs

Output MUST be identical under replay.

Receipt MUST include:

- deterministic_seed
- kernel_version

---

# 8. Job Lifecycle

Job state machine:

Created → Assigned → Executing → Completed → Settled
                ↘ Failed
                ↘ Cancelled
                ↘ Expired

---

## 8.1 Created

- Escrow locked
- JobTicket anchored
- Eligible for routing

---

## 8.2 Assigned

- Router selects operator(s)
- Operator accepts job
- Execution nonce generated

---

## 8.3 Executing

- Kernel execution underway
- Resource metering active

---

## 8.4 Completed

- Artifact produced
- Receipt submitted
- output_hash anchored

---

## 8.5 Settled

- Verification window closed
- Funds distributed
- Job finalized

---

# 9. Retry Semantics

JobTicket.max_retries defines maximum retry attempts.

Retry conditions:

- Operator failure
- Resource cap exceeded
- Deterministic mismatch
- Soft execution error

Retry rules:

1. Retry increments retry_counter.
2. Retry MUST use same job_id.
3. Execution_nonce MUST differ.
4. Deterministic mode must reuse same seed.

Retry MUST NOT exceed max_retries.

If exceeded → job marked Failed.

---

# 10. Partial Completion

If partial_allowed == true:

Operator MAY:

- Submit intermediate artifact
- Submit partial receipt

Partial receipt MUST include:

- completion_percentage
- partial_output_hash

Truth Plane may anchor partial state.

Settlement rules:

- Payment proportional to completion
- Remaining budget may fund continuation

Partial outputs MUST be clearly marked non-final.

---

# 11. Cancellation Semantics

Cancellation may occur via:

1. Client request (before execution starts)
2. Governance intervention
3. Budget exhaustion
4. Emergency halt

If cancellation before assignment:

- Escrow refunded minus minimal fee.

If cancellation during execution:

- Operator compensated for consumed resources.
- Remaining escrow refunded.

If cancellation after completion:

- Only valid if verification fraud proven.

Cancellation MUST NOT violate anchored receipts.

---

# 12. Failure Conditions

Job may fail due to:

- Resource limit exceeded
- Kernel panic
- Sandbox violation
- Invalid input
- Operator offline
- Verification failure

Failure handling:

- Retry if allowed
- Otherwise mark Failed
- Refund remaining escrow

Failure MUST be deterministic when re-evaluated.

---

# 13. Verification Policy

JobTicket.verification_policy includes:

- verification_probability
- redundancy_factor
- deterministic_required (bool)
- challenger_reward_fraction

Verification MAY include:

- Full re-execution
- Partial sampling
- Redundant operator execution
- Random audit

Truth Plane enforces slashing if fraud proven.

---

# 14. Multi-Operator Execution

High-assurance jobs MAY:

- Require redundant execution
- Compare multiple outputs
- Select majority output

If outputs mismatch:

- Trigger dispute
- Escrow frozen
- Verification escalated

Redundancy factor defined in JobTicket.

---

# 15. Ordering Constraints

Execution is asynchronous.

Ordering only required when:

- Anchoring receipt
- Settling escrow
- Resolving dispute

Truth Plane block height defines canonical ordering for settlement.

---

# 16. Resource Enforcement

Execution MUST respect:

- CPU cap
- GPU cap
- Memory cap
- Output size cap

If exceeded:

- Abort execution
- Return failure receipt
- No settlement without retry

Resource usage MUST be included in receipt.

---

# 17. Replay Semantics

For deterministic jobs:

Given:

- KernelID
- Input
- Seed
- State
- ExecutionContext

Re-execution MUST produce identical:

output_hash
resource_hash

If mismatch → fraud.

---

# 18. Rust Implementation Requirements

Implementation MUST:

- Use canonical JobTicket serialization
- Enforce lifecycle state transitions
- Track retry counter deterministically
- Enforce budget accounting with fixed-width integers
- Prevent nonce reuse
- Ensure consistent deadline interpretation

---

# 19. Compliance Requirements

To claim LV-SPEC-0203 compliance:

Implementation MUST:

- Support JobTicket format
- Enforce escrow budgeting
- Support deterministic and soft modes
- Support retry logic
- Support cancellation
- Support partial completion
- Enforce deadline expiration
- Produce receipt anchors

---

# 20. Design Invariants

1. JobID is immutable.
2. Budget cannot be exceeded.
3. Deterministic mode must be replayable.
4. Retries must not alter job identity.
5. Partial completion must not masquerade as final.
6. Cancellation must not rewrite anchored history.

---

# 21. Summary

LV-SPEC-0203 defines the formal execution contract between:

Clients.
Operators.
Verifiers.
Truth Plane.

It ensures:

Budget accountability.
Deterministic replay when required.
Asynchronous scalability.
Clear lifecycle semantics.

Execution remains GPU-powered.
Settlement remains CPU-secured.

---

End of LV-SPEC-0203