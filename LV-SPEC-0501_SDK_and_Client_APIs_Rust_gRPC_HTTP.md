# LV-SPEC-0501 — SDK and Client APIs (Rust + gRPC / HTTP)
Version: 1.0  
Status: Draft (Engineering)  
Conformance Level: Core (Mandatory for public-facing nodes)  
Implements: LV-SPEC-0203, LV-SPEC-0205, LV-SPEC-0207, LV-SPEC-0301, LV-SPEC-0400, LV-SPEC-0404, LV-SPEC-0500  
Language Target: Rust  

---

# 1. Purpose

This specification defines the Lite-Vision SDK and public API surface.

It formalizes:

- Rust crate architecture
- gRPC service definitions
- HTTP/REST compatibility layer
- API surface for:
  - Jobs
  - Receipts
  - Memory
  - Artifacts
  - RPACK
- Authentication mechanisms
- Rate limiting
- Pagination

APIs must:

- Be deterministic in serialization
- Respect consensus boundaries
- Avoid exposing private regional state improperly
- Remain versioned and backward-compatible

---

# 2. SDK Architecture

Lite-Vision SDK consists of the following Rust crates:

---

## 2.1 Crate: `lv-client`

Purpose:

- Submit jobs
- Query receipts
- Fetch artifacts
- Apply RPACK-Δ
- Verify output hashes

Core modules:

- jobs
- artifacts
- rpack
- receipts
- auth
- pagination

---

## 2.2 Crate: `lv-operator`

Purpose:

- Register operator
- Receive job assignments
- Submit receipts
- Participate in verification
- Manage regional memory

Core modules:

- registration
- execution
- receipts
- verification
- regional_memory
- metering

---

## 2.3 Crate: `lv-validator`

Purpose:

- Participate in BFT consensus
- Validate blocks
- Enforce slashing
- Process disputes
- Anchor receipts

Core modules:

- consensus
- staking
- slashing
- dispute
- state
- networking

---

## 2.4 Crate: `lv-tools`

Purpose:

- CLI utilities
- Snapshot management
- Artifact pinning
- Conformance testing
- Diagnostics

---

# 3. API Transport Model

All APIs MUST be exposed via:

- gRPC (primary)
- HTTP/JSON (compatibility layer)

gRPC is canonical.
HTTP MUST map 1:1 to gRPC methods.

Serialization MUST be canonical and deterministic.

---

# 4. Authentication Model

Authentication supported via:

- Public-key signature headers
- JWT (optional)
- API keys (enterprise)
- Mutual TLS (optional)

Every authenticated request MUST include:

X-LV-Signature  
X-LV-PublicKey  
X-LV-Nonce  

Signature covers:

method || path || body || nonce

Nonce MUST prevent replay.

---

# 5. Core API Surfaces

---

# 6. Job APIs

## 6.1 Submit Job

gRPC:

rpc SubmitJob(JobRequest) returns (JobResponse);

HTTP:

POST /v1/jobs

JobRequest:
- kernel_id
- input_hash
- execution_mode
- budget
- deadline
- qos_class
- verification_policy

JobResponse:
- job_id
- escrow_status
- submission_block_height

---

## 6.2 Get Job Status

GET /v1/jobs/{job_id}

Returns:

- status
- assigned_operator
- receipt_hash (if available)
- verification_status

---

## 6.3 Cancel Job

POST /v1/jobs/{job_id}/cancel

Requires:

- Client signature
- Job not finalized

---

# 7. Receipt APIs

## 7.1 Submit Receipt (Operator)

rpc SubmitReceipt(Receipt) returns (ReceiptAck);

HTTP:

POST /v1/receipts

Requires:

- Operator signature
- Valid execution_nonce

---

## 7.2 Get Receipt

GET /v1/receipts/{receipt_hash}

Returns:

- receipt fields
- verification status
- dispute status

---

# 8. Artifact APIs

## 8.1 Upload Artifact

POST /v1/artifacts

Body:

- Artifact bytes

Response:

- artifact_id
- verification_status

---

## 8.2 Fetch Artifact

GET /v1/artifacts/{artifact_id}

Supports:

- Range requests
- Streaming
- Chunk verification

---

## 8.3 Pin Artifact

POST /v1/artifacts/{artifact_id}/pin

---

## 8.4 Unpin Artifact

POST /v1/artifacts/{artifact_id}/unpin

---

# 9. RPACK APIs

## 9.1 Fetch RPACK

GET /v1/rpack/{artifact_id}

Returns:

- RPACK binary
- Metadata
- Renderer profile hint

---

## 9.2 Apply RPACK-Δ

POST /v1/rpack/{artifact_id}/delta

Body:

- Delta bytes

Returns:

- new_rpack_hash

---

# 10. Memory APIs

---

## 10.1 Regional Memory (Operator)

rpc GetMemory(KeyRequest) returns (ValueResponse);
rpc PutMemory(KeyValueRequest) returns (Ack);

HTTP:

GET /v1/memory/{namespace}/{key}
POST /v1/memory

Regional memory access may require operator authentication.

---

## 10.2 Snapshot Regional Memory

POST /v1/memory/snapshot

Returns:

- snapshot_hash
- optional anchor_block_height

---

# 11. Dispute APIs

---

## 11.1 Submit Fraud Proof

POST /v1/disputes

Body:

- FraudProof structure
- Challenger bond reference

---

## 11.2 Get Dispute Status

GET /v1/disputes/{job_id}

Returns:

- dispute_phase
- escrow_status
- slashing_outcome

---

# 12. Pagination

List endpoints MUST support:

Query parameters:

- limit
- cursor
- sort_order

Response:

{
  items: [...],
  next_cursor: optional,
}

Cursor MUST be opaque and deterministic.

Pagination MUST NOT expose unstable ordering.

---

# 13. Rate Limits

Nodes MUST enforce:

- Per-IP request cap
- Per-PeerID request cap
- Per-endpoint cap
- Burst limits
- Token bucket model

Rate limit headers:

X-LV-RateLimit-Limit  
X-LV-RateLimit-Remaining  
X-LV-RateLimit-Reset  

Rate limits MUST prioritize:

- Consensus
- Fraud proofs
- Receipt submissions

---

# 14. Error Handling

All APIs MUST return structured error:

ErrorResponse {
  code: enum,
  message: string,
  details: optional,
}

Common codes:

- INVALID_SIGNATURE
- NOT_FOUND
- RATE_LIMITED
- INVALID_NONCE
- DISPUTE_ACTIVE
- INSUFFICIENT_STAKE
- INVALID_ARTIFACT

Error responses MUST be deterministic.

---

# 15. Streaming APIs

Streaming endpoints:

- Artifact streaming
- Block streaming
- Receipt subscription
- PubSub topic subscription

gRPC streaming preferred.

HTTP SSE MAY be supported.

Streaming MUST enforce:

- Backpressure
- Rate limits
- Authentication

---

# 16. Versioning

All endpoints MUST include version prefix:

/v1/

Major version changes may break compatibility.

Minor version changes MUST remain backward-compatible.

SDK crates MUST declare supported protocol versions.

---

# 17. Rust Implementation Requirements

Implementation MUST:

- Use canonical serialization
- Use fixed-width integers
- Validate signature before processing
- Enforce nonce replay protection
- Implement rate limiting
- Support async streaming
- Avoid floating-point arithmetic in consensus-bound logic

gRPC definitions MUST be canonical.

---

# 18. Security Requirements

APIs MUST:

- Reject malformed payloads early
- Validate max payload size
- Authenticate before executing state change
- Enforce per-role permissions
- Log suspicious activity

Sensitive endpoints MUST require signature.

---

# 19. Compliance Requirements

To claim LV-SPEC-0501 compliance:

Implementation MUST:

- Provide Rust SDK crates
- Expose gRPC endpoints
- Provide HTTP compatibility
- Support authentication and nonce protection
- Enforce rate limits
- Support pagination
- Support streaming
- Preserve deterministic serialization

---

# 20. Design Invariants

1. SDK must not bypass Truth Plane rules.
2. APIs must be versioned.
3. Authentication must prevent replay.
4. Rate limits must protect consensus traffic.
5. Artifact integrity verified by hash.
6. Pagination must be deterministic.
7. Network must not expose raw regional memory across partitions without policy.

---

# 21. Summary

LV-SPEC-0501 defines the SDK and public API layer:

- Rust crate architecture
- gRPC/HTTP APIs
- Job and receipt management
- Artifact and RPACK access
- Memory interaction
- Dispute submission
- Authentication and rate limiting
- Pagination and streaming

It enables:

Developer-friendly integration.
Secure operator interaction.
Validator coordination.
Client artifact retrieval.
Scalable distributed intelligence.

Lite-Vision remains:

CPU-secured.
GPU-powered.
Hash-anchored.
Client-rendered.
API-disciplined.

---

End of LV-SPEC-0501