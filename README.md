# H2H Protocol

**Peer-to-peer human authentication. Cryptographic proof that a real, biometrically verified human is on the other end.**

---

H2H is an open protocol for verifying human identity between two people, without any central authority. It uses the same hardware that powers passkeys (Secure Enclave, StrongBox) but redirects it: instead of proving yourself to a server, you prove yourself to another person.

## The Problem

Generative AI can now clone voices, faces, and mannerisms in real time. No widely deployed mechanism exists to cryptographically verify that a specific person — not software, not AI, not an impersonator — authorized a message, call, or file transfer.

- **Passkeys** prove you to a server. Not to a person.
- **Signal** encrypts your messages. It doesn't prove a human sent them.
- **Proof-of-personhood** (Worldcoin) proves you're human. Not *which* human.

H2H fills this gap.

## How It Works

1. **Identity Creation** — A P-256 key pair is generated inside your device's hardware security module. The private key never leaves the chip. Every use requires biometric authentication (Face ID, fingerprint).

2. **Contact Exchange** — You meet someone in person. You scan each other's QR codes. Each QR contains a signed payload with your public key and network address. The signature proves the payload came from the hardware — not a copy, not a replay.

3. **Connection** — Later, you connect peer-to-peer. Your device proves it holds the same key you exchanged in person, creating an unbroken trust chain from the physical meeting to the digital conversation.

4. **Presence Proof** — At any point, either party can challenge the other to produce a fresh signature, requiring a live biometric check. This proves a human is present *right now*, not just that a human set things up earlier.

## Key Properties

| Property | How |
|----------|-----|
| **No central authority** | Trust is rooted in physical meetings, not servers or CAs |
| **Hardware-bound keys** | Identity key lives in Secure Enclave / StrongBox, non-exportable |
| **Biometric-gated** | Signing requires Face ID / fingerprint at the hardware level |
| **Transport-agnostic** | Works over any encrypted P2P transport (QUIC, iroh, libp2p) |
| **Standard encoding** | CBOR/COSE wire formats, did:peer identity representation |
| **Recoverable** | Social recovery via K-of-N trusted contacts |

## Specification

The full protocol specification is in [`H2H_PROTOCOL.md`](H2H_PROTOCOL.md).

It covers:
- Identity creation and DID representation (Section 5)
- In-person contact exchange (Section 6)
- Trust tiers: in-person, vouched, video, unverified (Section 7)
- Social key recovery (Section 8)
- Generic data transfer with multiplexed channels (Sections 9-10)
- Presence Proof challenges (Section 11)
- Optional mailbox nodes for offline messaging (Section 12)
- Safety number verification (Section 13)
- Wire formats using CBOR and COSE (Section 15)
- Security and privacy considerations (Sections 16-17)

## Companion Documents (Planned)

| Document | Status | Scope |
|----------|--------|-------|
| **H2H-PROTOCOL** | Draft | Core protocol (this repo) |
| **H2H-CRYPTO** | Planned | E2E encryption: X3DH + Double Ratchet adapted for P-256 |
| **H2H-GROUP** | Planned | Group communication and membership |
| **H2H-TRANSPORT-IROH** | Planned | Transport binding for iroh |

## Prior Art and Novelty

H2H builds on established cryptographic standards (P-256 ECDSA, CBOR, COSE, did:peer, QUIC) and draws inspiration from existing systems (passkeys, Signal safety numbers, PGP web of trust, ISO 18013-5). The novel contribution is the specific combination of hardware-bound biometric identity, in-person trust bootstrapping, Presence Proofs, and tiered peer-to-peer trust — which has no equivalent in any existing specification.

## Status

This protocol is in **experimental draft** stage. It has not been audited or reviewed by a standards body. Do not use in production.

## License

The protocol specification is licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). You are free to share and adapt it for any purpose with appropriate credit.

## Authors

Rolab — [dev@rolabtech.com](mailto:dev@rolabtech.com)
