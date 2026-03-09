# LV-SPEC-0103 — Block, Transaction, and State Transition Format
Version: 1.0  
Status: Draft (Engineering)  
Conformance Level: Core (Mandatory)  
Implements: LV-SPEC-0100, LV-SPEC-0101, LV-SPEC-0102  
Language Target: Rust  

---

# 1. Purpose

This specification defines:

- The canonical block header and body schema
- Merkle commitment structure
- Transaction types and validation rules
- The deterministic state transition function δ

All Truth Plane state transitions MUST be:

- Deterministic
- Canonically serialized
- Hash-stable across platforms
- Replayable

---

# 2. Canonical Encoding Rules

All structures defined in this spec MUST:

- Use canonical serialization (see LV-SPEC-0602)
- Use fixed field ordering
- Avoid platform-dependent representations
- Avoid floating-point values
- Use explicit integer widths (u64, u128, etc.)

Hashing function:

H(x) = SHA-3-256(x)  (or governance-approved alternative)

---

# 3. Block Structure

A block B_h consists of:

B_h = (Header_h, Body_h)

---

# 4. Block Header Schema

Header_h MUST contain:

- chain_id: u64
- height: u64
- epoch: u64
- view: u64
- parent_hash: Hash32
- state_root: Hash32
- tx_root: Hash32
- receipt_root: Hash32
- validator_set_hash: Hash32
- timestamp: u64 (logical time only)
- proposer_id: ValidatorID
- parent_qc: QuorumCertificate

## 4.1 Header Hash

block_hash = H(serialize(Header_h))

The block hash is computed ONLY from the header.

---

# 5. Block Body Schema

Body_h contains:

- transactions: Vec<Transaction>
- receipts: Vec<AnchoredReceipt>

The Merkle roots MUST satisfy:

tx_root = MerkleRoot(transactions)
receipt_root = MerkleRoot(receipts)

---

# 6. Merkle Commitment Rules

## 6.1 Merkle Tree Construction

- Leaves are canonical serialized objects
- Leaves hashed individually
- Internal nodes hashed as H(left || right)
- Odd leaf count duplicates last leaf

MerkleRoot(empty_set) = H(0x00)

---

# 7. Transaction Structure

Transaction:

- tx_type: enum
- sender: AccountID
- nonce: u64
- payload: bytes
- signature: Signature

Transaction hash:

tx_hash = H(serialize(tx_without_signature) || signature)

---

# 8. Transaction Types

The following transaction types are defined:

## 8.1 Transfer

- From account A
- To account B
- Amount
- Fee

Validation:

- Sufficient balance
- Correct nonce
- Valid signature

---

## 8.2 Validator Register

- Public key
- Stake amount

Validation:

- Stake ≥ minimum
- No duplicate validator ID

---

## 8.3 Validator Unbond

- Validator ID
- Unbond request

Validation:

- Unbonding period initiated
- Stake remains slashable until completion

---

## 8.4 Delegation

- Delegator
- Validator
- Amount

Validation:

- Balance sufficient
- Validator exists

---

## 8.5 Governance Proposal

- Parameter change
- Target spec or protocol param
- Voting window

Validation:

- Governance deposit posted

---

## 8.6 Governance Vote

- Proposal ID
- Vote (yes/no/abstain)

Validation:

- Eligible voter
- Within voting window

---

## 8.7 Operator Registration (Intelligence Plane)

- Operator ID
- Capability declaration
- Stake bond

Validation:

- Stake locked
- Capability registry updated

---

## 8.8 Receipt Anchor

- Operator ID
- job_id
- input_hash
- output_hash (RPACK hash)
- resource_metrics_hash

Validation:

- Valid signature
- Operator registered
- Escrow exists

Truth Plane does NOT inspect RPACK contents.

---

## 8.9 Slashing Proof

- Proof type
- Evidence payload

Validation:

- Deterministic verification
- Applies penalty if valid

---

# 9. Transaction Validation Rules

For each transaction:

1. Signature MUST be valid.
2. Nonce MUST match expected account nonce.
3. State preconditions MUST hold.
4. Gas/fee MUST be sufficient.
5. Transaction MUST be canonical encoded.

Invalid transactions MUST:

- Cause rejection at proposal stage.
- Never be included in committed block.

---

# 10. Deterministic State Transition Function

State transition defined as:

T_{t+1} = δ(T_t, B_t)

Where:

B_t = ordered list of transactions

Algorithm:

1. Validate block header.
2. Verify parent hash matches previous block.
3. Verify QC validity.
4. Recompute tx_root and receipt_root.
5. For each transaction in order:
   - Validate
   - Apply state mutation
6. Compute new state_root.
7. Compare computed state_root with header.state_root.

If mismatch → block invalid.

---

# 11. State Root Computation

State is represented as:

- Account store
- Validator registry
- Stake ledger
- Governance store
- Operator registry
- Receipt registry
- Partition registry

State_root computed as:

state_root = MerkleRoot([
  AccountsRoot,
  ValidatorsRoot,
  StakeRoot,
  GovernanceRoot,
  OperatorRoot,
  ReceiptRoot,
  PartitionRoot
])

Each sub-root must itself be a Merkle root of canonical structures.

---

# 12. Determinism Constraints

The following are forbidden inside δ:

- Floating point operations
- Random number generation
- System clock usage
- Non-deterministic iteration order
- External network calls
- File I/O

All maps MUST use ordered structures (e.g., BTreeMap).

All arithmetic MUST use fixed-width integers.

---

# 13. Gas / Fee Accounting

Each transaction consumes:

gas_used = deterministic_function(tx_type, payload_size)

Fees:

fee = gas_used * gas_price

Fees distributed to:

- Proposer
- Validator pool

Gas accounting MUST be deterministic.

---

# 14. Replay Rules

Given:

- Genesis state
- Ordered list of committed blocks

Replaying δ MUST produce identical state_root on all compliant implementations.

Replay invariants:

- Hash stability
- Serialization stability
- Deterministic iteration

---

# 15. Block Validity Conditions

A block is valid if:

1. Parent exists and matches parent_hash.
2. QC for parent is valid.
3. Transactions are valid and ordered.
4. Merkle roots are correct.
5. State transition matches header state_root.
6. Validator set hash matches current epoch.

---

# 16. Interaction with Intelligence Plane

Truth Plane processes:

- Operator registration
- Job escrow
- Receipt anchoring
- Slashing

Truth Plane MUST NOT:

- Validate intelligence output semantics
- Execute kernels
- Process RPACK content

Only output_hash is anchored.

---

# 17. State Pruning Compatibility

When pruning:

- Old state snapshots MAY be removed
- state_root MUST remain provable via archive
- Historical Merkle proofs MUST remain verifiable

Pruning MUST NOT alter historical block hashes.

---

# 18. Rust Implementation Requirements

Implementation MUST:

- Use canonical serializer
- Use explicit endian ordering
- Use deterministic hashing
- Avoid unsafe non-deterministic patterns
- Pass conformance test vectors

---

# 19. Compliance Requirements

To claim LV-SPEC-0103 compliance:

Implementation MUST:

- Implement block schema as defined
- Implement Merkle root construction
- Implement all transaction types
- Implement deterministic δ
- Enforce block validity checks
- Produce identical state_root across platforms

---

# 20. Summary

LV-SPEC-0103 defines:

- Canonical block structure
- Merkle commitments
- Transaction taxonomy
- Deterministic state transition function

It ensures:

- Replayability
- Determinism
- Cross-platform consistency
- CPU-secured consensus integrity

The Truth Plane remains:

Deterministic.
Economically authoritative.
GPU-independent.

---

End of LV-SPEC-0103