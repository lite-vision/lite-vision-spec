# LV-SPEC-0303 — Asset Referencing, Caching, and Fetch Protocol
Version: 1.0  
Status: Draft (Engineering)  
Conformance Level: Core (Mandatory)  
Implements: LV-SPEC-0107, LV-SPEC-0300, LV-SPEC-0301, LV-SPEC-0302  
Language Target: Rust  

---

# 1. Purpose

This specification defines:

- Content-addressed Asset IDs
- Asset referencing rules inside RPACK
- Cache hierarchy model
- Integrity verification rules
- Offline mode behavior
- Fallback synthesis semantics

Assets are external to the Truth Plane.

Truth Plane anchors only:

output_hash (RPACK hash)

Assets are:

- Content-addressed
- Hash-verified
- Client-fetched
- Never consensus-validated byte-by-byte

---

# 2. Asset Identity Model

## 2.1 AssetID Definition

AssetID = H(canonical_asset_bytes)

Hash function MUST match LV-SPEC-0104.

AssetID MUST:

- Be 32 bytes
- Be immutable
- Uniquely identify content

AssetID is globally unique by hash.

---

## 2.2 Asset Descriptor

AssetDescriptor {
  asset_id: Hash32,
  asset_type: enum,
  mime_type: String,
  size_bytes: u64,
  compression: Option<u16>,
  encryption: Option<u16>,
}

AssetDescriptor MAY appear in:

- Scene IR metadata
- Dedicated RPACK chunk
- Asset registry extension

---

# 3. Asset Types

Asset types include:

- Geometry
- Texture
- Shader
- Audio
- Animation
- Material preset
- Procedural definition
- Extension-specific

Assets MUST NOT include:

- Framebuffers
- Device memory snapshots

---

# 4. Asset Referencing Rules

Within Scene IR:

geometry_ref: Hash32
texture_refs: Vec<Hash32>
shader_ref: Hash32

References MUST:

- Use AssetID only
- Not embed raw URL
- Not embed unverified content

Optional fetch hints MAY include:

FetchHint {
  preferred_transport: enum,
  region_hint: Option<u16>,
  mirror_hint: Option<Hash32>,
}

Hints are advisory only.

---

# 5. Fetch Protocol

Asset fetch is client-side responsibility.

Supported fetch mechanisms MAY include:

- P2P content-addressed protocol
- HTTP(S) gateway
- Operator-hosted storage
- Decentralized storage networks
- Local cache

Truth Plane does NOT enforce fetch method.

---

## 5.1 Fetch Flow

1. Client reads AssetID.
2. Client queries cache.
3. If not found → fetch via configured transport.
4. Download asset bytes.
5. Verify H(bytes) == AssetID.
6. Store in cache.
7. Pass to renderer.

If verification fails → reject asset.

---

# 6. Cache Hierarchy

Clients SHOULD implement multi-tier cache:

Tier 0 — In-memory cache  
Tier 1 — Local disk cache  
Tier 2 — Regional cache  
Tier 3 — Network fetch  

Cache lookup order:

Tier 0 → Tier 1 → Tier 2 → Network

Cache MUST be:

- Content-addressed
- Immutable
- Verified on first load

---

# 7. Integrity Verification

Integrity verification rule:

H(asset_bytes) MUST equal AssetID.

If mismatch:

- Reject asset.
- Do not render.
- Do not cache corrupted data.

Compression MUST be decompressed before hash verification unless:

AssetID defined as compressed hash (explicit flag required).

---

# 8. Compression Rules

If AssetDescriptor.compression present:

- Client decompresses after fetch.
- Hash verification must follow canonical rule.

Compression algorithm ID must be deterministic and known.

Allowed algorithms governance-defined.

---

# 9. Encryption Rules

If encryption present:

AssetEncryption {
  encryption_algo: u16,
  key_id: Hash32,
  nonce: bytes
}

Decryption MUST occur before hash verification.

Key management is outside Truth Plane scope.

Encrypted assets still identified by hash of plaintext bytes.

---

# 10. Optional Offline Mode

Offline mode allows rendering without network connectivity.

Requirements:

- All referenced AssetIDs must exist in local cache.
- No network fetch permitted.
- Deterministic profile recommended.

If asset missing in offline mode:

Fallback behavior triggered (Section 11).

---

# 11. Fallback Synthesis Rules

If asset unavailable:

Client MAY synthesize fallback asset.

Fallback policy defined by:

RendererFallbackPolicy {
  geometry_missing: enum,
  texture_missing: enum,
  shader_missing: enum,
}

Fallback options:

- Substitute placeholder geometry
- Replace texture with flat color
- Use default material
- Hide object
- Abort rendering

Fallback MUST NOT:

- Modify RPACK
- Alter output_hash
- Modify Scene IR structure

Fallback applies only to client pixel output.

---

# 12. Determinism and Assets

If deterministic rendering required:

- Asset versions must be exact match.
- Fallback not permitted unless deterministic fallback defined.
- Compression and decompression must be deterministic.
- Asset ordering must be stable.

Deterministic profile MAY require:

Offline-capable complete asset set.

---

# 13. Asset Deduplication

Because AssetID is content-addressed:

Identical assets across RPACKs share same AssetID.

Clients MAY:

- Deduplicate storage
- Share cache across projects

Deduplication MUST NOT alter asset bytes.

---

# 14. Asset Pruning

Clients MAY prune cache using:

- LRU policy
- Size-based limits
- TTL policy

Pruning MUST NOT:

- Break offline mode unexpectedly (if offline required)
- Remove active session assets

Truth Plane not involved in cache pruning.

---

# 15. Security Considerations

Clients MUST:

- Verify AssetID hash before use.
- Avoid executing unverified shader code.
- Sandbox shader compilation.
- Avoid arbitrary URL execution.
- Enforce size limits before allocation.

Assets MUST be treated as untrusted input until verified.

---

# 16. Cross-Partition Asset Availability

If Intelligence Plane partitioned:

Asset MAY reside in specific region.

Fetch hints MAY optimize region selection.

Failure to fetch across partition MUST trigger fallback.

---

# 17. Rust Implementation Requirements

Implementation MUST:

- Hash asset bytes using canonical hash.
- Use fixed-size Hash32 representation.
- Validate size before allocation.
- Avoid unsafe memory operations.
- Support async fetch interface.
- Provide pluggable transport layer.

Cache keys MUST equal AssetID exactly.

---

# 18. Compliance Requirements

To claim LV-SPEC-0303 compliance:

Implementation MUST:

- Use content-addressed AssetID
- Verify hash before rendering
- Support cache hierarchy
- Support offline mode
- Support fallback synthesis rules
- Avoid embedding raw unverified URLs in Scene IR

Encryption and compression support MUST be recognized even if unused.

---

# 19. Design Invariants

1. Assets are content-addressed.
2. Asset bytes must match hash.
3. Network never stores pixel buffers.
4. Asset verification is client-side.
5. Offline rendering must not break determinism if declared.
6. Fallback must not modify canonical artifact.

---

# 20. Summary

LV-SPEC-0303 defines:

- Content-addressed asset model
- Cache hierarchy
- Fetch and integrity verification
- Offline rendering mode
- Fallback synthesis behavior

It ensures:

Scalable distribution.
Hash-based integrity.
Client-side trust boundary.
Deterministic rendering when required.

Lite-Vision remains:

Artifact-anchored.
Renderer-independent.
Network-light.
GPU-dedicated to cognition.

---

End of LV-SPEC-0303