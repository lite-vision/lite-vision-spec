# LV-SPEC-0000 — Corpus Index and Conformance Map
Version: 1.0  
Status: Draft (Engineering)  
Scope: Whole-system specification corpus  
Implementation Language: Rust (primary)

---

## 0. Purpose

This document defines the Lite-Vision specification corpus structure, taxonomy, dependency graph, and conformance levels.

It answers:

- What specs exist?
- What depends on what?
- What is required to claim Core / Enterprise / Reference conformance?
- How do implementations prove conformance?

Lite-Vision is a two-plane system:

- Truth Plane: CPU-secured BFT consensus and deterministic ledger/state machine
- Intelligence Plane: GPU/accelerator compute fabric with cryptoeconomic security
- Rendering: strictly client-side; network outputs RPACK (Render Packet)

---

## 1. Conformance Levels

### 1.1 Core Conformance (CORE)

A Core implementation MUST:

- Implement the Truth Plane BFT ledger with deterministic state transitions
- Implement the Intelligence Plane job model, routing, receipts, and disputes
- Produce RPACK outputs (no pixels computed on-network)
- Provide dual-plane partition and storage lifecycle controls (minimum viable)
- Provide basic observability and replay hooks sufficient for disputes

Core is sufficient to run a minimal Lite-Vision network.

### 1.2 Enterprise Conformance (ENT)

An Enterprise implementation MUST:

- Meet all Core requirements
- Provide production-grade ops/runbooks, upgrade strategy, and HA guidance
- Provide SDK/API stability and external integration contracts
- Provide security hardening, fuzzing, and conformance testing
- Provide deterministic execution policy for audit-critical paths
- Provide enhanced partitioning and elasticity policies

Enterprise is sufficient for production deployments.

### 1.3 Reference Conformance (REF)

A Reference implementation MUST:

- Meet all Enterprise requirements
- Implement the full conformance harness (golden vectors, fuzz, integration suites)
- Provide multi-client interop validation for RPACK/Scene IR
- Provide deterministic replay capture with forensic-grade audit extraction
- Provide optional advanced features where specified as REF-only (e.g., extended proof hooks)

Reference is sufficient to serve as the canonical compatibility target.

---

## 2. Normative Language

The terms **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, **MAY** are to be interpreted as in RFC 2119.

---

## 3. Corpus Taxonomy

The corpus is organized by functional domain and numeric ranges:

- 0000: Corpus governance / index
- 0100–0199: Truth Plane (BFT consensus + deterministic ledger)
- 0200–0299: Intelligence Plane (compute fabric + economic security)
- 0300–0399: RPACK + client-side rendering protocol
- 0400–0499: Dual-plane memory/storage lifecycle management
- 0500–0599: Networking, APIs, observability, operations
- 0600–0699: Rust implementation rules and deterministic execution constraints
- 0700–0799: Enterprise add-ons (elasticity, compliance, HA, equilibrium)

---

## 4. Spec Index (By ID)

### 4.1 Core Runtime (30 Specifications)

#### Corpus
- LV-SPEC-0000 — Corpus Index and Conformance Map

#### Truth Plane (BFT, CPU-Secured)
- LV-SPEC-0100 — Truth Plane Architecture Overview
- LV-SPEC-0101 — BFT Consensus Protocol Specification
- LV-SPEC-0102 — Validator Set, Staking, and Slashing
- LV-SPEC-0103 — Block, Transaction, and Deterministic State Transitions
- LV-SPEC-0104 — Cryptography and Key Management
- LV-SPEC-0105 — Governance and Parameter Control
- LV-SPEC-0106 — Truth Plane Storage, Pruning, and Archival
- LV-SPEC-0107 — Cross-Partition and Cross-Plane Commitments

#### Intelligence Plane (GPU Compute Fabric)
- LV-SPEC-0200 — Intelligence Plane Architecture Overview
- LV-SPEC-0201 — Operator Node Lifecycle and Registration
- LV-SPEC-0202 — Intelligence Kernel Interface (Rust ABI + Sandbox)
- LV-SPEC-0203 — Job Model and Execution Semantics
- LV-SPEC-0204 — Sparse Routing and Activation Protocol
- LV-SPEC-0205 — Receipts, Metering, and Attestation
- LV-SPEC-0206 — Verification and Redundancy Model
- LV-SPEC-0207 — Challenge and Dispute Protocol

#### RPACK & Client-Side Rendering
- LV-SPEC-0300 — Render Offload Model
- LV-SPEC-0301 — RPACK Binary Container Format
- LV-SPEC-0302 — Scene IR Schema
- LV-SPEC-0303 — Asset Referencing and Fetch Protocol
- LV-SPEC-0304 — RPACK Delta Update Semantics

#### Dual-Plane Memory & Storage
- LV-SPEC-0400 — Intelligence Memory Model (Ephemeral/Regional/Committed)
- LV-SPEC-0401 — CRDT Merge and Regional Memory Semantics
- LV-SPEC-0402 — Partition Management (Add/Delete/Rebalance)
- LV-SPEC-0403 — Storage Tiering and Compaction
- LV-SPEC-0404 — Artifact Store and Content Addressing

#### Networking, Ops, Rust
- LV-SPEC-0500 — P2P Networking and Message Protocol
- LV-SPEC-0502 — Observability and Deterministic Replay
- LV-SPEC-0600 — Rust Workspace Architecture and Crate Layout

---

### 4.2 Enterprise Add-Ons (10 Specifications)

- LV-SPEC-0501 — SDK and External API Surface (Enterprise)
- LV-SPEC-0503 — Security Hardening and Threat Mitigation
- LV-SPEC-0504 — Deployment Topologies and Operational Runbook
- LV-SPEC-0505 — Conformance Test Harness and Fuzzing Suite
- LV-SPEC-0601 — Deterministic Execution Policy (Rust-Level)
- LV-SPEC-0602 — Stable Wire Formats and Backward Compatibility
- LV-SPEC-0700 — Advanced Partitioning and Elastic Scaling
- LV-SPEC-0701 — Economic Equilibrium and Incentive Modeling
- LV-SPEC-0702 — Enterprise Compliance and Audit Framework
- LV-SPEC-0703 — High-Availability and Fault Tolerance Enhancements

---

## 5. Dependency Graph (High-Level)

This section defines normative dependencies between specs. A spec that depends on another spec MUST NOT contradict it.

### 5.1 Truth Plane Dependencies

- LV-SPEC-0100 depends on: none (top-level)
- LV-SPEC-0101 depends on: LV-SPEC-0100
- LV-SPEC-0102 depends on: LV-SPEC-0101
- LV-SPEC-0103 depends on: LV-SPEC-0101, LV-SPEC-0104
- LV-SPEC-0104 depends on: none (cryptographic primitives)
- LV-SPEC-0105 depends on: LV-SPEC-0103
- LV-SPEC-0106 depends on: LV-SPEC-0103
- LV-SPEC-0107 depends on: LV-SPEC-0103, LV-SPEC-0404 (commitments to artifacts)

### 5.2 Intelligence Plane Dependencies

- LV-SPEC-0200 depends on: LV-SPEC-0100 (coupling boundary)
- LV-SPEC-0201 depends on: LV-SPEC-0102, LV-SPEC-0104
- LV-SPEC-0202 depends on: LV-SPEC-0104, LV-SPEC-0600 (Rust layout constraints)
- LV-SPEC-0203 depends on: LV-SPEC-0201, LV-SPEC-0202, LV-SPEC-0107
- LV-SPEC-0204 depends on: LV-SPEC-0203, LV-SPEC-0206
- LV-SPEC-0205 depends on: LV-SPEC-0104, LV-SPEC-0202, LV-SPEC-0404
- LV-SPEC-0206 depends on: LV-SPEC-0203, LV-SPEC-0205
- LV-SPEC-0207 depends on: LV-SPEC-0102, LV-SPEC-0107, LV-SPEC-0205, LV-SPEC-0206

### 5.3 RPACK Dependencies

- LV-SPEC-0300 depends on: LV-SPEC-0203 (job outputs), LV-SPEC-0107 (commitments)
- LV-SPEC-0301 depends on: LV-SPEC-0300, LV-SPEC-0602 (wire formats)
- LV-SPEC-0302 depends on: LV-SPEC-0300
- LV-SPEC-0303 depends on: LV-SPEC-0404 (content addressing), LV-SPEC-0500 (transport)
- LV-SPEC-0304 depends on: LV-SPEC-0301, LV-SPEC-0302

### 5.4 Storage Dependencies

- LV-SPEC-0400 depends on: LV-SPEC-0200
- LV-SPEC-0401 depends on: LV-SPEC-0400
- LV-SPEC-0402 depends on: LV-SPEC-0106 (truth storage rules), LV-SPEC-0400, LV-SPEC-0500
- LV-SPEC-0403 depends on: LV-SPEC-0106, LV-SPEC-0400
- LV-SPEC-0404 depends on: LV-SPEC-0104, LV-SPEC-0500

### 5.5 Networking / Ops / Rust Dependencies

- LV-SPEC-0500 depends on: LV-SPEC-0104
- LV-SPEC-0502 depends on: LV-SPEC-0500, LV-SPEC-0601 (determinism rules for replay)
- LV-SPEC-0600 depends on: none
- LV-SPEC-0501 depends on: LV-SPEC-0500, LV-SPEC-0602
- LV-SPEC-0503 depends on: LV-SPEC-0104, LV-SPEC-0205, LV-SPEC-0404
- LV-SPEC-0504 depends on: LV-SPEC-0101, LV-SPEC-0500
- LV-SPEC-0505 depends on: all CORE specs (conformance harness spans whole system)
- LV-SPEC-0601 depends on: LV-SPEC-0600
- LV-SPEC-0602 depends on: LV-SPEC-0600
- LV-SPEC-0700 depends on: LV-SPEC-0402, LV-SPEC-0500
- LV-SPEC-0701 depends on: LV-SPEC-0102, LV-SPEC-0206
- LV-SPEC-0702 depends on: LV-SPEC-0106, LV-SPEC-0502
- LV-SPEC-0703 depends on: LV-SPEC-0101, LV-SPEC-0700

---

## 6. Cross-Reference Rules

### 6.1 Compatibility Rule
A newer version of any spec MUST be backward compatible at the wire level unless LV-SPEC-0602 explicitly defines a migration path.

### 6.2 Canonical Encoding Rule
All data that is hashed for commitments MUST be encoded canonically as defined by LV-SPEC-0602.

### 6.3 Determinism Rule
All code that affects Truth Plane state MUST be deterministic. Determinism requirements are expanded in LV-SPEC-0601.

### 6.4 “No Pixels On-Network” Rule
Any implementation claiming Lite-Vision conformance MUST NOT compute final pixel outputs as a network service. RPACK outputs are required by LV-SPEC-0300.

### 6.5 Dual-Plane Lifecycle Rule
Both planes MUST support partitioning, addition/removal, and governed deletion/pruning in conformance with LV-SPEC-0106 and LV-SPEC-0402.

---

## 7. Mandatory vs Optional Features

### 7.1 Mandatory (Core)
- BFT consensus, deterministic ledger transitions
- Operator registration + capability registry
- Job execution semantics + sparse routing
- Receipts + verification + disputes
- RPACK container + Scene IR + asset references
- Artifact store with content addressing
- Partition management (add/delete/rebalance)
- Basic observability + replay for disputes
- Rust workspace architecture alignment

### 7.2 Optional (Core-Optional)
- Deterministic execution mode for intelligence (seeded) (recommended)
- TEEs / remote attestation hooks
- Advanced routing learning (beyond deterministic scoring)
- Multi-client renderer conformance tiers

### 7.3 Enterprise-Required (ENT)
- SDK and stable APIs
- Security hardening guide
- Operational runbook + HA topologies
- Conformance harness + fuzzing suite
- Deterministic execution policy (Rust-level)
- Stable wire formats and schema evolution
- Advanced elasticity and partition automation
- Compliance and audit framework

---

## 8. Conformance Evidence

To claim conformance, an implementation MUST provide:

### 8.1 Core Evidence
- Protocol version manifest (spec versions implemented)
- Golden-vector proof for Truth Plane δ transitions
- End-to-end job execution with receipts and settlement
- Dispute flow demonstration with slashing capability
- RPACK generation and client-side rendering demo
- Partition add/remove operation demo on both planes

### 8.2 Enterprise Evidence
- CI logs showing conformance harness passing
- Fuzzing target coverage summary (parsers and wire formats)
- Disaster recovery and upgrade rehearsal reports
- Deterministic replay capture and verification procedure
- Audit export demonstration

---

## 9. Versioning Policy for the Corpus

- Each spec has: Version, Status, and Change Log.
- Major version changes MAY break compatibility but MUST include a migration plan.
- Minor version changes MUST be backward compatible at the wire level.
- Patch changes MUST be editorial or clarifying only.

---

## 10. End of Document

This concludes LV-SPEC-0000.
