# LV-SPEC-0116 — LVU Staking, Locking, and Unbonding Framework
Version: 1.0  
Status: Draft (Security / Economic Enforcement Layer)  
Conformance Level: Foundational (Mandatory for Validator-Based Deployments)  
Implements: LV-SPEC-0101, LV-SPEC-0102, LV-SPEC-0115, LV-SPEC-0701, LV-SPEC-0999  
Language: Rust  

---

# 1. Purpose

This specification defines the staking, locking, delegation, and unbonding mechanics for Lite Vision Units (LVU).

It formalizes:

- Validator staking requirements
- Delegation model
- Locking rules
- Unbonding periods
- Slashing mechanics
- Reward distribution
- Governance voting weight linkage
- Economic security bounds

LVU staking secures the Truth Plane.

---

# 2. Core Concepts

Stake: LVU locked to secure the network.  
Bonded Stake: Active stake participating in consensus.  
Delegated Stake: LVU assigned to a validator by a delegator.  
Unbonding Stake: Stake in withdrawal period.  
Slashing: Penalty reducing bonded stake for misbehavior.  

All staking state transitions MUST be deterministic.

---

# 3. Validator Bonding Requirements

A validator MUST:

- Bond at least MinValidatorStake LVU.
- Maintain bonded stake ≥ MinValidatorStake.
- Lock stake before activation.

ValidatorState:

Validator {
  operator_address,
  bonded_stake,
  delegated_stake,
  commission_rate,
  status: Active | Jailed | Unbonding,
}

MinValidatorStake is governance-defined.

---

# 4. Delegation Model

Delegators MAY:

- Delegate LVU to validator.
- Redelegate to another validator.
- Begin unbonding process.

Delegation structure:

Delegation {
  delegator_address,
  validator_address,
  amount,
  bonded: bool,
}

Delegation MUST:

- Increase validator voting power.
- Increase validator slashing exposure.
- Preserve supply invariants.

---

# 5. Bonding Process

To bond stake:

1. Lock LVU in staking contract.
2. Mark as Bonded.
3. Update validator voting power.
4. Emit deterministic event.

Bonding MUST occur at deterministic block height.

---

# 6. Unbonding Process

Unbonding requires:

UnbondingPeriod blocks.

UnbondingEntry {
  delegator,
  validator,
  amount,
  unlock_height,
}

Rules:

- Stake cannot be used during unbonding.
- Stake remains slashable during unbonding period.
- Funds released at unlock_height.

Unbonding MUST be replay-safe.

---

# 7. Slashing Mechanics

Slashing conditions include:

- Double-signing.
- Equivocation.
- Extended downtime.
- Fraud adjudication.
- Censorship proof (optional).

SlashEvent {
  validator,
  slash_fraction,
  reason,
  block_height,
}

Slashed amount:

SlashAmount = BondedStake × SlashFraction

Slashing MUST:

- Reduce bonded stake.
- Reduce delegated stake proportionally.
- Optionally burn portion.
- Optionally redistribute portion.
- Be deterministic.

---

# 8. Jailing and Rehabilitation

Validator may be:

Jailed for misconduct.

While jailed:

- Cannot propose blocks.
- Cannot earn rewards.
- May require governance reinstatement.

Rehabilitation requires:

- Bonded stake ≥ threshold.
- Governance or automatic timeout.
- Deterministic reactivation.

---

# 9. Reward Distribution

Per block:

BlockReward distributed proportionally to:

ValidatorVotingPower / TotalVotingPower

Delegator rewards:

DelegatorShare =
  ValidatorReward × (DelegationAmount / TotalValidatorStake)

Validator commission applied:

Commission = DelegatorReward × CommissionRate

Reward calculation MUST be deterministic.

---

# 10. Voting Power Calculation

VotingPower(validator) =

BondedStake + Σ DelegatedStake

Voting power snapshot MUST be:

Taken at proposal start height.

Voting power MUST not change mid-proposal.

---

# 11. Commission Rules

Validator MAY set:

CommissionRate ∈ [0, MaxCommission]

Commission changes MUST:

- Be governance-limited.
- Activate at deterministic height.
- Prevent retroactive changes.

Commission cannot exceed governance cap.

---

# 12. Redelegation Rules

Delegators MAY:

- Move stake from validator A to B.

Constraints:

- Must not bypass unbonding safety.
- Redelegation lock period MAY apply.
- Slashing exposure must persist for prior fault window.

Redelegation MUST be deterministic.

---

# 13. Economic Security Bound

Network security requires:

TotalBondedStake ≥ SecurityThreshold

SecurityThreshold may be defined as:

Minimum stake required for BFT safety.

Governance MAY enforce minimum bonding ratio.

---

# 14. Inflation Distribution

If inflation enabled:

Minted LVU distributed to:

- Validators
- Delegators

Inflation reward must follow:

BondedStake proportion.

Inflation MUST be deterministic.

---

# 15. Supply Conservation

Staking MUST NOT:

- Create LVU.
- Destroy LVU (except slashing burn).
- Violate TotalSupply invariant.

All state transitions must preserve:

Σ balances + Σ bonded + Σ unbonding = TotalSupply

---

# 16. Replay Guarantees

Replay MUST reproduce:

- Bonding events
- Unbonding entries
- Slashing events
- Reward distributions
- Commission deductions
- Voting power snapshots

Any divergence invalidates block.

---

# 17. Edge Conditions

System MUST handle deterministically:

- Simultaneous slashing and unbonding.
- Validator dropping below minimum stake.
- Delegator unbonding during validator jail.
- Large-scale slashing event.
- Inflation and slashing same block.

All arithmetic MUST use fixed-width integers.

---

# 18. Rust Implementation Requirements

Implementation MUST:

- Use u128 for LVU accounting.
- Avoid floating-point arithmetic.
- Use deterministic collections.
- Provide property tests:
  - Slashing invariants
  - Reward distribution invariants
  - Supply conservation
- Include simulation tests for:
  - Mass slashing
  - Validator churn
  - Long-term inflation dynamics

Unsafe code forbidden in staking logic.

---

# 19. Compliance Requirements

To claim LV-SPEC-0116 compliance:

Deployment MUST:

- Enforce MinValidatorStake.
- Support delegation.
- Enforce unbonding period.
- Implement slashing rules.
- Distribute rewards deterministically.
- Preserve supply invariants.
- Snapshot voting power.
- Maintain replay integrity.

Reference-tier MUST include economic security simulation.

---

# 20. Design Invariants

1. Bonded stake secures consensus.
2. Slashing must reduce stake deterministically.
3. Unbonding must delay withdrawal.
4. Rewards must be proportional and deterministic.
5. Supply must remain conserved.
6. Voting power must be snapshot-based.
7. Commission must be bounded.
8. Replay must reproduce staking state exactly.

---

# 21. Summary

LV-SPEC-0116 defines the LVU Staking, Locking, and Unbonding Framework:

- Validator bonding rules
- Delegation model
- Deterministic slashing
- Unbonding safety delay
- Reward distribution
- Commission mechanics
- Economic security guarantees
- Replay-safe staking transitions

It establishes the economic foundation of Lite-Vision security:

CPU-secured truth through bonded LVU.  
Deterministic staking mechanics.  
Economically enforced Byzantine tolerance.  
Replay-verifiable validator state.  

Staking transforms LVU into security.

---

End of LV-SPEC-0116