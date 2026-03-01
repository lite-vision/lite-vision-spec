# LV-SPEC-0100 — Truth Plane Architecture Overview
Version: 1.0  
Status: Draft (Engineering)  
Conformance Level: Core (Mandatory)  
Implementation Language: Rust  

---

# 1. Purpose

This specification defines the architecture of the Lite-Vision Truth Plane.

The Truth Plane is:

- CPU-secured
- Byzantine Fault Tolerant (BFT)
- Deterministic
- Economically authoritative
- Independent from GPU requirements

It provides:

- Identity and staking
- Validator governance
- Job escrow and settlement
- Receipt anchoring
- Artifact commitments
- Partition governance
- Slashing enforcement

The Truth Plane does not execute intelligence workloads.
It does not render pixels.
It enforces accountability.

---

# 2. Architectural Role in Lite-Vision

Lite-Vision consists of two primary planes:

Global state:

X_t = (T_t, I_t)

Where:

T_t = Truth Plane state  
I_t = Intelligence Plane state  

Truth Plane evolution:

T_{t+1} = δ(T_t, tx_t)

Intelligence Plane evolution:

I_{t+1} = 𝔉(I_t, U_t ; T_t)

The Truth Plane:

- Constrains Intelligence via economic and capability rules
- Anchors intelligence outputs cryptographically
- Enforces slashing and settlement

The Intelligence Plane cannot mutate Truth Plane state without consensus.

---

# 3. Trust Boundaries

The Truth Plane defines strict trust boundaries between roles.

## 3.1 Validators

Validators:

- Participate in BFT consensus
- Propose and vote on blocks
- Execute deterministic state transitions
- Maintain full ledger state
- Enforce slashing rules

Validators MUST:

- Run CPU-based execution only
- Execute deterministic Rust code
- Not require GPUs

Security assumption:

f < N / 3 Byzantine validators

Where:
N = total validator count  
f = malicious validators  

Validators are economically bonded via stake.

---

## 3.2 Delegators

Delegators:

- Bond stake to validators
- Share in rewards and penalties
- Influence validator selection weight

Delegators do NOT:

- Participate in consensus voting
- Execute state transitions

Delegation affects validator stake weight.

---

## 3.3 Observers (Full Nodes)

Observers:

- Replicate the ledger
- Verify state transitions
- Do not vote
- Do not require stake

Observers enhance transparency and auditability.

---

## 3.4 Intelligence Operators (External Role)

Intelligence operators:

- Exist primarily in the Intelligence Plane
- Register stake and capabilities in the Truth Plane
- Submit receipts and artifacts
- Are subject to slashing via Truth Plane

They do not participate in BFT consensus unless separately registered as validators.

---

# 4. Deterministic State Machine Scope

The Truth Plane state machine MUST be fully deterministic.

## 4.1 Determinism Requirements

- No floating-point arithmetic
- No non-seeded randomness
- No wall-clock time dependencies
- No external I/O
- Canonical serialization for hashing
- Stable ordering of iteration over maps/sets

All state transitions MUST produce identical results across validators.

---

## 4.2 State Domains

Truth Plane state T_t includes:

- Accounts
- Stakes
- Validator set
- Governance parameters
- Job escrow registry
- Receipt registry
- Reputation metrics
- Artifact commitment registry
- Partition registry

It MUST NOT include:

- Intelligence internal memory
- Model weights
- Scene data
- Pixels

---

## 4.3 State Partitioning

Truth Plane supports partitioned state domains:

T_t = ⋃ T_t^k

Partitions MAY represent:

- Account shards
- Job domains
- Capability registries
- Artifact namespaces

Partitioning MUST NOT violate global consensus safety.

Cross-partition commits MUST include cryptographic proofs.

---

# 5. Consensus Scope

The Truth Plane:

- Orders transactions
- Commits blocks
- Finalizes state transitions

It does NOT:

- Execute intelligence kernels
- Perform GPU computation
- Simulate world models
- Generate RPACK

Consensus throughput MUST remain independent of intelligence workload size.

---

# 6. Coupling Surface to Intelligence Plane

The Truth Plane interfaces with the Intelligence Plane via a minimal, well-defined surface.

This coupling is economic and cryptographic — not computational.

## 6.1 Operator Registration

Intelligence operators MUST:

- Register identity
- Declare capabilities
- Bond stake
- Declare kernel versions

Truth Plane records:

OperatorID → CapabilitySet → StakeAmount

---

## 6.2 Job Escrow

When a job is created:

- Escrow is locked in Truth Plane
- Budget is committed
- Verification policy recorded

Truth Plane does not execute the job.
It only tracks economic guarantees.

---

## 6.3 Receipt Anchoring

After intelligence execution:

Operator submits receipt containing:

- Input hash
- Kernel ID + version
- Output hash (RPACK hash)
- Resource metrics
- Signature

Truth Plane verifies:

- Signature validity
- Capability authorization
- Escrow availability

Truth Plane anchors:

H(RPACK)

It does not inspect RPACK contents.

---

## 6.4 Artifact Commitments

Artifacts produced by Intelligence Plane are content-addressed:

artifact_id = H(artifact_bytes)

Truth Plane stores:

artifact_id
producer_id
timestamp
associated_job_id

The artifact bytes reside off-plane.

---

## 6.5 Slashing and Disputes

Truth Plane enforces:

- Stake slashing
- Challenger reward distribution
- Validator penalties (if applicable)

Dispute resolution may trigger:

- Deterministic replay (if job required it)
- Verification sampling
- Governance action

Slashing is deterministic once fraud is proven.

---

# 7. Security Guarantees

The Truth Plane guarantees:

1. Deterministic finality
2. Economic accountability
3. Immutable artifact commitments
4. Enforceable slashing
5. Validator-level fault tolerance (< 1/3 Byzantine)
6. CPU-friendly decentralization

It does NOT guarantee:

- Correctness of intelligence outputs
- Model integrity
- Visual consistency
- Rendering determinism

Those are handled by other specs.

---

# 8. Hardware Constraints

Validators MUST NOT require GPUs.

Truth Plane execution cost:

O(n_tx + n_sig + n_hash)

This ensures:

- Accessibility
- Decentralization
- Stability independent of GPU market dynamics

---

# 9. Design Invariants

1. Consensus MUST remain GPU-independent.
2. Truth Plane state MUST remain deterministic.
3. Intelligence computation MUST never occur inside Truth Plane.
4. All economic enforcement MUST flow through Truth Plane.
5. Artifact commitments MUST be hash-based.
6. Partition deletion MUST preserve provability.

---

# 10. Non-Goals

The Truth Plane is not:

- A rendering service
- A GPU compute marketplace engine
- A model execution engine
- A scene validator
- A world simulator

It is:

A deterministic, CPU-secured, economically authoritative state machine.

---

# 11. Compliance Requirements

To claim conformance with LV-SPEC-0100:

An implementation MUST:

- Implement a deterministic Rust state machine
- Enforce BFT validator voting
- Support stake bonding and slashing
- Anchor receipt and artifact hashes
- Expose partition management controls
- Maintain strict hardware separation from GPU requirements

---

# 12. Summary

The Truth Plane provides:

- Finality
- Accountability
- Governance
- Economic enforcement

It is:

- CPU-secured
- Deterministic
- Partition-aware
- Independent from intelligence execution

It constrains the system.

It does not compute intelligence.

---

End of LV-SPEC-0100