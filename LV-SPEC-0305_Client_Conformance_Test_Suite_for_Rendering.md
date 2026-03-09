# LV-SPEC-0305 — Client Conformance Test Suite for Rendering
Version: 1.0  
Status: Draft (Engineering)  
Conformance Level: Core (Mandatory for certified clients)  
Implements: LV-SPEC-0300, LV-SPEC-0301, LV-SPEC-0302, LV-SPEC-0303, LV-SPEC-0304  
Language Target: Rust (reference harness)

---

# 1. Purpose

This specification defines the Client Conformance Test Suite (CCTS) for Lite-Vision rendering clients.

It formalizes:

- Schema validators
- Deterministic-seed reproducibility tests
- Reference scenes and expected semantic outcomes
- Compatibility tiers for client engines

The Conformance Suite verifies:

- Correct RPACK decoding
- Correct Scene IR interpretation
- Asset integrity validation
- Deterministic rendering compliance (when required)

The network never renders pixels.
Conformance is enforced at the client layer.

---

# 2. Conformance Philosophy

Lite-Vision rendering is:

- Client-side
- Engine-agnostic
- Profile-driven
- Optionally deterministic

Therefore, conformance MUST test:

1. Structural correctness
2. Semantic correctness
3. Deterministic reproducibility (if enabled)
4. Asset integrity behavior
5. Delta update handling

Pixel-perfect equality is only required in deterministic tiers.

---

# 3. Test Suite Architecture

The Client Conformance Test Suite consists of:

1. Static Validators
2. Runtime Rendering Tests
3. Deterministic Replay Tests
4. Asset Integrity Tests
5. Delta Application Tests
6. Performance Envelope Tests (non-normative)

Reference harness MUST be implemented in Rust.

---

# 4. Schema Validators

## 4.1 RPACK Container Validator

Validates:

- Magic bytes
- Version
- Chunk table integrity
- Chunk content_hash correctness
- Canonical ordering
- Compression hooks
- Encryption hooks (structure only)

Failure MUST reject RPACK.

---

## 4.2 Scene IR Schema Validator

Validates:

- Node graph acyclic property
- Valid parent-child references
- Valid constraint targets
- Valid camera references
- Valid material references
- No pixel buffer presence
- Canonical ordering of nodes

Validation MUST be deterministic.

---

## 4.3 Asset Reference Validator

Validates:

- AssetID format (Hash32)
- Asset hash verification
- No raw URLs embedded
- Compression/decompression integrity
- Encryption metadata structure

---

# 5. Deterministic-Seed Tests

Applicable when:

renderer_profile == Deterministic
OR
deterministic_mode flag set

Test procedure:

1. Load reference RPACK.
2. Apply deterministic_seed.
3. Render.
4. Capture pixel buffer.
5. Compute PixelHash.
6. Compare to reference PixelHash.

PixelHash must match exactly.

---

## 5.1 Seed Replay Test

Procedure:

1. Render with seed S.
2. Render again with same seed S.
3. Compare PixelHash.

Must match.

Changing seed must produce predictable difference.

---

## 5.2 Floating-Point Stability Test

Test scenes include:

- Deep transform chains
- Small-scale geometry
- Repeated animation interpolation

PixelHash must remain stable across:

- Multiple executions
- Same hardware
- Same engine version

---

# 6. Reference Scenes

Conformance Suite includes standardized reference scenes.

---

## 6.1 Basic Graph Scene

Tests:

- Node hierarchy
- Transform propagation
- Visibility flags

Expected outcome:

Semantic object positions verified via bounding box checks.

---

## 6.2 Constraint Scene

Tests:

- LookAt constraint
- IK chain
- Parent constraint

Expected outcome:

Constraint resolution matches reference transforms.

---

## 6.3 Lighting Scene

Tests:

- Directional light
- Point light
- Shadow flag

Expected outcome:

Semantic lighting properties match expected intensity and direction.

---

## 6.4 Animation Scene

Tests:

- Keyframe interpolation
- Channel binding
- Deterministic playback

Expected outcome:

Frame N produces expected transform hash.

---

## 6.5 Progressive Refinement Scene

Tests:

- RPACK-Δ application
- LOD replacement
- Texture upgrade

Expected outcome:

Final RPACK_hash matches expected.

---

# 7. Semantic Outcome Validation

Conformance suite evaluates semantic outcomes via:

- Transform hash validation
- Node graph hash validation
- Bounding volume validation
- Material binding validation
- Animation timeline hash validation

Pixel comparison only required for deterministic tier.

---

# 8. Delta Update Tests

## 8.1 Sequential Delta Test

Procedure:

1. Load R₀.
2. Apply Δ₁.
3. Apply Δ₂.
4. Verify R₂_hash matches expected.

---

## 8.2 Idempotency Test

Procedure:

1. Apply Δ₁ twice.
2. Verify state unchanged after second application.

---

## 8.3 Conflict Detection Test

Apply delta with incorrect expected_hash.

Client MUST reject delta.

---

# 9. Asset Integrity Tests

Tests include:

- Corrupted asset byte
- Wrong hash
- Missing asset (offline mode)
- Encrypted asset with wrong key

Expected behavior:

- Reject corrupted asset
- Trigger fallback if allowed
- Reject rendering if deterministic profile requires exact asset

---

# 10. Compatibility Tiers

Clients may declare compatibility tier.

---

## Tier 0 — Basic Compatibility

Supports:

- RPACK decoding
- Scene IR parsing
- Soft rendering
- Asset hash verification

No deterministic guarantee.

---

## Tier 1 — Deterministic Rendering

Supports:

- Deterministic profile
- PixelHash reproducibility
- Seed replay
- Stable ordering

Required for high-assurance jobs.

---

## Tier 2 — Reference Renderer

Supports:

- Software fallback mode
- Strict floating-point emulation
- Formal deterministic validation
- Full delta compliance
- Offline mode completeness

Used for certification and auditing.

---

# 11. Certification Process

To achieve certification:

1. Pass all Tier 0 tests.
2. If claiming Tier 1 → pass deterministic tests.
3. If claiming Tier 2 → pass full suite including reference hash comparisons.

Certification artifact:

ClientConformanceCertificate {
  engine_name,
  engine_version,
  tier,
  test_suite_version,
  certificate_hash,
  signature
}

---

# 12. Test Suite Versioning

Test suite version MUST be included in:

- Certification
- Client metadata

New test suite versions MUST remain backward compatible.

Major changes require re-certification.

---

# 13. Performance Tests (Non-Normative)

Optional performance metrics include:

- Scene load time
- Delta application time
- Memory footprint
- Asset cache efficiency

Performance does not affect conformance.

---

# 14. Rust Reference Harness

Reference harness MUST:

- Load RPACK
- Apply validation rules
- Execute deterministic profile
- Compute PixelHash
- Compare expected hashes
- Produce structured test report

Report format MUST be canonical JSON.

---

# 15. Security Requirements

Conformance suite MUST test:

- Shader sandbox enforcement
- Asset integrity rejection
- Memory bounds validation
- Delta conflict rejection

Clients failing security tests MUST not be certified.

---

# 16. Determinism Edge Cases

Suite MUST test:

- Zero-scale transforms
- Very large coordinate values
- Deep nested constraints
- Rapid animation sampling
- Floating-point boundary conditions

Deterministic clients must remain stable.

---

# 17. Compliance Requirements

To claim LV-SPEC-0305 compliance:

Client MUST:

- Implement schema validators
- Support deterministic-seed testing
- Pass reference scenes
- Enforce asset integrity
- Support RPACK-Δ idempotency
- Declare compatibility tier honestly

Tier claims must match test results.

---

# 18. Design Invariants

1. Conformance is client-enforced.
2. Pixel hashing required only for deterministic tiers.
3. Scene IR semantics must be preserved.
4. Delta application must be idempotent.
5. Asset integrity must be verified.
6. Rendering remains separate from consensus.

---

# 19. Summary

LV-SPEC-0305 defines the Client Conformance Test Suite:

- Structural validation
- Deterministic reproducibility testing
- Semantic outcome verification
- Compatibility tiers
- Certification artifacts

It ensures:

Interoperable clients.
Deterministic rendering when required.
Asset integrity enforcement.
Delta correctness.
Renderer independence.

Lite-Vision remains:

Consensus-separated.
Compute-scalable.
Client-rendered.
Deterministically verifiable when required.

---

End of LV-SPEC-0305