# LV-SPEC-0300 — Render Offload Model and Guarantees
Version: 1.0  
Status: Draft (Engineering)  
Conformance Level: Core (Mandatory)  
Implements: LV-SPEC-0103, LV-SPEC-0203, LV-SPEC-0205  
Language Target: Rust  

---

# 1. Purpose

This specification defines the Render Offload Model of Lite-Vision.

It formalizes:

- The invariant that the network never computes pixels
- Client-side rendering responsibilities
- Trust boundaries between network and renderer
- Determinism controls (“determinism knobs”)
- Renderer profile classes

Lite-Vision produces structured artifacts (RPACK).
Clients render pixels.

Consensus and intelligence compute are strictly separated from rendering.

---

# 2. Core Invariant

## 2.1 Pixel Prohibition Rule

The Lite-Vision network MUST NOT:

- Render raster images
- Compute final pixel buffers
- Execute framebuffers
- Provide GPU rendering as a service
- Commit pixel hashes to the Truth Plane

The network outputs:

RPACK (Render Packet)

Rendering occurs entirely on clients.

---

# 3. RPACK Output Model

The Intelligence Plane produces:

RPACK = StructuredSceneIR + Assets + Metadata

RPACK includes:

- Scene graph
- Geometry references
- Shader references
- Procedural seeds
- Animation timelines
- Deterministic metadata

RPACK does NOT include:

- Framebuffers
- Composited images
- Rasterized outputs

---

# 4. Client Responsibilities

Clients are responsible for:

1. Fetching RPACK artifacts
2. Fetching referenced assets
3. Executing renderer engine
4. Producing pixel output
5. Managing renderer configuration

Clients MAY:

- Use custom engines
- Apply post-processing
- Use different hardware

Rendering consistency is not guaranteed unless deterministic profile used.

---

# 5. Trust Model

## 5.1 Network Trust

The network guarantees:

- RPACK integrity (via output_hash)
- Kernel execution accountability
- Artifact hash commitment
- Deterministic replay (if required)

The network does NOT guarantee:

- Visual fidelity
- Exact pixel identity across devices
- GPU driver consistency
- Shader precision consistency

---

## 5.2 Client Trust Boundary

Clients must trust:

- Their own renderer implementation
- Local GPU hardware
- Driver correctness

If pixel-perfect reproducibility required:

Client MUST use deterministic renderer profile.

---

# 6. Renderer Profiles

Lite-Vision defines standardized renderer profiles.

---

## 6.1 Soft Profile (Default)

- Hardware-accelerated
- Floating-point allowed
- Post-processing allowed
- Performance optimized

Pixels may differ across:

- GPU vendors
- Drivers
- OS versions

Suitable for creative/interactive applications.

---

## 6.2 Deterministic Profile

- Fixed-point math
- Deterministic shader subset
- Fixed random seed
- No hardware-specific extensions
- Stable draw order
- No non-deterministic blending

Deterministic profile aims for:

Pixel reproducibility across compliant engines.

Used for:

- Audit-sensitive visualization
- Scientific reproducibility
- Formal verification scenarios

---

## 6.3 Reference Profile

- Minimal shader set
- Software rasterization option
- Strict ordering guarantees
- Stable floating-point emulation

Used for:

- Conformance testing
- Cross-client comparison
- Replay verification

---

# 7. Determinism Knobs

JobTicket MAY define:

RendererConstraints {
  require_deterministic_render: bool,
  shader_precision_mode: enum,
  fixed_seed: Option<Hash32>,
  stable_draw_order: bool,
  post_processing_allowed: bool,
  color_space: enum
}

These constraints are:

- Included in RPACK metadata
- Enforced by compliant clients
- Not enforced by Truth Plane

---

# 8. RPACK Integrity Guarantees

RPACK integrity guaranteed by:

output_hash = H(canonical_RPACK)

Truth Plane anchors output_hash.

Clients MUST verify:

H(downloaded_RPACK) == anchored_output_hash

Failure → reject artifact.

---

# 9. Asset Handling

Assets referenced in RPACK MUST be:

- Content-addressed
- Hash-verified
- Immutable

Asset hash mismatch → client reject.

Network may store asset hash, not bytes (optional).

---

# 10. Rendering Determinism Scope

Deterministic rendering guarantees:

- Same RPACK
- Same renderer profile
- Same version
- Same seed
- Same deterministic settings

→ Identical pixel output.

Different hardware MAY still introduce minor floating-point variation unless:

- Reference profile used.

---

# 11. Anti-Coupling Guarantee

Rendering MUST remain decoupled from:

- Consensus layer
- Verification layer
- Slashing logic
- Escrow settlement

Rendering errors MUST NOT affect:

- Truth Plane state
- Receipt validity
- Economic settlement

---

# 12. Network Resource Allocation Guarantee

All GPUs on the network are reserved for:

- Intelligence computation
- Verification
- Redundant execution

Network GPUs MUST NOT be used for pixel rendering tasks.

Clients provide rendering GPUs independently.

---

# 13. Replay and Audit

For deterministic jobs:

Audit flow:

1. Retrieve RPACK.
2. Retrieve renderer profile.
3. Execute deterministic renderer.
4. Compare pixel output hash (optional).

Pixel hash may be locally computed but never anchored in Truth Plane.

---

# 14. Security Considerations

Clients MUST:

- Validate RPACK hash before rendering.
- Validate asset hashes.
- Avoid executing arbitrary unverified shaders.

Renderer SHOULD:

- Sandbox shader execution.
- Limit GPU memory access.
- Prevent arbitrary file access.

Network is not responsible for client-side GPU security.

---

# 15. Failure Handling

If client fails to render:

- May retry
- May downgrade profile
- May switch renderer implementation

Rendering failure does NOT invalidate:

- Job execution
- Receipt anchoring
- Settlement

---

# 16. Conformance Requirements

To claim LV-SPEC-0300 compliance:

Implementation MUST:

- Never render pixels on network nodes
- Produce RPACK as structured output
- Anchor only output_hash
- Support renderer profile metadata
- Support deterministic render flag
- Separate rendering hardware from consensus

Enterprise mode MAY provide:

- Reference renderer implementation
- Deterministic conformance test suite

---

# 17. Design Invariants

1. Network never computes pixels.
2. Rendering occurs only on clients.
3. RPACK is hash-anchored.
4. Pixel buffers are never committed to ledger.
5. Deterministic rendering is optional but supported.
6. Rendering errors do not affect consensus.

---

# 18. Summary

LV-SPEC-0300 establishes the Render Offload Model:

- Structured artifact output (RPACK)
- Client-side rendering
- GPU dedication to intelligence
- Determinism knobs for reproducibility
- Strict separation from consensus

Lite-Vision ensures:

CPU-secured truth.
GPU-powered cognition.
Client-rendered reality.

Without collapsing execution into consensus.

---

End of LV-SPEC-0300