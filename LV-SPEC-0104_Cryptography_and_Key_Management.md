# LV-SPEC-0104 — Cryptography and Key Management
Version: 1.0  
Status: Draft (Engineering)  
Conformance Level: Core (Mandatory)  
Implements: LV-SPEC-0100, LV-SPEC-0101, LV-SPEC-0103  
Language Target: Rust  

---

# 1. Purpose

This specification defines the cryptographic primitives and key management rules for the Lite-Vision Truth Plane.

It covers:

- Signature schemes (Ed25519 and optional BLS)
- Hashing algorithms
- Merkle tree construction
- Key rotation procedures
- Multi-signature and threshold options
- Domain separation and replay protection

All cryptographic operations MUST be deterministic, canonical, and cross-platform consistent.

---

# 2. Cryptographic Primitives

## 2.1 Hash Function

Default hash function:

H(x) = SHA3-256(x)

Properties required:

- Collision resistance
- Preimage resistance
- Deterministic output
- Stable across platforms

Governance MAY upgrade hash function via epoch transition with migration rules.

---

## 2.2 Signature Schemes

Lite-Vision supports two signature modes:

### 2.2.1 Ed25519 (Default, Required)

Used for:

- Transaction signatures
- Validator votes (non-aggregated mode)
- Operator receipts
- Slashing proofs

Requirements:

- RFC 8032 compliant
- Deterministic signing
- Canonical encoding of public keys and signatures

Public key length: 32 bytes  
Signature length: 64 bytes  

---

### 2.2.2 BLS12-381 (Optional, Recommended for Aggregation)

Used for:

- Validator vote aggregation
- Quorum certificates

Requirements:

- BLS12-381 curve
- Deterministic signing
- Secure aggregation
- Domain-separated hash-to-curve

Public key length: 48 bytes  
Signature length: 96 bytes  

If BLS is enabled:

- QC signatures MUST be aggregated
- Validator set hash MUST reference BLS public keys

---

# 3. Merkle Trees

## 3.1 Construction Rules

- Leaves are canonical serialized objects
- Leaf hash = H(serialized_object)
- Internal node = H(left || right)
- If odd number of nodes, duplicate last leaf

MerkleRoot(empty_set) = H(0x00)

---

## 3.2 State Root

State root is Merkle root over ordered sub-roots:

state_root = H(
  AccountsRoot ||
  ValidatorsRoot ||
  StakeRoot ||
  GovernanceRoot ||
  OperatorRoot ||
  ReceiptRoot ||
  PartitionRoot
)

Ordering MUST be fixed and immutable.

---

# 4. Key Types

Lite-Vision defines the following key roles:

## 4.1 Validator Consensus Key

Used for:

- Block proposal
- Vote signing
- QC aggregation

May be:

- Ed25519 (non-aggregated)
- BLS (aggregated mode)

---

## 4.2 Account Key

Used for:

- Transactions
- Governance voting
- Delegation

Must use Ed25519 in Core.

---

## 4.3 Operator Key (Intelligence Plane)

Used for:

- Receipt signing
- Capability registration

Must use Ed25519 in Core.

---

# 5. Key Rotation

## 5.1 Validator Key Rotation

Validator MAY rotate consensus key by:

1. Submitting KeyRotation transaction
2. Signing with old key
3. Providing new public key

Rotation takes effect at next epoch boundary.

Old key remains valid for:

- Slashing proof verification
- Historical signature validation

---

## 5.2 Account Key Rotation

Accounts MAY rotate keys by:

- Submitting rotation transaction
- Proving control of old key

State update:

AccountID → new public key

Rotation MUST preserve account balance and nonce.

---

# 6. Multi-Signature Support

Lite-Vision supports multi-sig accounts.

## 6.1 Multi-Sig Account Model

MultiSigAccount:

- list of public keys
- threshold t
- total keys n

Transaction valid if:

≥ t signatures valid

Signatures MUST be:

- Unique
- Canonically ordered
- Verified deterministically

---

# 7. Threshold Signatures (Optional)

If BLS aggregation enabled:

Validators MAY use threshold signing:

- t-of-n threshold scheme
- Aggregated QC signatures

Requirements:

- Deterministic aggregation
- Signer set explicitly recorded
- Aggregation proof verifiable

---

# 8. Domain Separation

All signatures MUST use domain-separated message hashing.

Domain separation prevents cross-context replay.

Examples:

DOMAIN_BLOCK_PROPOSAL = "LV-BLOCK-V1"
DOMAIN_VOTE = "LV-VOTE-V1"
DOMAIN_TRANSACTION = "LV-TX-V1"
DOMAIN_RECEIPT = "LV-RECEIPT-V1"
DOMAIN_SLASH_PROOF = "LV-SLASH-V1"

Signed message format:

H(DOMAIN || canonical_message)

Changing domain invalidates signature.

---

# 9. Replay Protection

Replay protection enforced via:

## 9.1 Chain ID

Each block header includes:

chain_id

Transactions MUST include chain_id.

Prevents cross-chain replay.

---

## 9.2 Nonce

Each account maintains:

nonce

Transaction valid only if:

tx.nonce == account.nonce

Prevents duplicate execution.

---

## 9.3 Height/View Binding (Consensus Votes)

Votes MUST include:

- height
- view
- block_hash

Prevents vote replay across views/heights.

---

# 10. Slashing Proof Verification

Slashing proofs MUST include:

- Conflicting signed messages
- Validator identity
- Canonical serialization

Proof verification MUST:

- Validate both signatures
- Confirm same height/view
- Confirm different block_hash

Deterministic result required.

---

# 11. Randomness (Optional)

Truth Plane MUST NOT use non-deterministic randomness.

If randomness required (e.g., leader selection):

- Derived from previous block hash
- Deterministic function:

leader = H(prev_block_hash || view) mod N

---

# 12. Secure Storage Requirements

Validators and operators MUST:

- Protect private keys via secure storage
- Support hardware security modules (optional)
- Support encrypted key files
- Never expose raw private keys over network

Key compromise protocol defined in security hardening spec.

---

# 13. Hash Stability Requirements

Hashes MUST:

- Be computed on canonical byte representation
- Be endian-consistent
- Be stable across Rust compiler versions

Any serialization change requires version bump.

---

# 14. Cryptographic Agility

Governance MAY upgrade:

- Hash function
- Signature scheme
- Curve parameters

Upgrade process:

1. Governance vote
2. Epoch transition
3. Dual-support period
4. Final migration

Backward compatibility MUST be preserved during transition.

---

# 15. Rust Implementation Requirements

Implementations MUST:

- Use vetted cryptographic crates
- Avoid custom crypto implementations
- Enforce constant-time verification
- Avoid unsafe memory exposure
- Zero memory after key use (if possible)

Crates SHOULD be audited and widely adopted.

---

# 16. Compliance Requirements

To claim LV-SPEC-0104 compliance:

Implementation MUST:

- Support Ed25519
- Support SHA3-256
- Implement Merkle tree as defined
- Enforce domain separation
- Enforce replay protection
- Support key rotation
- Support multi-sig accounts

BLS support MAY be optional in Core, required in Enterprise.

---

# 17. Security Invariants

1. No signature valid across domains.
2. No transaction replayable across chains.
3. No vote replayable across views.
4. Hash commitments deterministic.
5. Key rotation does not invalidate historical proofs.

---

# 18. Summary

LV-SPEC-0104 defines:

- Cryptographic foundations
- Signature standards
- Key lifecycle management
- Replay prevention
- Domain separation
- Deterministic hashing

These primitives secure:

- BFT consensus
- Slashing enforcement
- Artifact commitments
- Operator accountability

The Truth Plane remains:

Deterministic.
Cryptographically sound.
CPU-secured.
Platform-consistent.

---

End of LV-SPEC-0104