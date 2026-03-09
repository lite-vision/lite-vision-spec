# LV-SPEC-0201 — Operator Node Lifecycle and Registration
Version: 1.0  
Status: Draft (Engineering)  
Conformance Level: Core (Mandatory)  
Implements: LV-SPEC-0102, LV-SPEC-0103, LV-SPEC-0107, LV-SPEC-0200  
Language Target: Rust  

---

# 1. Purpose

This specification defines:

- Operator identity binding to the Truth Plane
- Capability declarations (hardware, kernels, regions)
- Join, active, suspended, and leave semantics
- Economic penalties and lifecycle transitions

An Operator Node is an Intelligence Plane participant that:

- Provides GPU/accelerator compute
- Executes Intelligence Kernels
- Produces artifacts (RPACK)
- Submits receipts to the Truth Plane
- Is economically bonded and slashable

Operators do NOT participate in BFT consensus unless separately registered as validators.

---

# 2. Operator Identity Model

## 2.1 Operator Identity Definition

Each operator is uniquely identified by:

OperatorID = H(public_key)

Required identity fields:

- public_key (Ed25519 required)
- optional_bls_key (if used for aggregation roles)
- network_endpoint
- version
- registration_block_height

OperatorID MUST be deterministic and canonical.

---

## 2.2 Binding to Truth Plane

Operator registration is a Truth Plane transaction:

OperatorRegister {
  operator_pubkey
  bonded_stake
  capability_hash
  region_id
  metadata_hash
  signature
}

Upon successful inclusion:

Truth Plane stores:

- OperatorID
- StakeAmount
- CapabilityHash
- Status = Active
- RegistrationHeight

---

# 3. Capability Declarations

Operators MUST declare capabilities at registration.

Capabilities include:

## 3.1 Hardware Capabilities

- GPU type (string identifier)
- GPU memory (VRAM size)
- Accelerator class (optional)
- Max concurrent jobs
- Bandwidth capacity

Hardware data is declarative and not enforced by Truth Plane.

---

## 3.2 Kernel Capabilities

Operators declare:

- Supported kernel IDs
- Kernel versions
- Deterministic mode support (yes/no)
- Maximum model size
- Execution limits

KernelID MUST correspond to approved registry entry.

CapabilityHash = H(canonical_capability_struct)

Truth Plane stores only CapabilityHash.

---

## 3.3 Regional Affinity

Operators MAY declare:

- Geographic region
- Partition ID
- Latency zone

Routers MAY use region metadata for job selection.

Truth Plane does not validate geographic truthfulness.

---

# 4. Stake Bonding

## 4.1 Minimum Stake

Operator MUST bond stake S_op ≥ S_min_operator.

Stake remains locked while:

Status ∈ {Active, Suspended}

Stake is slashable during bonding and unbonding period.

---

## 4.2 Stake Update

Operator MAY increase stake at any time.

Stake decrease requires:

- Unbond request
- Unbonding period T_unbond_operator
- No pending disputes

---

# 5. Operator Lifecycle States

Operator state machine:

Registered → Active → Suspended → Unbonding → Exited

---

## 5.1 Registered

- Registration transaction accepted
- Stake locked
- Awaiting activation epoch

---

## 5.2 Active

- Eligible for routing
- Eligible for job execution
- Receipt submissions allowed
- Subject to verification and slashing

---

## 5.3 Suspended

Triggered by:

- Slashing event
- Repeated verification failure
- Governance suspension
- Failure to meet minimum uptime

Suspended operators:

- Cannot receive new jobs
- Remain slashable
- May appeal (if governance allows)

---

## 5.4 Unbonding

Operator requests exit:

UnbondRequest {
  operator_id
  signature
}

Unbonding period begins:

- Cannot accept new jobs
- Stake remains slashable
- Existing disputes must resolve

After T_unbond_operator:

Status → Exited

---

## 5.5 Exited

- Stake released
- Operator removed from routing
- Identity remains in history for audit

---

# 6. Join Semantics

## 6.1 Registration Flow

1. Submit OperatorRegister transaction.
2. Truth Plane validates:
   - Signature
   - Stake >= S_min_operator
   - No duplicate OperatorID
3. Inclusion in block.
4. Activation at next epoch boundary.

Operators MUST NOT execute jobs before activation.

---

# 7. Leave Semantics

Operator MAY voluntarily exit:

1. Submit UnbondRequest.
2. Enter Unbonding state.
3. After T_unbond_operator and no active disputes:
   - Stake released.
   - Status set to Exited.

Operator MUST remain slashable during unbonding.

---

# 8. Suspension and Penalties

## 8.1 Automatic Suspension

Triggered if:

- Fraud proven
- Multiple invalid receipts
- Repeated verification failure
- Failing deterministic mode when required

Suspension duration defined by governance.

---

## 8.2 Slashing Penalties

Operator slashed if:

- Fraud proven in challenge protocol
- Double-submitted receipt
- Forged resource metrics
- Capability mismatch

Slash amount:

S_op -= penalty_amount

Penalty MUST be deterministic.

---

# 9. Capability Updates

Operator MAY update capabilities:

OperatorUpdate {
  new_capability_hash
  signature
}

Update takes effect at next epoch boundary.

Update MUST NOT:

- Invalidate historical receipts
- Alter past job commitments

---

# 10. Verification Interaction

Operators MUST:

- Accept potential re-execution by verifiers
- Provide reproducible execution context (if deterministic mode)
- Maintain logs required for dispute resolution

Failure to cooperate MAY result in slashing.

---

# 11. Anti-Replay Protections

Each operator execution MUST include:

- execution_nonce
- job_id
- block_height reference

Truth Plane MUST reject:

- Duplicate receipt for same (job_id, operator_id)
- Receipt referencing inactive operator
- Receipt after unbond period

---

# 12. Economic Constraints

Operator security condition:

ExpectedFraudGain < StakeAtRisk × VerificationProbability

Governance MUST maintain S_min_operator such that:

Network fraud economically irrational.

---

# 13. Hardware Independence Rule

Operator hardware MAY include GPU.

Truth Plane MUST NOT:

- Require GPU hardware for operator registration.
- Inspect GPU details.
- Depend on GPU availability.

GPU enforcement is economic and off-plane.

---

# 14. Rust Implementation Requirements

Implementation MUST:

- Use canonical serialization for OperatorRegister.
- Use deterministic stake accounting.
- Enforce lifecycle state machine strictly.
- Avoid floating-point arithmetic in stake logic.
- Maintain ordered state structures.

---

# 15. Compliance Requirements

To claim LV-SPEC-0201 compliance:

Implementation MUST:

- Support operator registration.
- Enforce minimum stake.
- Support lifecycle transitions.
- Enforce suspension and slashing.
- Support capability declaration.
- Support unbonding period.
- Prevent replayed receipts.

---

# 16. Design Invariants

1. Operators must be economically bonded.
2. Operators must be slashable.
3. Lifecycle transitions must be deterministic.
4. Operators cannot mutate Truth Plane state.
5. GPU hardware cannot influence consensus safety.
6. Unbonding must preserve fraud accountability.

---

# 17. Summary

LV-SPEC-0201 defines the Operator lifecycle:

- Identity binding to Truth Plane
- Capability declaration
- Stake bonding
- Join/leave semantics
- Suspension and slashing enforcement

Operators power the Intelligence Plane.

The Truth Plane enforces accountability.

Separation remains intact:

CPU-secured consensus.
GPU-powered execution.
Deterministic economic enforcement.

---

End of LV-SPEC-0201