# LV-SPEC-0204 — Routing and Sparse Activation (Thalamus)
Version: 1.0  
Status: Draft (Engineering)  
Conformance Level: Core (Mandatory)  
Implements: LV-SPEC-0200, LV-SPEC-0201, LV-SPEC-0203  
Language Target: Rust  

---

# 1. Purpose

This specification defines the Routing and Sparse Activation subsystem of the Intelligence Plane, code-named **Thalamus**.

It formalizes:

- k-of-N activation policy
- Operator scoring signals
- Deterministic routing mode (seeded)
- Learned routing mode (adaptive)

Routing determines which operators execute a given job.

Routing MUST:

- Respect capability constraints
- Respect budget and QoS constraints
- Remain independent of Truth Plane consensus
- Preserve optional determinism when required

---

# 2. Conceptual Model

Given:

- Total operators: N
- Eligible operators for job J: E ⊆ N
- Activation size: k

Thalamus selects:

A ⊆ E  
|A| = k

Where typically:

k << |E|

This is sparse activation.

---

# 3. Eligibility Filter

Before scoring, operators must pass eligibility filter:

Eligibility conditions:

- Operator status == Active
- KernelID ∈ declared_capabilities
- Stake ≥ minimum threshold
- Region compatible (if required)
- Resource capacity sufficient
- Not currently suspended

Filtered set:

E = {operators satisfying eligibility}

Routing MUST NOT consider ineligible operators.

---

# 4. k-of-N Activation Policy

## 4.1 k Determination

k may be derived from:

- JobTicket.redundancy_factor
- QoS class
- Verification policy
- Governance limits

Examples:

Low-Latency → k = 1  
Balanced → k = 2  
High-Assurance → k = 3–5  
Deterministic-Critical → k ≥ 2 for redundancy

k MUST satisfy:

1 ≤ k ≤ |E|

---

## 4.2 Activation Constraints

Selected operators MUST:

- Not exceed job budget
- Not exceed per-operator concurrency cap
- Not violate geographic or partition policy

---

# 5. Scoring Model

Each eligible operator i receives a score:

Score_i = f(Sim_i, Rep_i, Lat_i, Price_i, Load_i, Region_i)

---

## 5.1 Similarity Score (Sim_i)

Measures how well operator matches job characteristics:

- Kernel specialization
- Historical job similarity
- Model size compatibility
- Memory proximity

Range: [0, 1]

---

## 5.2 Reputation Score (Rep_i)

Derived from:

- Historical success rate
- Fraud incidents
- Verification pass rate
- Uptime

Rep_i ∈ [0, 1]

Reputation MUST be bounded and decay over time.

---

## 5.3 Latency Score (Lat_i)

Estimated response time:

Lower latency → higher score

May include:

- Network RTT
- Region proximity
- Current queue depth

---

## 5.4 Price Score (Price_i)

Derived from:

- Operator declared price
- Historical effective cost
- Budget compatibility

Lower cost → higher score.

---

## 5.5 Load Score (Load_i)

Reflects current resource usage:

Lower load → higher score.

Prevents overload concentration.

---

## 5.6 Composite Score

Composite scoring function MUST be deterministic when required:

Score_i = w1·Sim_i + w2·Rep_i + w3·Lat_i + w4·Price_i + w5·Load_i

Weights defined per QoS class.

Weights MUST be bounded and governance-adjustable.

---

# 6. Selection Algorithm

## 6.1 Deterministic Routing Mode

Used when:

- Job execution_mode == Deterministic
- QoS == Deterministic-Critical

Selection procedure:

1. Compute eligible set E.
2. Compute Score_i deterministically.
3. Sort E by (Score_i, OperatorID).
4. Use seeded tie-break:

seed = H(job_id || block_height_reference)

5. Select top k using deterministic shuffle seeded by seed.

Result MUST be reproducible across verifiers.

---

## 6.2 Learned Routing Mode

Default for soft jobs.

Selection procedure MAY:

- Use reinforcement learning
- Update weights dynamically
- Adapt to network conditions
- Use historical outcome signals

Constraints:

- Must respect eligibility filter
- Must respect budget and QoS
- Must not violate k-of-N rule

Learned routing MUST NOT affect Truth Plane consensus.

---

# 7. Deterministic Mode Requirements

If deterministic routing required:

- All scoring inputs must be canonical
- No wall-clock time
- No local randomness
- All weights fixed at job creation
- All data hashed or anchored

Routing decision MUST be reproducible.

---

# 8. Redundant Execution Policy

For k > 1:

Options:

1. Parallel redundant execution
2. Sequential fallback
3. Majority selection
4. First-valid result wins

Policy defined in JobTicket.

If outputs differ:

- Trigger dispute
- Escrow frozen
- Verification escalated

---

# 9. Anti-Centralization Controls

Routing SHOULD avoid excessive concentration.

Possible constraints:

- Maximum share of jobs per operator
- Regional balancing
- Randomized selection within top percentile
- Decaying reputation advantage

These controls MUST NOT break deterministic mode.

---

# 10. Partition Awareness

If Intelligence Plane partitioned:

Routing MUST:

- Prefer operators within same partition (if latency-critical)
- Allow cross-partition routing if required
- Respect partition resource caps

Partition transitions must not invalidate routing reproducibility for deterministic jobs.

---

# 11. Failure Handling

If selected operator fails:

- Retry using next-ranked operator
- Increment retry counter
- Preserve job_id
- Maintain deterministic seed (if deterministic mode)

Fallback selection MUST respect same constraints as initial routing.

---

# 12. Security Model

Routing security depends on:

- Reputation weighting
- Verification probability
- Stake bonding
- Slashing enforcement

Routing MUST assume:

Some operators may behave maliciously.

Economic + verification model must compensate.

---

# 13. Resource Fairness

Routing SHOULD:

- Avoid starvation of new operators
- Avoid permanent reputation lock-in
- Allow exploration

Exploration in deterministic mode MUST be seeded and reproducible.

---

# 14. Rust Implementation Requirements

Implementation MUST:

- Use deterministic sorting (BTree structures)
- Use canonical score calculation
- Avoid floating-point nondeterminism (use fixed-point arithmetic)
- Provide pluggable scoring module
- Support both deterministic and learned modes

Floating-point MAY be used in learned mode only if:

- Not required for deterministic replay
- Not used in dispute-sensitive jobs

---

# 15. Compliance Requirements

To claim LV-SPEC-0204 compliance:

Implementation MUST:

- Implement eligibility filtering
- Support k-of-N activation
- Support scoring model
- Support deterministic routing mode
- Support learned routing mode
- Respect budget and QoS constraints
- Ensure reproducibility in deterministic mode

---

# 16. Design Invariants

1. k-of-N activation must always hold.
2. Deterministic routing must be replayable.
3. Learned routing must not affect consensus safety.
4. Routing must respect capability constraints.
5. Routing must never bypass stake bonding.
6. GPU routing must remain off-plane from consensus.

---

# 17. Summary

LV-SPEC-0204 defines Thalamus:

- Sparse activation engine
- Operator scoring and selection
- Deterministic and adaptive routing modes
- Redundancy and fallback logic

It enables:

Scalability through sparse compute.
Security through stake + verification.
Flexibility through adaptive routing.
Reproducibility when required.

Consensus remains CPU-secured.
Execution remains GPU-powered.
Routing remains economically bounded.

---

End of LV-SPEC-0204