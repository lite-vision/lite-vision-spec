# LV-SPEC-0500 — Network Protocols and Message Formats
Version: 1.0  
Status: Draft (Engineering)  
Conformance Level: Core (Mandatory)  
Implements: LV-SPEC-0101, LV-SPEC-0107, LV-SPEC-0203, LV-SPEC-0205, LV-SPEC-0404  
Language Target: Rust  

---

# 1. Purpose

This specification defines the Lite-Vision Network Protocol layer.

It formalizes:

- P2P transport model
- PubSub topics
- Request/Response APIs
- Message authentication
- Anti-spam controls
- Backpressure and QoS mechanisms

The network layer must support:

- Truth Plane consensus traffic
- Intelligence Plane job routing
- Artifact distribution
- Verification and dispute messages

The protocol must be:

- Deterministic where required
- Authenticated
- DoS-resistant
- Resource-aware

---

# 2. Network Architecture Overview

Lite-Vision uses:

Hybrid P2P architecture

Nodes may play roles:

- Validator (Truth Plane)
- Operator (Intelligence Plane)
- Router
- Verifier
- Artifact Node
- Light Client

Nodes connect via authenticated P2P overlay.

---

# 3. Transport Layer

## 3.1 Transport Requirements

Transport MUST:

- Support encrypted connections (TLS 1.3 or Noise protocol)
- Support peer authentication
- Support multiplexed streams
- Support backpressure
- Support binary message framing

Recommended:

QUIC-based transport  
OR  
TCP + Noise + Yamux-style multiplexer

---

## 3.2 Peer Identity

PeerID = H(node_public_key)

Handshake MUST:

- Exchange public keys
- Verify signatures
- Bind session to PeerID
- Establish encrypted channel

Unauthenticated peers MUST NOT send consensus messages.

---

# 4. Message Framing

All messages follow canonical frame:

NetworkMessage {
  version: u16,
  message_type: u16,
  payload_length: u32,
  payload: bytes,
  signature: Option<Signature>
}

payload MUST be canonical serialized.

Max message size MUST be governance-defined.

---

# 5. Message Types

MessageType enum:

--- Truth Plane ---
0x01 BlockProposal  
0x02 Vote  
0x03 QuorumCertificate  
0x04 BlockRequest  
0x05 BlockResponse  

--- Intelligence Plane ---
0x10 JobBroadcast  
0x11 JobAssignment  
0x12 ReceiptSubmission  
0x13 VerificationRequest  
0x14 VerificationResult  
0x15 FraudProof  

--- Artifact Layer ---
0x20 ArtifactRequest  
0x21 ArtifactResponse  
0x22 DeltaRequest  
0x23 DeltaResponse  

--- Regional Memory ---
0x30 CRDTOperation  
0x31 CRDTSync  
0x32 SnapshotRequest  
0x33 SnapshotResponse  

--- Control ---
0xF0 Ping  
0xF1 Pong  
0xF2 PeerInfo  

---

# 6. PubSub Topics

Nodes MAY use PubSub overlay for broadcast-style traffic.

Topics MUST be namespaced:

lv/truth/blocks  
lv/truth/votes  
lv/intel/jobs  
lv/intel/receipts  
lv/artifacts/announcements  
lv/crdt/updates  

Topic naming MUST be deterministic and versioned.

Nodes subscribe only to relevant topics.

---

# 7. Request / Response APIs

RequestResponse {
  request_id: u64,
  method: enum,
  params: bytes,
}

Response {
  request_id: u64,
  status: enum { OK, ERROR },
  result: bytes,
}

Supported methods:

get_block  
get_receipt  
get_artifact  
get_snapshot  
get_partition_state  
submit_job  
submit_receipt  

Request/Response MUST enforce timeout and size limits.

---

# 8. Message Authentication

## 8.1 Signature Requirements

Consensus messages MUST be signed.

ReceiptSubmission MUST include:

- Operator signature (already inside receipt)

FraudProof MUST be signed by challenger.

ArtifactResponse MAY omit signature if:

- Transport authenticated
- Integrity verified via ArtifactID

---

## 8.2 Domain Separation

Signatures MUST use domain separation:

DOMAIN_BLOCK  
DOMAIN_VOTE  
DOMAIN_RECEIPT  
DOMAIN_FRAUD_PROOF  
DOMAIN_NETWORK_MSG  

Prevents cross-message replay.

---

# 9. Anti-Spam Controls

Network MUST implement:

- Rate limiting per PeerID
- Per-IP throttling (optional)
- Token bucket per topic
- Maximum concurrent streams
- Message size caps

---

## 9.1 Stake-Weighted Priority

Validators and staked operators MAY receive higher bandwidth allowance.

Unstaked peers limited to:

Light-client traffic only.

---

## 9.2 Job Broadcast Spam Mitigation

JobBroadcast MUST:

- Include escrow reference
- Include minimum fee
- Be rejected if escrow not found

Prevents zero-cost spam.

---

# 10. Backpressure Control

Nodes MUST:

- Monitor queue depth
- Drop lowest-priority messages first
- Apply dynamic throttling
- Advertise capacity via PeerInfo

Priority order (highest first):

1. Consensus votes
2. Block proposals
3. Fraud proofs
4. Receipt submissions
5. Job assignments
6. Artifact responses
7. CRDT updates

Low-priority messages MAY be dropped under congestion.

---

# 11. QoS Classes

QoSClass enum:

- ConsensusCritical
- VerificationHigh
- JobExecution
- ArtifactTransfer
- BackgroundSync

Each message tagged with QoS.

Transport layer MUST:

- Prioritize high-QoS
- Enforce bandwidth caps per QoS

QoS must not starve consensus traffic.

---

# 12. Flow Control

Per-connection flow control:

- Sliding window
- Max in-flight bytes
- Stream-level backpressure

Global flow control:

- Max total bandwidth
- Max memory for message buffering

Node MUST reject traffic exceeding limits.

---

# 13. Replay Protection

Messages MUST include:

- Nonce
- Timestamp (logical block height where relevant)
- Signature domain separation

Consensus messages must include:

block_height  
view_number  

Duplicate message detection REQUIRED.

---

# 14. Gossip Strategy

Gossip MAY be:

- Full flood (small networks)
- Partial fanout
- Score-based propagation

Nodes MUST:

- Avoid infinite rebroadcast loops
- Track seen message hashes
- Avoid rebroadcasting duplicates

---

# 15. Anti-DoS Safeguards

Nodes MUST:

- Validate message headers before allocation
- Enforce max payload length
- Reject malformed frames
- Limit CRDT operation rate
- Limit ArtifactRequest rate

Invalid peers MAY be temporarily banned.

---

# 16. Encryption and Privacy

Transport MUST encrypt all traffic.

Sensitive job payloads MAY be:

- Encrypted end-to-end
- Decrypted only by assigned operator

Encryption keys exchanged via secure handshake.

Truth Plane never sees plaintext job payload.

---

# 17. Versioning and Compatibility

Each message MUST include version field.

Nodes MUST:

- Reject incompatible major versions
- Allow minor version backward compatibility
- Negotiate supported features at handshake

---

# 18. Rust Implementation Requirements

Implementation MUST:

- Use canonical serialization (no float ambiguity)
- Enforce strict frame parsing
- Validate signatures before processing
- Use async IO
- Support multiplexed streams
- Implement per-peer rate limiting
- Maintain seen-message cache
- Avoid blocking consensus thread

Floating-point arithmetic MUST NOT influence consensus logic.

---

# 19. Compliance Requirements

To claim LV-SPEC-0500 compliance:

Implementation MUST:

- Support authenticated P2P transport
- Implement message framing
- Support defined message types
- Implement PubSub topics
- Enforce anti-spam rules
- Implement QoS prioritization
- Implement backpressure
- Prevent replay attacks
- Support version negotiation

---

# 20. Design Invariants

1. Consensus traffic must never be starved.
2. All critical messages must be authenticated.
3. Artifact integrity verified by hash, not trust.
4. Anti-spam must be stake-aware.
5. Backpressure must be enforced.
6. Replay must be impossible.
7. Network layer must not compute GPU workloads.

---

# 21. Summary

LV-SPEC-0500 defines the Lite-Vision network layer:

- Authenticated P2P transport
- Canonical message framing
- PubSub and request/response APIs
- Anti-spam and replay protection
- Backpressure and QoS prioritization

It ensures:

Scalable communication.
Consensus safety.
Efficient artifact distribution.
Resilient verification traffic.
Resource-aware networking.

Lite-Vision remains:

CPU-secured truth.
GPU-powered intelligence.
Client-rendered outputs.
Hash-anchored artifacts.
Network-disciplined coordination.

---

End of LV-SPEC-0500