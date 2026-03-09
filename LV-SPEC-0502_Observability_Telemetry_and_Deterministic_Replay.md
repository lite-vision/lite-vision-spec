# LV-SPEC-0502 — Observability, Telemetry, and Deterministic Replay
Version: 1.0  
Status: Draft (Engineering)  
Conformance Level: Core (Mandatory for validators/operators; replay mandatory for deterministic jobs)  
Implements: LV-SPEC-0202, LV-SPEC-0203, LV-SPEC-0205, LV-SPEC-0206, LV-SPEC-0400, LV-SPEC-0500  
Language Target: Rust  

---

# 1. Purpose

This specification defines:

- Metrics schema
- Distributed tracing model
- Structured logging format
- Deterministic replay capture format
- Redaction and privacy rules

Lite-Vision must provide:

Operational visibility  
Fraud investigation capability  
Deterministic replay (when required)  
Privacy-preserving telemetry  

Observability must never:

- Alter consensus behavior
- Introduce nondeterminism
- Leak sensitive data

---

# 2. Observability Scope

Applies to:

- Validators
- Operators
- Routers
- Artifact nodes
- Regional memory replicas

Does NOT apply to:

- Client-side rendering (separate domain)

---

# 3. Metrics Schema

Metrics MUST be:

- Structured
- Namespaced
- Typed
- Deterministic in definition

---

## 3.1 Metric Naming Convention

lv.<plane>.<component>.<metric_name>

Examples:

lv.truth.consensus.block_time  
lv.truth.mempool.size  
lv.intel.operator.gpu_cycles  
lv.intel.verification.failure_rate  
lv.storage.artifact.replication_factor  

---

## 3.2 Metric Types

Supported metric types:

- Counter (monotonic)
- Gauge (instant value)
- Histogram
- Summary
- Boolean flag

Floating-point MAY be used in metrics but MUST NOT influence consensus logic.

---

## 3.3 Required Core Metrics

Validators MUST expose:

- block_height
- block_time_ms
- vote_latency_ms
- peer_count
- mempool_size
- dispute_count

Operators MUST expose:

- active_jobs
- gpu_cycles_used
- cpu_cycles_used
- receipt_failures
- verification_pass_rate

Artifact nodes MUST expose:

- artifact_count
- replication_factor
- storage_usage_bytes
- gc_events

---

# 4. Telemetry Transport

Metrics MAY be exported via:

- Prometheus endpoint
- OpenTelemetry
- gRPC streaming
- Structured log scraping

Telemetry export MUST NOT:

- Block consensus thread
- Exceed rate limits
- Leak secret keys

---

# 5. Structured Logs

Logs MUST be:

- JSON structured
- Timestamped (logical time preferred)
- Include node_id
- Include correlation_id

Log schema:

LogEntry {
  level: enum { DEBUG, INFO, WARN, ERROR },
  component: String,
  event: String,
  block_height: Option<u64>,
  job_id: Option<Hash32>,
  correlation_id: Option<Hash32>,
  message: String,
  metadata: Map<String, Value>,
}

Logs MUST be canonical and machine-readable.

---

# 6. Tracing Model

Tracing MUST support:

- Job lifecycle tracing
- Receipt submission tracing
- Dispute tracing
- Artifact replication tracing

Each trace MUST include:

TraceContext {
  trace_id: Hash32,
  span_id: u64,
  parent_span_id: Option<u64>,
}

Trace IDs MUST propagate across:

- Job assignment
- Operator execution
- Verification
- Dispute resolution

---

# 7. Deterministic Replay Overview

Deterministic replay required when:

execution_mode == Deterministic

Replay MUST allow:

- Full re-execution
- Output hash comparison
- Resource metric verification

Replay capture MUST be:

- Canonical
- Complete
- Hash-verifiable

---

# 8. Replay Capture Format

ReplayBundle {
  version: u16,
  job_id: Hash32,
  kernel_id: Hash32,
  kernel_version: (u16, u16, u16),
  deterministic_seed: Hash32,
  input_hash: Hash32,
  input_bytes: bytes,
  initial_state_hash: Option<Hash32>,
  execution_context: bytes,
  expected_output_hash: Hash32,
  resource_caps: ResourceLimits,
  capture_signature: Signature
}

ReplayBundleHash = H(canonical_bundle)

ReplayBundle MUST be content-addressable.

---

# 9. Replay Procedure

Replay engine MUST:

1. Load kernel by kernel_id.
2. Verify kernel hash.
3. Load input_bytes.
4. Apply deterministic_seed.
5. Execute within resource_caps.
6. Compute output_hash.
7. Compare to expected_output_hash.

Mismatch → fraud candidate.

Replay MUST:

- Avoid external network calls.
- Avoid system clock.
- Avoid nondeterministic RNG.
- Avoid floating-point instability (if deterministic profile).

---

# 10. Replay Storage Policy

ReplayBundle MUST be stored:

- At least until verification_window expires.
- Longer for deterministic-critical jobs.

Replay data MAY be:

- Encrypted at rest.
- Pruned after safe retention window.

Replay MUST NOT store:

- Raw private user data unless explicitly allowed.
- Sensitive regional memory not included in job context.

---

# 11. Privacy and Redaction

Observability must respect privacy.

---

## 11.1 Sensitive Data Categories

Sensitive data includes:

- Private prompts
- API keys
- Encryption keys
- Personal user identifiers
- Proprietary model weights

Sensitive data MUST be redacted.

---

## 11.2 Redaction Rules

RedactedField {
  field_name,
  redaction_type: enum { HASHED, REMOVED, MASKED },
}

Logs MUST NOT expose:

- Raw input prompts
- Deterministic seed (if confidential job)
- Encryption keys

ReplayBundle MAY encrypt sensitive payload.

---

# 12. Determinism Constraints in Logging

Logging MUST NOT:

- Introduce side effects.
- Alter execution timing in deterministic mode.
- Influence state.

Log emission must be:

- Non-blocking
- Asynchronous
- Buffered

---

# 13. Audit Trail Requirements

Validators MUST maintain audit trail for:

- Slashing events
- Fraud proofs
- Governance actions
- Partition migration

Audit trail MUST:

- Be immutable
- Be hash-verifiable
- Be retained per governance policy

---

# 14. Performance Telemetry Isolation

Metrics MUST be isolated from:

- Consensus-critical paths
- GPU execution loops

Telemetry collection MUST:

- Use separate threads
- Avoid locking hot execution paths

---

# 15. Sampling Policies

High-frequency metrics MAY use sampling.

Sampling MUST NOT:

- Affect fraud detection
- Affect dispute logic
- Affect deterministic replay

---

# 16. Time Semantics

Metrics and logs SHOULD include:

- Logical block height
- Monotonic clock
- Avoid reliance on wall-clock time for replay

Wall-clock MAY be included for observability but not for consensus logic.

---

# 17. Rust Implementation Requirements

Implementation MUST:

- Use structured logging crate
- Use canonical JSON for logs
- Use fixed-width integers
- Separate replay capture from execution thread
- Validate replay bundle hash
- Enforce redaction policies
- Avoid floating-point influence on consensus paths

Replay engine MUST be deterministic and sandboxed.

---

# 18. Compliance Requirements

To claim LV-SPEC-0502 compliance:

Implementation MUST:

- Expose required metrics
- Support structured logs
- Implement trace context propagation
- Support ReplayBundle capture
- Support deterministic replay
- Enforce redaction rules
- Preserve replay data until safe window expires

---

# 19. Design Invariants

1. Observability must not alter consensus behavior.
2. Replay must be deterministic and verifiable.
3. Sensitive data must be redacted.
4. Replay must not require external network access.
5. Metrics must not influence economic logic.
6. Logs must be structured and machine-readable.
7. Replay bundles must be content-addressed.

---

# 20. Summary

LV-SPEC-0502 defines:

- Observability architecture
- Metrics and tracing schema
- Structured logging
- Deterministic replay format
- Redaction and privacy safeguards

It enables:

Operational visibility.
Fraud investigation.
Deterministic reproducibility.
Privacy protection.
Auditable governance.

Lite-Vision remains:

CPU-secured.
GPU-powered.
Hash-anchored.
Replay-verifiable.
Privacy-aware.

---

End of LV-SPEC-0502