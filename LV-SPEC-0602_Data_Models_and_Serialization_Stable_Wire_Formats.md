# LV-SPEC-0602 — Data Models and Serialization (Stable Wire Formats)
Version: 1.0  
Status: Draft (Engineering)  
Conformance Level: Core (Mandatory for all interoperable nodes)  
Implements: LV-SPEC-0103, LV-SPEC-0301, LV-SPEC-0400, LV-SPEC-0500, LV-SPEC-0601  
Language: Rust  

---

# 1. Purpose

This specification defines the stable wire format and canonical serialization rules for Lite-Vision.

It formalizes:

- Canonical encoding rules
- Hash-stable representations
- Backward and forward compatibility guarantees
- Commitment-safe serialization
- Versioning discipline

All committed hashes (e.g., state_root, ArtifactID, receipt_hash) MUST be derived from canonical, stable encodings.

Serialization MUST be:

Deterministic  
Platform-independent  
Versioned  
Hash-stable  

---

# 2. Serialization Domains

Lite-Vision defines three serialization domains:

1. Wire Format (network transport)
2. Storage Format (on-disk representation)
3. Commitment Format (hash input representation)

These formats MAY differ, but:

Commitment Format MUST be canonical and stable.

---

# 3. Canonical Encoding Rules

---

## 3.1 General Principles

Canonical encoding MUST:

- Use fixed field ordering
- Use fixed integer widths
- Use explicit length prefixes
- Avoid implicit defaults
- Avoid platform-dependent padding
- Avoid unordered maps
- Avoid float ambiguity

---

## 3.2 Primitive Types

| Type          | Encoding Rule |
|---------------|--------------|
| u8–u128       | Little-endian fixed width |
| i8–i128       | Little-endian two’s complement |
| bool          | 1 byte (0x00 or 0x01) |
| Hash32        | 32 raw bytes |
| String        | length-prefixed UTF-8 |
| Vec<T>        | length-prefixed, ordered elements |
| Option<T>     | 1 byte tag + value if present |
| Enum          | u16 discriminant + variant payload |

All integers MUST use fixed-width encoding.

Variable-length integers are FORBIDDEN in commitment format.

---

## 3.3 Map Encoding

Maps MUST:

- Use deterministic key ordering (lexicographic)
- Encode as ordered Vec<(Key, Value)>
- Reject duplicate keys

HashMap MUST NOT be serialized directly.

---

## 3.4 Floating-Point Encoding

Floating-point values MUST:

- Use IEEE-754 encoding
- Canonicalize NaN
- Normalize ±0
- Be forbidden in consensus-critical structures

If float included in commitment context:

Canonical float encoding MUST normalize all edge cases.

---

# 4. Wire Format (Network Layer)

Primary wire format: Protobuf-like or equivalent schema-driven format.

Requirements:

- Field numbering fixed and documented
- No field reordering
- Unknown fields ignored (forward compatibility)
- Backward-compatible field addition only

Wire format MAY use:

- Protobuf
- Prost
- FlatBuffers
- Custom codec

BUT commitment hashing MUST NOT use raw protobuf bytes.

Wire format is transport-level only.

---

# 5. Commitment Format

Commitment format is canonical.

Used for:

- state_root hashing
- receipt_hash
- ArtifactID
- snapshot_root
- ReplayBundleHash
- RPACK root

Commitment format MUST:

- Use canonical byte encoding (LV Canonical Format, LVCF)
- Avoid schema evolution ambiguity
- Avoid field omission
- Be stable across versions

Hash = H(LVCF_bytes)

---

# 6. LVCF (Lite-Vision Canonical Format)

LVCF encoding rules:

1. Fixed field ordering as declared in struct.
2. No omitted fields (even if default).
3. No optional reordering.
4. Explicit version tag for top-level objects.
5. Deterministic nested encoding.
6. No implicit padding.

Example:

struct Receipt {
    version: u16,
    job_id: Hash32,
    kernel_id: Hash32,
    input_hash: Hash32,
    output_hash: Hash32,
    metrics: Metrics,
}

LVCF = concat(
    version,
    job_id,
    kernel_id,
    input_hash,
    output_hash,
    encode(metrics)
)

---

# 7. Backward Compatibility

---

## 7.1 Field Addition Rules

Fields MAY be added if:

- Appended to struct definition
- Assigned new version number
- Default value defined

Consensus-critical structures require governance upgrade for field addition.

---

## 7.2 Field Removal

Field removal requires:

- Major version increment
- Governance upgrade
- Explicit migration logic

Removal MUST NOT change historical hash semantics.

---

## 7.3 Enum Evolution

Enum discriminants MUST:

- Be stable
- Never reused
- Never reordered

New variants MAY be appended.

---

# 8. Forward Compatibility

Nodes MUST:

- Ignore unknown fields in wire format
- Reject unknown discriminants in commitment format
- Validate version tags before decoding

Commitment format does NOT allow partial decoding.

---

# 9. Versioning Strategy

Each major data structure MUST include:

version: u16

Rules:

- Minor changes: version unchanged (wire format only)
- Major commitment change: version increment
- Version included in hash input

Version MUST be first field in commitment format.

---

# 10. Hash-Stable Representations

Commitment hashing MUST:

- Use LVCF encoding
- Use domain separation
- Use fixed hash function (e.g., SHA-256 or BLAKE3)
- Avoid platform-specific behavior

Hash inputs MUST be byte-identical across platforms.

---

# 11. RPACK and Artifact Serialization

RPACK:

- Chunk table ordered
- Chunk hashes canonical
- Metadata sorted
- Compression applied after canonicalization
- Hash computed before encryption

ArtifactID MUST be computed from canonical plaintext.

---

# 12. Snapshot and Partition Serialization

Snapshot MUST include:

- version
- partition_id
- state_root
- canonical state encoding
- timestamp (logical)

Snapshot hash MUST remain stable across nodes.

---

# 13. Replay Bundle Serialization

ReplayBundle MUST:

- Use canonical encoding
- Include kernel version
- Include deterministic seed
- Include input_hash
- Include output_hash
- Include resource caps

ReplayBundleHash MUST be stable across platforms.

---

# 14. Testing Requirements

Serialization MUST pass:

- Round-trip tests
- Cross-platform tests
- Golden byte tests
- Version upgrade tests
- Backward compatibility tests

Golden bytes MUST be stored in test fixtures.

---

# 15. Rust Implementation Guidelines

Implementation MUST:

- Use dedicated canonical encoding module
- Avoid direct serde_json for commitments
- Avoid non-deterministic map iteration
- Enforce fixed-width integer encoding
- Normalize floats before hashing
- Use BTreeMap for deterministic ordering
- Disallow HashMap in commitment contexts

Serialization logic MUST be unit-tested independently.

---

# 16. CI Enforcement

CI MUST:

- Run golden serialization tests
- Compare byte-for-byte outputs
- Fail on hash mismatch
- Run cross-platform compatibility test (if supported)
- Verify no field reordering occurred

Schema changes MUST require explicit approval.

---

# 17. Compliance Requirements

To claim LV-SPEC-0602 compliance:

Implementation MUST:

- Implement LVCF encoding
- Enforce canonical ordering
- Support version tagging
- Maintain backward compatibility rules
- Provide golden serialization tests
- Ensure hash stability across platforms
- Prevent non-deterministic encoding

Reference-tier MUST publish canonical schema definitions.

---

# 18. Design Invariants

1. Hash inputs must be byte-identical across nodes.
2. Field ordering must never change silently.
3. Maps must always be deterministically ordered.
4. Unknown wire fields must not break nodes.
5. Commitment format must never omit fields.
6. Enum discriminants must never be reused.
7. Version tags must be explicit.
8. Serialization must not depend on compiler layout.

---

# 19. Summary

LV-SPEC-0602 defines Lite-Vision data model and serialization guarantees:

- Canonical encoding (LVCF)
- Stable wire format discipline
- Backward/forward compatibility rules
- Hash-stable commitment representations
- Strict versioning and ordering

It ensures Lite-Vision remains:

Interoperable.
Deterministic.
Audit-friendly.
Consensus-safe.
Future-proof.

---

End of LV-SPEC-0602