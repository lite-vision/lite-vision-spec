LV-SPEC-0600_Rust_Workspace_Architecture_and_Crate_Layout.md

# LV-SPEC-0600 — Rust Workspace Architecture and Crate Layout
Version: 1.0  
Status: Draft (Engineering)  
Conformance Level: Core (Mandatory for official implementations)  
Language: Rust  
MSRV: Defined herein  

---

# 1. Purpose

This specification defines the canonical Rust workspace architecture for Lite-Vision.

It formalizes:

- Workspace structure
- Crate boundaries and layering rules
- Feature flag policy
- Unsafe usage policy
- MSRV (Minimum Supported Rust Version)
- Build profiles
- Lints, formatting, and CI gates

This document ensures:

Determinism  
Security  
Maintainability  
Reproducibility  
Clear separation of planes  

---

# 2. Workspace Structure

Top-level workspace:

lite-vision/
Cargo.toml
rust-toolchain.toml
crates/
lv-core/
lv-truth/
lv-intel/
lv-network/
lv-artifact/
lv-storage/
lv-rpack/
lv-sdk/
lv-validator/
lv-operator/
lv-router/
lv-tools/
tests/
fuzz/
scripts/
docs/

All production code MUST live in `crates/`.

No circular crate dependencies allowed.

---

# 3. Crate Layering Model

Layering MUST follow strict dependency direction.

---

## 3.1 Layer 0 — Core

`lv-core`

Contains:

- Types (Hash32, BlockHeight, IDs)
- Canonical serialization
- Error types
- Domain separation constants
- Deterministic utilities

MUST NOT depend on any other internal crate.

---

## 3.2 Layer 1 — Protocol Primitives

- `lv-truth` (consensus, staking, state machine)
- `lv-intel` (kernel interface, receipts)
- `lv-rpack` (RPACK container + IR)
- `lv-storage` (tiering + GC logic)

These may depend on `lv-core`.

Must NOT depend on network or SDK crates.

---

## 3.3 Layer 2 — Infrastructure

- `lv-network`
- `lv-artifact`

May depend on Layer 0–1.

Must NOT depend on application crates.

---

## 3.4 Layer 3 — Application Nodes

- `lv-validator`
- `lv-operator`
- `lv-router`

May depend on all lower layers.

Must NOT define new consensus-critical types.

---

## 3.5 Layer 4 — SDK and Tools

- `lv-sdk`
- `lv-tools`

May depend on lower layers.

Must NOT define consensus logic.

---

# 4. Crate Boundary Rules

Each crate MUST:

- Have a clearly defined public API.
- Avoid exposing internal modules unnecessarily.
- Use `pub(crate)` for internal boundaries.
- Provide feature flags for optional components.

Consensus logic MUST reside exclusively in `lv-truth`.

GPU execution logic MUST reside exclusively in `lv-intel`.

---

# 5. Feature Flag Policy

Feature flags MUST:

- Be additive only.
- Not change consensus semantics.
- Not introduce hidden behavior changes.
- Be documented.

Recommended feature categories:

- `telemetry`
- `enterprise`
- `fuzzing`
- `serde-json`
- `grpc`
- `http`
- `encryption`
- `archive-node`

Consensus behavior MUST NOT be gated behind feature flags.

---

# 6. Unsafe Code Policy

Unsafe Rust is allowed only if:

- Necessary for performance or FFI.
- Fully documented.
- Covered by tests.
- Encapsulated in minimal scope.
- Reviewed by two maintainers.

Each `unsafe` block MUST include:

```rust
// SAFETY: Explanation of invariant and justification

Unsafe code MUST NOT:
	•	Affect consensus determinism.
	•	Bypass memory safety guarantees without justification.

Unsafe usage MUST be tracked in a central audit file.

⸻

7. MSRV (Minimum Supported Rust Version)

MSRV MUST be declared in:
	•	Cargo.toml
	•	rust-toolchain.toml

Example:

MSRV = 1.75.0

Policy:
	•	MSRV updates require minor version bump.
	•	MSRV must be tested in CI.
	•	No usage of unstable features in stable builds.

⸻

8. Build Profiles

Defined in workspace root:

⸻

8.1 dev
	•	Debug symbols enabled
	•	Overflow checks enabled
	•	Assertions enabled
	•	Lower optimization

⸻

8.2 release
	•	Optimized
	•	Overflow checks enabled (consensus-critical crates)
	•	Debug assertions disabled
	•	Panic strategy: abort (for node binaries)

⸻

8.3 fuzz
	•	Sanitizers enabled
	•	AddressSanitizer (if supported)
	•	Lower optimization

⸻

8.4 deterministic (optional)

Build profile for deterministic replay:
	•	Fixed feature set
	•	Deterministic RNG
	•	Logging minimized
	•	Telemetry disabled

⸻

9. Determinism Requirements

Consensus-critical crates (lv-core, lv-truth) MUST:
	•	Avoid floating-point arithmetic.
	•	Avoid system time calls.
	•	Avoid nondeterministic RNG.
	•	Avoid unordered map iteration (use BTreeMap).
	•	Use fixed-width integers.

All hashing MUST use canonical encoding.

⸻

10. Lints and Static Analysis

Workspace MUST enable:

#![forbid(unsafe_op_in_unsafe_fn)]
#![deny(warnings)]
#![deny(clippy::all)]
#![deny(clippy::pedantic)]

Recommended additional lints:
	•	clippy::unwrap_used
	•	clippy::expect_used
	•	clippy::panic
	•	clippy::todo
	•	clippy::dbg_macro

Consensus crates MUST forbid:
	•	unwrap()
	•	expect()
	•	panic!()

Except in tests.

⸻

11. Formatting

Formatting MUST use:
	•	rustfmt stable profile
	•	Workspace-wide configuration

cargo fmt --check MUST pass in CI.

No manual formatting deviations allowed.

⸻

12. CI Gates

CI MUST enforce:
	1.	Build on MSRV.
	2.	Build on latest stable Rust.
	3.	Run golden tests.
	4.	Run integration tests.
	5.	Run clippy with denied warnings.
	6.	Run format check.
	7.	Run fuzz smoke tests.
	8.	Run documentation build (cargo doc).
	9.	Verify no new unsafe blocks without review.

Release builds MUST:
	•	Embed git commit hash.
	•	Produce reproducible binaries (where feasible).

⸻

13. Documentation Policy

Each crate MUST:
	•	Include crate-level docs.
	•	Document all public APIs.
	•	Include examples.
	•	Document invariants.

Consensus-critical functions MUST document:
	•	State transition assumptions
	•	Safety invariants
	•	Determinism constraints

⸻

14. Dependency Policy

Rules:
	•	Avoid large transitive dependency trees.
	•	Avoid crates with unstable APIs.
	•	Avoid crates using heavy unsafe internally (unless audited).
	•	Lock dependency versions in Cargo.lock for binaries.

Security-sensitive crates MUST be audited.

⸻

15. Testing Requirements

Each crate MUST include:
	•	Unit tests
	•	Property tests (where applicable)
	•	Integration tests (if external behavior)

Consensus crates MUST have:
	•	Golden state transition tests.

Network and parser crates MUST have:
	•	Fuzz targets.

⸻

16. Binary Layout

Binary crates:
	•	lv-validator
	•	lv-operator
	•	lv-router
	•	lv-cli (optional)

Binaries MUST:
	•	Support structured logging.
	•	Support graceful shutdown.
	•	Support configuration via file + env.
	•	Validate configuration before start.

⸻

17. Configuration Management

Configuration:
	•	TOML or YAML
	•	Strict schema validation
	•	Versioned
	•	No silent default overrides

Consensus parameters MUST be sourced from on-chain state, not config file.

⸻

18. Reproducible Builds

Recommended:
	•	Use cargo vendor
	•	Lock dependencies
	•	Provide Docker build recipe
	•	Provide checksum for release artifacts

Enterprise deployments SHOULD verify binary hashes before rollout.

⸻

19. Compliance Requirements

To claim LV-SPEC-0600 compliance:

Implementation MUST:
	•	Follow defined crate layering
	•	Enforce MSRV
	•	Enforce lint policy
	•	Forbid unsafe without justification
	•	Enforce deterministic constraints
	•	Implement CI gates
	•	Maintain canonical formatting
	•	Document public APIs

Reference-tier MUST provide reproducible build pipeline.

⸻

20. Design Invariants
	1.	Consensus logic must be isolated.
	2.	GPU logic must not bleed into Truth Plane.
	3.	No feature flag may change consensus semantics.
	4.	Unsafe must be minimized and justified.
	5.	Builds must be reproducible.
	6.	Lints must fail on warnings.
	7.	Determinism must be enforced structurally.
	8.	CI must prevent regression.

⸻

21. Summary

LV-SPEC-0600 defines the Rust workspace foundation for Lite-Vision:
	•	Clear crate layering
	•	Strict feature flag discipline
	•	Determinism enforcement
	•	Unsafe code policy
	•	MSRV definition
	•	Lint and CI gates
	•	Reproducible builds

It ensures the implementation remains:

Secure.
Deterministic.
Maintainable.
Auditable.
Enterprise-ready.

Lite-Vision remains:

CPU-secured truth.
GPU-powered intelligence.
Hash-anchored state.
Rust-disciplined by design.

⸻

End of LV-SPEC-0600

