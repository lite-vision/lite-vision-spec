# LV-SPEC-0205 — Receipts, Metering, and Attestation
Version: 1.0  
Status: Draft (Engineering)  
Conformance Level: Core (Mandatory; TEE optional)  
Implements: LV-SPEC-0103, LV-SPEC-0107, LV-SPEC-0202, LV-SPEC-0203  
Language Target: Rust  

---

# 1. Purpose

This specification defines:

- The canonical Receipt schema
- Resource metering rules (GPU, VRAM, CPU, bandwidth)
- Signed claims and validation flow
- Optional Trusted Execution Environment (TEE) / remote attestation hooks

Receipts are the cryptographic accountability mechanism of the Intelligence Plane.

Truth Plane anchors hashes.
It does not validate GPU execution.
It validates signatures and structure.

---

# 2. Receipt Overview

Each successful (or failed) job execution produces a Receipt.

Receipt binds:

- Input
- Kernel identity
- Execution context
- Output
- Resource usage
- Operator identity

ReceiptHash = H(canonical_receipt_struct)

Truth Plane anchors:

output_hash
resource_hash
receipt_hash

---

# 3. Receipt Schema

Receipt {
  version: u32,
  job_id: Hash32,
  operator_id: Hash32,
  kernel_id: Hash32,
  kernel_version: (u16, u16, u16),
  input_hash: Hash32,
  deterministic_seed: Option<Hash32>,
  execution_nonce: u64,
  output_hash: Hash32,
  resource_hash: Hash32,
  execution_mode: enum { Soft, Deterministic },
  start_block_height: u64,
  end_block_height: u64,
  signature: Signature
}

All fields MUST be canonically serialized.

---

# 4. Input Binding

input_hash MUST equal:

H(canonical_input)

If deterministic mode:

deterministic_seed MUST be included and hashed into input binding:

bound_input_hash = H(input_hash || deterministic_seed)

Replay verification MUST use bound_input_hash.

---

# 5. Kernel Binding

kernel_id MUST match:

H(kernel_binary || kernel_metadata)

kernel_version MUST correspond to registry entry.

Receipt referencing unknown kernel MUST be rejected.

---

# 6. Output Binding

output_hash MUST equal:

H(canonical_output)

Where canonical_output is:

- RPACK artifact OR
- Intermediate artifact

Truth Plane anchors output_hash only.

Artifact bytes remain off-plane.

---

# 7. Resource Metering

## 7.1 Metering Structure

ResourceMetrics {
  cpu_cycles_used: u64,
  gpu_cycles_used: u64,
  vram_bytes_peak: u64,
  memory_bytes_peak: u64,
  bandwidth_bytes_sent: u64,
  bandwidth_bytes_received: u64,
  wall_time_ms: u64,
  sandbox_exit_code: u32
}

resource_hash = H(canonical(ResourceMetrics))

Receipt MUST include resource_hash.

---

# 7.2 Metering Rules

Metering MUST:

- Start at kernel execution start
- Stop at kernel completion or abort
- Include retries separately
- Exclude routing overhead

Resource counters MUST be:

- Monotonic
- Non-negative
- Deterministic where possible

---

# 7.3 GPU Time Accounting

gpu_cycles_used MAY represent:

- Kernel execution cycles
- CUDA stream time
- Device-level profiling time

Exact implementation defined by operator.

Truth Plane does NOT verify GPU cycles directly.

Verification MAY compare claimed vs recomputed metrics.

---

# 7.4 VRAM Accounting

vram_bytes_peak MUST represent:

- Maximum allocated VRAM during execution

Memory reclamation MUST NOT reduce peak metric.

---

# 8. Signed Claims

Receipt MUST be signed:

signature = Sign(operator_private_key, H(receipt_without_signature))

Signature MUST use domain separation:

DOMAIN_RECEIPT = "LV-RECEIPT-V1"

Invalid signature → receipt rejected.

---

# 9. Receipt Validation (Truth Plane)

Truth Plane MUST verify:

1. Operator registered and Active
2. Signature valid
3. job_id exists
4. execution_nonce unused
5. kernel_id allowed for operator
6. Escrow exists

Truth Plane MUST NOT:

- Recompute output
- Validate RPACK structure
- Validate GPU metrics

---

# 10. Verification and Challenge

If verification triggered:

Verifier MUST:

1. Retrieve kernel binary.
2. Retrieve input + seed.
3. Re-execute under constraints.
4. Compare output_hash.
5. Optionally compare resource_hash.

If mismatch:

FraudProof {
  receipt_hash,
  expected_output_hash,
  recomputed_output_hash,
  evidence
}

Submitted to Truth Plane for slashing.

---

# 11. Partial Receipts

If partial completion allowed:

Receipt MUST include:

partial_flag: bool
completion_percentage: u8

Partial receipts MUST:

- Not finalize escrow
- Clearly distinguish from final receipts

---

# 12. Retry Receipts

If job retried:

execution_nonce MUST increment.

Only one final receipt may settle escrow.

Multiple receipts for same job_id MUST be:

- Tracked
- Deduplicated
- Bound by nonce

---

# 13. Attestation (Optional)

TEE/remote attestation is optional and non-required.

---

## 13.1 Attestation Structure

Attestation {
  attestation_type: enum { SGX, SEV, TPM, Other },
  enclave_measurement: Hash32,
  firmware_version: String,
  report_data: bytes,
  signature: bytes
}

Attestation hash:

attestation_hash = H(canonical(Attestation))

Receipt MAY include:

attestation_hash

---

## 13.2 Attestation Validation

If attestation provided:

Verifier MAY:

- Validate vendor signature
- Validate enclave measurement
- Validate firmware version

Truth Plane does NOT enforce TEE requirement in Core.

Enterprise mode MAY require attestation for specific jobs.

---

# 14. Remote Attestation Hooks

Kernel sandbox MAY expose:

fn get_attestation() -> Option<Attestation>

If enabled:

- Attestation included in receipt.
- Bound to execution_nonce.

Attestation MUST be bound to:

H(job_id || execution_nonce || output_hash)

Prevents attestation replay.

---

# 15. Anti-Replay Constraints

Receipt MUST be rejected if:

- Same (job_id, operator_id, execution_nonce) seen before
- Signature reused for different receipt body
- Kernel version mismatch

Truth Plane maintains:

ConsumedReceiptRegistry

---

# 16. Deterministic Mode Guarantees

For deterministic jobs:

Re-execution MUST produce identical:

- output_hash
- resource_hash (within deterministic constraints)

Non-deterministic mode does not guarantee identical metrics.

---

# 17. Storage and Pruning

Receipts MUST:

- Be included in block body
- Be part of receipt_root Merkle tree
- Remain provable after pruning

Receipt bytes MAY be pruned after retention window, provided:

- receipt_hash preserved
- Inclusion proof preserved

---

# 18. Rust Implementation Requirements

Implementation MUST:

- Canonically serialize Receipt
- Hash and sign receipt deterministically
- Enforce resource cap boundaries
- Prevent nonce reuse
- Enforce domain separation
- Support optional attestation hooks

Floating-point metrics MUST NOT affect deterministic replay.

---

# 19. Compliance Requirements

To claim LV-SPEC-0205 compliance:

Implementation MUST:

- Implement Receipt schema
- Sign receipts correctly
- Anchor output_hash and resource_hash
- Enforce anti-replay rules
- Support deterministic mode replay
- Support optional attestation field (even if unused)

TEE enforcement is optional in Core.

---

# 20. Design Invariants

1. Every execution must produce a signed receipt.
2. Receipt must bind input, kernel, and output.
3. Resource claims must be hash-committed.
4. Nonce reuse is forbidden.
5. Deterministic mode must be replayable.
6. Truth Plane never executes GPU validation.

---

# 21. Summary

LV-SPEC-0205 defines:

- The formal receipt contract
- Resource metering and claims
- Fraud detection hooks
- Optional TEE integration
- Anti-replay safeguards

It provides the accountability layer that allows:

GPU-powered cognition
CPU-secured settlement
Economic enforcement
Scalable asynchronous execution

Without collapsing execution into consensus.

---

End of LV-SPEC-0205