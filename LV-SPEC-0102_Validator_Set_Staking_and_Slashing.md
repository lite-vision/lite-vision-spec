# LV-SPEC-0102 — Validator Set, Staking, and Slashing
Version: 1.0  
Status: Draft (Engineering)  
Conformance Level: Core (Mandatory)  
Implements: LV-SPEC-0100, LV-SPEC-0101  
Language Target: Rust  

---

# 1. Purpose

This specification defines:

- Validator admission and removal rules
- Stake bonding and delegation model
- Slashing conditions and enforcement
- Reward distribution and stake-weighting mechanics

This document governs the economic security layer of the Lite-Vision Truth Plane.

The validator set secures:

- BFT consensus
- Deterministic state transitions
- Artifact commitments
- Economic enforcement for the Intelligence Plane

---

# 2. Definitions

Let:

- V = current validator set
- N = |V|
- w_i = stake weight of validator i
- W = Σ w_i (total voting power)
- Q = quorum threshold = ≥ 2/3 W
- f = Byzantine validators
- Epoch = fixed-length validator period

Security assumption:

f < N / 3  
Or equivalently  
Malicious voting power < 1/3 W

---

# 3. Validator Admission

## 3.1 Eligibility Requirements

A candidate validator MUST:

- Register identity (public key)
- Lock minimum stake S_min
- Publish network endpoint
- Accept slashing conditions

Optional:

- Hardware disclosure (CPU class)
- Uptime commitment

GPU hardware is NOT required.

---

## 3.2 Admission Process

Validator admission is governed by:

- Governance transaction OR
- Automatic admission if stake ≥ threshold and capacity available

Admission occurs at epoch boundaries.

State update:

V_{t+1} = V_t ∪ {candidate}

Validator power:

w_i = bonded_stake_i

---

# 4. Validator Removal

Validators may be removed by:

1. Voluntary exit (unbonding)
2. Governance removal
3. Forced removal via slashing

Removal occurs at epoch boundary.

Unbonding period:

T_unbond (e.g., X epochs)

During unbonding:

- Stake remains slashable
- Validator cannot propose blocks

---

# 5. Stake Model

## 5.1 Bonded Stake

Each validator bonds stake S_i.

Total voting power:

W = Σ S_i

Voting power proportional to stake.

Optional cap:

w_i ≤ w_max (anti-centralization safeguard)

---

## 5.2 Delegation

Delegators MAY delegate stake to validators.

Delegated stake:

S_i = self_bond_i + Σ delegated_i

Delegators share:

- Rewards
- Slashing penalties (pro-rata)

Delegators cannot vote directly.

---

# 6. Slashing Conditions

Slashing is triggered by cryptographic proof of misbehavior.

All slashing is enforced by deterministic state transition.

---

## 6.1 Double-Sign / Equivocation

Condition:

Validator signs two different votes:

- Same height
- Same view
- Different block_hash

Proof:

(sig_1, block_hash_1)
(sig_2, block_hash_2)
Where block_hash_1 ≠ block_hash_2

Penalty:

- Immediate slash of S_double
- Removal from validator set
- Jailed for J epochs

S_double MUST be significant enough to deter equivocation.

---

## 6.2 Double-Propose

Leader proposes two blocks at same height/view.

Same proof structure as double-sign.

Penalty:

Equivalent to equivocation penalty.

---

## 6.3 Surround Vote (If Enabled)

If protocol variant includes locking rules that allow:

Vote(v1, h1) and Vote(v2, h2)
Where h1 < h2 but v2 < v1 (invalid progression)

Proof MUST be formally defined if enabled.

Penalty:

Equivalent to equivocation.

---

## 6.4 Censorship Proof (Optional but Recommended)

Condition:

Validator refuses to include valid transaction over prolonged period.

Requires:

- Proof of valid transaction
- Evidence of repeated omission
- Governance or automated trigger

Penalty:

- Minor slash S_censor
- Reputation reduction
- Temporary suspension

Censorship detection MUST avoid griefing vectors.

---

## 6.5 Invalid Block Proposal

Condition:

Validator proposes block with:

- Invalid state transition
- Invalid transaction
- Invalid QC reference

If ≥ 2/3 reject and proof submitted:

Penalty:

- Slash S_invalid
- Remove proposer if repeated

---

# 7. Slashing Enforcement

When proof submitted:

1. Proof verified deterministically.
2. Stake reduced.
3. Portion burned or redistributed.
4. Validator status updated.
5. Delegators penalized proportionally.

Slashing MUST be deterministic and replayable.

---

# 8. Jailing

Jailed validators:

- Cannot propose or vote
- Remain slashable
- May rejoin after J epochs (if allowed)

Repeated offenses MAY result in permanent ban.

---

# 9. Reward Model

Validators earn rewards for:

- Block proposal
- Voting participation
- Uptime

Let:

R_block = base reward per block

Distribution:

Proposer reward:

R_prop = α * R_block

Voter pool:

R_vote = (1 - α) * R_block

Each validator i receives:

R_i = (w_i / W) * R_vote

Delegators receive proportional share after validator commission.

---

# 10. Commission Model

Validator MAY define commission rate:

c_i ∈ [0, c_max]

Delegator reward:

R_del = (1 - c_i) * delegated_share

Commission incentivizes validator operations.

---

# 11. Inflation / Fee Model

Rewards funded by:

- Inflation schedule OR
- Transaction fees OR
- Hybrid model

Inflation function:

I_t = f(epoch)

Governance MAY adjust inflation within bounded range.

---

# 12. Anti-Centralization Controls (Optional)

To prevent validator dominance:

- Stake cap per validator
- Progressive commission curve
- Governance-limited validator count
- Minimum decentralization threshold

These MUST preserve BFT safety.

---

# 13. Stake-Weighted Voting

Voting power:

w_i = effective_stake_i

QC threshold:

Σ w_i ≥ 2/3 W

Safety condition:

Malicious voting power < 1/3 W

Stake-weighting MUST be deterministic and transparent.

---

# 14. Epoch Transitions

At epoch boundary:

- Apply validator additions/removals
- Apply updated stake weights
- Reset jailed status (if applicable)

Consensus MUST reference validator set by epoch ID.

---

# 15. Economic Security Bound

To economically secure consensus:

Let:

G = gain from successful attack  
L = total slashable stake at risk  

Security requires:

L > G

Thus:

Minimum total bonded stake MUST exceed plausible attack gain.

Governance SHOULD monitor stake security margin.

---

# 16. Interaction with Intelligence Plane

Validators enforce:

- Operator stake bonding
- Receipt settlement
- Slashing for fraud (if proven)
- Capability registry integrity

Validators DO NOT:

- Execute intelligence jobs
- Evaluate RPACK contents
- Perform GPU computation

---

# 17. Rust Determinism Constraints

Slashing logic MUST:

- Use canonical encoding
- Avoid floating-point arithmetic
- Use deterministic map iteration
- Avoid system time
- Be reproducible across platforms

---

# 18. Compliance Requirements

To claim LV-SPEC-0102 compliance:

Implementation MUST:

- Support validator registration and removal
- Enforce stake bonding
- Enforce double-sign slashing
- Support delegation
- Distribute rewards deterministically
- Enforce quorum ≥ 2/3 voting power

---

# 19. Design Invariants

1. Slashing MUST be provable and deterministic.
2. Stake MUST remain bonded during unbonding period.
3. Voting power MUST be transparent.
4. Validator removal MUST not break safety.
5. Delegators share slashing penalties proportionally.
6. GPU hardware MUST NOT be required for validators.

---

# 20. Summary

LV-SPEC-0102 defines the economic backbone of Lite-Vision consensus:

- Stake-weighted validators
- BFT-secured finality
- Deterministic slashing
- Delegation model
- Reward distribution

The Truth Plane is secured by:

CPU execution + stake + BFT.

It remains independent from GPU-based intelligence execution.

---

End of LV-SPEC-0102