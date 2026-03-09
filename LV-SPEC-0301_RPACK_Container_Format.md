# LV-SPEC-0301 — RPACK Container Format
Version: 1.0  
Status: Draft (Engineering)  
Conformance Level: Core (Mandatory)  
Implements: LV-SPEC-0103, LV-SPEC-0202, LV-SPEC-0300  
Language Target: Rust  

---

# 1. Purpose

This specification defines the RPACK (Render Packet) binary container format.

It formalizes:

- Chunked binary layout
- Streaming and progressive decoding
- Content addressing
- Compression hooks
- Encryption hooks

RPACK is the canonical output artifact of the Intelligence Plane.

The Truth Plane anchors only:

output_hash = H(canonical_RPACK_bytes)

RPACK MUST be:

- Canonically serialized
- Hash-stable
- Streamable
- Extensible

---

# 2. Design Goals

RPACK MUST:

1. Be self-describing
2. Support progressive decoding
3. Allow chunk-level content addressing
4. Support optional compression
5. Support optional encryption
6. Be forward-compatible
7. Be deterministic under canonical encoding

---

# 3. File Structure Overview

RPACK binary layout:

| Magic | Version | Header | ChunkTable | ChunkData | Footer |

All multi-byte integers MUST be little-endian.

---

# 4. Magic and Version

## 4.1 Magic Bytes

First 8 bytes:

0x52 0x50 0x41 0x43 0x4B 0x00 0x01 0x00  
("RPACK\0\1\0")

Reject if magic mismatch.

---

## 4.2 Version

u32 version field:

Current version: 1

Major version changes MAY break compatibility.

Minor changes MUST preserve backward compatibility.

---

# 5. Header Structure

Header {
  container_version: u32,
  flags: u32,
  scene_ir_hash: Hash32,
  metadata_hash: Hash32,
  chunk_table_offset: u64,
  chunk_count: u32,
  total_uncompressed_size: u64,
  deterministic_seed: Option<Hash32>,
  renderer_profile_hint: u16,
}

Header MUST be canonical and fixed-size (except optional seed flag).

Flags bitfield:

- bit 0: compressed
- bit 1: encrypted
- bit 2: streaming_allowed
- bit 3: deterministic_mode
- others reserved

---

# 6. Chunked Layout

RPACK uses chunked binary segments.

Each chunk:

ChunkHeader {
  chunk_id: u32,
  chunk_type: u16,
  flags: u16,
  uncompressed_size: u64,
  compressed_size: u64,
  content_hash: Hash32,
  data_offset: u64
}

Chunk types:

1 = SceneIR  
2 = Geometry  
3 = Texture  
4 = Shader  
5 = Animation  
6 = Metadata  
7 = Extension  
8 = Custom  

Chunk table stored immediately after header.

ChunkData region stores chunk payloads.

---

# 7. Chunk Semantics

## 7.1 SceneIR Chunk

Contains:

- Scene graph
- Node hierarchy
- Transform data
- Asset references

SceneIR MUST be canonical JSON-like binary schema OR deterministic binary schema.

---

## 7.2 Geometry Chunk

Contains:

- Vertex buffers
- Index buffers
- Mesh descriptors

Geometry MAY be large and streamed progressively.

---

## 7.3 Texture Chunk

Contains:

- Texture binary
- Mipmaps
- Format descriptor

May be compressed separately.

---

## 7.4 Shader Chunk

Contains:

- Shader source OR bytecode
- Deterministic shader subset flag

Must be sandbox-safe.

---

## 7.5 Metadata Chunk

Contains:

- Author info
- Timestamp (logical)
- Determinism flags
- JobID reference
- KernelID reference

Metadata MUST NOT contain system wall-clock time unless explicitly declared.

---

# 8. Streaming and Progressive Decoding

RPACK MUST support progressive decoding.

Streaming rules:

- Chunk table MUST appear before chunk data.
- Chunks MUST be independently decodable.
- SceneIR chunk SHOULD appear first.
- Geometry/Texture chunks MAY stream after SceneIR.

Client MAY:

- Begin rendering once SceneIR loaded.
- Lazy-load textures and geometry.

Streaming_allowed flag indicates streaming compliance.

---

# 9. Content Addressing

Each chunk includes:

content_hash = H(uncompressed_payload)

Whole RPACK hash:

RPACK_hash = H(full_canonical_binary)

Content addressing rules:

- Chunk hashes must match content_hash.
- Client MUST verify chunk hash before use.
- Duplicate chunk hashes MAY be deduplicated across RPACKs.

Content-addressed storage is recommended but not required.

---

# 10. Compression Hooks

Compression is optional.

If flags.compressed == 1:

- Each chunk MAY be individually compressed.
- compressed_size != uncompressed_size.

Allowed compression algorithms:

- Zstd
- LZ4
- Governance-approved alternatives

Compression algorithm identifier MUST be stored in chunk flags.

Decompression MUST be deterministic.

---

# 11. Encryption Hooks

If flags.encrypted == 1:

- Chunk payload encrypted.
- Encryption metadata stored in header.

Encryption fields:

EncryptionHeader {
  encryption_algo: u16,
  key_id: Hash32,
  nonce: [u8; 12],
}

Encryption MUST:

- Be applied per chunk.
- Not alter canonical unencrypted hash.
- Preserve content_hash as hash of unencrypted payload.

Client must decrypt before verifying content_hash.

Truth Plane does NOT manage encryption keys.

---

# 12. Canonical Encoding Rules

RPACK MUST:

- Use stable field ordering.
- Avoid floating-point nondeterminism in metadata.
- Avoid system clock timestamps.
- Use fixed-width integers.
- Serialize chunk table deterministically.

Any deviation changes RPACK_hash.

---

# 13. Deterministic Mode Binding

If deterministic_mode flag set:

- deterministic_seed MUST be present.
- All procedural generation seeded by deterministic_seed.
- Renderer profile MUST enforce deterministic subset.

Deterministic RPACK must be replayable.

---

# 14. Extension Mechanism

Chunk type 7 (Extension) reserved for future use.

Extension chunk MUST include:

- extension_id
- version
- payload
- extension_hash

Unknown extensions MUST:

- Be safely ignored
- Not break decoding

Forward compatibility guaranteed.

---

# 15. Validation Procedure

Client validating RPACK MUST:

1. Verify magic.
2. Verify version supported.
3. Parse header.
4. Verify chunk table integrity.
5. For each chunk:
   - Verify content_hash.
6. Verify overall RPACK_hash (if provided by Truth Plane).
7. Enforce renderer profile constraints.

If any step fails → reject RPACK.

---

# 16. Size Constraints

RPACK size MUST respect JobTicket.max_output_size.

Chunk count MUST NOT exceed governance-defined maximum.

Large artifacts SHOULD use streaming.

---

# 17. Rust Implementation Requirements

Implementation MUST:

- Use canonical serializer.
- Avoid unsafe memory when parsing.
- Validate bounds before reading.
- Enforce little-endian decoding.
- Verify content hashes before decompression (if possible).
- Support streaming via async readers.

---

# 18. Compliance Requirements

To claim LV-SPEC-0301 compliance:

Implementation MUST:

- Implement magic/version validation.
- Implement chunk table parsing.
- Implement content_hash verification.
- Support streaming decode.
- Support compression hook.
- Support encryption hook (even if unused).
- Preserve canonical serialization for hashing.

---

# 19. Design Invariants

1. RPACK is hash-addressable.
2. Chunk hashes must match content.
3. Whole-container hash must be stable.
4. Streaming must not alter canonical structure.
5. Compression must not change uncompressed hash.
6. Encryption must preserve content_hash integrity.
7. RPACK must never contain pixel buffers.

---

# 20. Summary

LV-SPEC-0301 defines the RPACK container:

- Chunked binary layout
- Progressive streaming
- Content addressing
- Compression hooks
- Encryption hooks
- Deterministic binding

It ensures:

Network produces structured reality.
Clients render.
Artifacts remain verifiable.
Hashes remain stable.

Lite-Vision remains:

CPU-secured.
GPU-powered.
Client-rendered.
Hash-anchored.

---

End of LV-SPEC-0301