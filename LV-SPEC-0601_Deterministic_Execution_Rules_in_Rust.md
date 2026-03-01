# LV-SPEC-0601 — Deterministic Execution Rules in Rust
Version: 1.0  
Status: Draft (Engineering)  
Conformance Level: Core (Mandatory for consensus and deterministic-mode execution)  
Implements: LV-SPEC-0103, LV-SPEC-0202, LV-SPEC-0502, LV-SPEC-0600  
Language: Rust  

---

# 1. Purpose

This specification defines deterministic execution rules for Lite-Vision Rust implementations.

It formalizes:

- RNG seeding and usage policy
- Floating-point restrictions
- Canonical serialization rules
- Time and clock abstraction constraints
- Reproducible build guidance

Determinism is required for:

- Truth Plane state transitions
- Deterministic-mode job execution
- Replay validation
- Golden test reproducibility

Nondeterminism is forbidden in consensus logic.

---

# 2. Determinism Scope

Deterministic constraints apply to:

- `lv-core`
- `lv-truth`
- Deterministic paths in `lv-intel`
- Replay engine
- Snapshot hashing
- Partition migration hashing

Non-consensus crates may use nondeterminism if:

- It does not influence committed state.
- It does not influence output_hash in deterministic mode.

---

# 3. RNG Policy

---

## 3.1 Consensus RNG

Consensus crates MUST NOT use:

- `rand::thread_rng()`
- OS entropy sources
- Hardware RNG
- Unseeded RNG

If randomness required (e.g., leader election seed):

Randomness MUST be derived from:

H(previous_block_hash || domain_tag)

All randomness MUST be:

- Deterministic
- Derived from canonical inputs
- Domain-separated

---

## 3.2 Deterministic Job RNG

For deterministic-mode jobs:

RNG MUST be seeded with:

deterministic_seed (Hash32)

RNG type MUST be:

- Explicitly seeded
- Reproducible across platforms
- Stable across Rust versions

Recommended:

- ChaCha-based PRNG with fixed implementation version
- Or custom deterministic RNG in `lv-core`

RNG algorithm version MUST be pinned.

---

## 3.3 Prohibited RNG Usage

Prohibited in deterministic contexts:

- System time as seed
- Memory address-based randomness
- HashMap iteration order as entropy
- Floating-point noise

---

# 4. Floating-Point Policy

---

## 4.1 Consensus Logic

Floating-point arithmetic is FORBIDDEN in:

- State transition function δ
- Slashing logic
- Stake calculation
- Merkle tree computation
- Partition migration hashing

Use:

- Fixed-width integers
- Fixed-point arithmetic (if necessary)

---

## 4.2 Deterministic Job Mode

Floating-point MAY be used in deterministic-mode jobs IF:

- All hardware operations are reproducible
- No non-associative reductions without ordering
- No reliance on fused-multiply-add differences
- No architecture-specific intrinsics

If strict reproducibility required:

- Use software float emulation
- Or canonical reduction ordering

---

## 4.3 Serialization of Floats

If floats serialized:

- Use canonical encoding (IEEE-754)
- Normalize NaN representations
- Forbid signaling NaNs
- Forbid ±0 ambiguity

Float hashing MUST canonicalize representation.

---

# 5. Serialization Determinism

All serialized structures used in:

- Hashing
- Merkle roots
- ArtifactID
- Snapshot roots
- Replay bundles

MUST be:

- Canonically ordered
- Fully specified
- Versioned

---

## 5.1 Map Ordering

HashMap is FORBIDDEN in consensus code.

Use:

BTreeMap  
OrderedVec  

Iteration order MUST be deterministic.

---

## 5.2 Canonical Encoding Rules

- Field order fixed.
- No optional field reordering.
- No implicit default omission in hashed contexts.
- No platform-dependent struct padding.

All hashes computed over canonical byte representation.

---

# 6. Time and Clock Abstraction Rules

---

## 6.1 Consensus Time

Consensus MUST NOT rely on:

- System clock
- Wall-clock timestamps
- OS time APIs

Consensus time derived from:

- Block height
- Logical epoch
- View number

---

## 6.2 Deterministic Execution Time

Deterministic-mode jobs MUST NOT call:

- System clock
- Sleep/delay with wall time
- External timing APIs

If time needed:

Inject deterministic TimeProvider:

trait TimeProvider {
    fn now_logical(&self) -> u64;
}

Replay must inject same time sequence.

---

## 6.3 Non-Deterministic Mode

Soft execution mode MAY use wall-clock time, but:

- Output_hash must not be anchored in deterministic context.
- Replay not required.

---

# 7. Parallelism Rules

---

## 7.1 Consensus Code

Consensus crates MUST avoid:

- Data races
- Unordered parallel reduction
- Non-deterministic task scheduling affecting results

Parallelism allowed only if:

- Final result independent of scheduling
- Reduction order explicitly defined

---

## 7.2 Deterministic Job Mode

Parallel GPU execution allowed IF:

- Output reduction order fixed
- Deterministic kernels used
- No reliance on hardware scheduling quirks

---

# 8. Environment Isolation

Deterministic contexts MUST NOT read:

- Environment variables
- Filesystem state
- Network responses
- Process ID
- Thread ID
- Memory addresses

All external inputs must be injected explicitly.

---

# 9. Build Determinism

---

## 9.1 Dependency Pinning

All binaries MUST:

- Pin dependency versions
- Include Cargo.lock
- Avoid wildcard dependencies

---

## 9.2 Compiler Version Pinning

Use:

`rust-toolchain.toml`

Specifying exact toolchain.

CI MUST test:

- MSRV
- Latest stable

Release builds SHOULD use fixed toolchain.

---

## 9.3 Reproducible Builds

Guidelines:

- Avoid embedding timestamps in binaries.
- Avoid embedding build host path.
- Embed commit hash explicitly.
- Use deterministic linker flags (where possible).
- Use `--locked` builds.

Optional:

- Provide SHA256 checksum for release binaries.
- Provide Dockerfile for reproducible builds.

---

# 10. Hash Stability

Hash function MUST be:

- Fixed algorithm
- Versioned
- Domain-separated
- Independent of compiler version

Hash inputs MUST use canonical encoding.

---

# 11. Replay Determinism Requirements

Replay engine MUST:

- Disable network access
- Disable system clock access
- Disable thread RNG
- Use fixed scheduler (if multithreaded)
- Validate kernel hash before execution

Replay MUST produce identical output_hash.

---

# 12. Cross-Platform Guarantees

Consensus code MUST behave identically on:

- Linux
- macOS
- Windows (if supported)

Differences in:

- Endianness
- Floating-point rounding
- Atomic ordering

MUST NOT affect consensus.

All numeric encoding MUST use fixed endianness (little-endian recommended).

---

# 13. Testing Determinism

Determinism tests MUST include:

- Multiple replays on same machine
- Replay on different machines
- Replay across Rust minor versions (if supported)
- Randomized scheduling injection

Any divergence is a failure.

---

# 14. Unsafe Code Constraints

Unsafe code in deterministic paths MUST:

- Preserve invariants
- Avoid UB
- Avoid memory layout dependence
- Avoid pointer-based hashing

Unsafe MUST NOT introduce nondeterminism.

---

# 15. CI Determinism Gates

CI MUST:

- Run golden tests twice (same run)
- Run replay suite
- Compare outputs bit-for-bit
- Fail on mismatch
- Build with `--locked`

Optional:

- Cross-compile and compare replay hashes.

---

# 16. Compliance Requirements

To claim LV-SPEC-0601 compliance:

Implementation MUST:

- Forbid floating-point in consensus
- Use deterministic RNG in deterministic mode
- Canonicalize serialization
- Abstract time in deterministic paths
- Pin toolchain
- Avoid HashMap in consensus code
- Pass replay determinism tests
- Produce stable state_root across runs

Enterprise/Reference SHOULD provide reproducible binary builds.

---

# 17. Design Invariants

1. Consensus must be bitwise deterministic.
2. Deterministic jobs must replay identically.
3. RNG must always be seeded explicitly.
4. Time must be abstracted in deterministic contexts.
5. Hashes must be stable across platforms.
6. Serialization must be canonical.
7. Build process must be reproducible.
8. Unsafe must not introduce nondeterminism.

---

# 18. Summary

LV-SPEC-0601 defines deterministic execution rules in Rust:

- Strict RNG discipline
- Floating-point restrictions
- Canonical serialization
- Logical time abstraction
- Reproducible build policy
- Cross-platform stability

It ensures Lite-Vision remains:

Deterministic.
Auditable.
Replay-verifiable.
Consensus-safe.
Rust-disciplined by design.

---

End of LV-SPEC-0601