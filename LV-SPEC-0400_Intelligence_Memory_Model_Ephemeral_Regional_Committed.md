# LV-SPEC-0400 — Intelligence Memory Model (Ephemeral / Regional / Committed)
Version: 1.0  
Status: Draft (Engineering)  
Conformance Level: Core (Mandatory)  
Implements: LV-SPEC-0103, LV-SPEC-0107, LV-SPEC-0202, LV-SPEC-0205  
Language Target: Rust  

---

# 1. Purpose

This specification defines the Intelligence Memory Model of Lite-Vision.

It formalizes:

- Memory tiers
- Memory APIs
- Retention policies
- Committed memory anchoring into Truth Plane
- Provenance rules (what must be committed vs what must not)

Lite-Vision separates:

Execution memory (GPU-bound, transient)  
Shared intelligence memory (regional, mutable)  
Committed state memory (hash-anchored, immutable)

Truth Plane stores only hashes.

---

# 2. Memory Tier Overview

Lite-Vision defines three memory tiers:

Tier 0 — Ephemeral Memory  
Tier 1 — Regional Memory  
Tier 2 — Committed Memory  

Each tier has different:

- Lifetime
- Visibility
- Persistence
- Anchoring requirements
- Economic implications

---

# 3. Tier 0 — Ephemeral Memory

## 3.1 Definition

Ephemeral memory exists only during:

- Kernel execution
- Verification replay
- Routing decisions

Examples:

- GPU tensors
- Temporary inference activations
- Intermediate geometry buffers
- Temporary RPACK assembly buffers

---

## 3.2 Properties

Ephemeral memory:

- Is not persisted
- Is not shared across operators
- Is not hash-anchored
- Is destroyed after execution

Ephemeral memory MUST NOT be committed to Truth Plane.

---

## 3.3 Retention Rule

Ephemeral memory retention:

Lifetime ≤ execution scope

After kernel exit:

Memory MUST be released.

---

# 4. Tier 1 — Regional Memory

## 4.1 Definition

Regional memory is shared intelligence state within a region or partition.

Examples:

- Cached embeddings
- Shared model weights
- Context memory for agents
- Scene refinement state
- Adaptive routing metrics

Regional memory is mutable and local to a region.

---

## 4.2 Properties

Regional memory:

- Shared among operators in region
- Not globally committed
- May be eventually consistent
- May use CRDT or replication strategy

Regional memory MAY be pruned or replaced.

---

## 4.3 API

RegionalMemory API (conceptual):

```rust
trait RegionalMemory {
    fn get(key: MemoryKey) -> Option<MemoryValue>;
    fn put(key: MemoryKey, value: MemoryValue);
    fn delete(key: MemoryKey);
    fn snapshot() -> RegionalSnapshot;
}