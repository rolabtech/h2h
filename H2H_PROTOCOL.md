# Human-to-Human (H2H) Authentication and Data Transfer Protocol

**Document**: H2H-PROTOCOL-02
**Version**: 0.2.0-draft
**Date**: 2026-03-24
**Status**: Experimental
**Authors**: Rolab
**License**: This work is licensed under the Creative Commons
Attribution 4.0 International License (CC BY 4.0).

---

## Abstract

This document specifies the Human-to-Human (H2H) Protocol, a
cryptographic protocol that enables one human to prove their identity
to another human without relying on a trusted third party.

H2H binds a device's hardware security module and biometric
authentication to a persistent cryptographic identity represented as
a Decentralized Identifier (DID).  The protocol introduces the
concept of a **Human Proof**: cryptographic evidence that a
specific, biometrically verified individual authorized a particular
operation at a particular time.

The protocol defines identity creation, in-person contact exchange,
trust tiers, peer-to-peer data transfer, social key recovery, and
optional store-and-forward messaging.  It is transport-agnostic,
encoding-standardized (CBOR/COSE), and designed to operate without
central servers or certificate authorities.

---

## Status of This Document

This document is an experimental draft.  It has not been reviewed by
any standards body.  Distribution of this document is unlimited.

---

## Copyright Notice

Copyright (c) 2026 Rolab.  Licensed under Creative Commons
Attribution 4.0 International (CC BY 4.0).  You are free to share
and adapt this material for any purpose, including commercial, with
appropriate credit.

---

## Table of Contents

1.  Introduction
2.  Conventions and Terminology
3.  Protocol Overview
4.  Cryptographic Primitives
5.  Identity Layer
6.  Contact Exchange Protocol
7.  Trust Model
8.  Social Recovery
9.  Data Transfer Layer
10. Channel Types
11. Human Proof Challenges
12. Mailbox Nodes
13. Key Verification Fingerprint
14. Version Negotiation
15. Wire Formats
16. Security Considerations
17. Privacy Considerations
18. References

Appendix A.  P-256 Parameter Summary
Appendix B.  Test Vectors
Appendix C.  Relationship to Existing Standards
Appendix D.  Future Work

---

## 1.  Introduction

### 1.1.  Problem Statement

Advances in generative AI have made it trivial to clone a person's
voice, face, and mannerisms in real time.  A phone call from a
"family member" may be synthetic.  A video call with a "colleague"
may be a deepfake.  No widely deployed mechanism exists for one
person to cryptographically verify that another specific person --
not software acting on their behalf -- is on the other end of a
communication channel.

Existing identity systems address adjacent problems:

-  Passwords and passkeys (WebAuthn/FIDO2) authenticate a user to a
   server.  They do not authenticate one user to another user.

-  Certificate authorities (X.509, TLS) authenticate servers to
   clients.  They require a trusted third party.

-  End-to-end encryption (Signal Protocol) protects message content
   from eavesdroppers.  It does not prove that the person who
   controls a key is a specific human, nor that a human (rather than
   malware) authorized a particular message.

-  Proof-of-personhood systems (Worldcoin, Humanity Protocol) prove
   that an entity is human, but do not prove WHICH specific human.
   They require dedicated hardware or centralized infrastructure.

H2H fills this gap: peer-to-peer, hardware-rooted, biometric-gated
identity verification of a specific individual.

### 1.2.  Design Goals

The protocol is designed with the following goals, in priority order:

1.  **Human Proof**: Signed operations SHOULD be authorized by a
    biometric authentication event on the sender's device, producing
    cryptographic evidence of human presence.  The protocol defines
    two assurance levels for operations where biometric authentication
    is not available (Section 5.2).

2.  **No Trusted Third Party**: Identity verification MUST NOT depend
    on any central authority, certificate issuer, or identity
    provider.  Trust is established through physical human presence.

3.  **Transport Agnosticism**: The protocol defines abstract
    transport requirements.  Any transport layer that provides
    encrypted, multiplexed, peer-to-peer byte streams MAY be used
    (Section 9.1).

4.  **Generic Data Transfer**: The protocol provides a single
    authenticated channel over which any application-layer protocol
    (messaging, file transfer, audio, video) can operate.

5.  **Cross-Platform**: The protocol MUST be implementable on any
    device with a hardware security module and biometric sensor,
    regardless of operating system or manufacturer.

6.  **Recoverable**: Identity loss (device loss or destruction) MUST
    be recoverable through social mechanisms without requiring a
    central authority (Section 8).

### 1.3.  Scope

This document specifies:

-  Identity creation, key management, and DID representation (Sec. 5)
-  In-person contact exchange and verification (Section 6)
-  Trust establishment model (Section 7)
-  Social key recovery (Section 8)
-  Peer-to-peer data transfer requirements (Section 9)
-  Application-layer channel types (Section 10)
-  Human Proof challenges (Section 11)
-  Optional store-and-forward via mailbox nodes (Section 12)
-  Key Verification Fingerprint (Section 13)
-  Version negotiation (Section 14)
-  Wire formats using CBOR and COSE (Section 15)

This document does not specify:

-  End-to-end message encryption (defined in a companion document,
   H2H-CRYPTO, covering X3DH and Double Ratchet adapted for P-256)
-  Media codec selection or real-time media processing
-  User interface requirements
-  Group communication (defined in a companion document, H2H-GROUP)
-  A specific transport implementation (e.g., iroh, libp2p)

---

## 2.  Conventions and Terminology

### 2.1.  Requirements Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described in
BCP 14 [RFC2119] [RFC8174] when, and only when, they appear in all
capitals, as shown here.

### 2.2.  Definitions

**Human Proof**:  A COSE_Sign1 structure that provides cryptographic
   evidence a specific, biometrically verified individual authorized
   an operation.  Distinct from proof-of-personhood (which proves
   that an entity is A human); a Human Proof proves that a specific,
   previously verified individual authorized a specific operation.

**Device Authentication**:  A signing operation authorized by a
   device credential (PIN, passcode, pattern) rather than a biometric.
   Provides lower assurance than a Human Proof.  Used as a
   fallback when biometric authentication is unavailable.

**Identity Key (IK)**:  A P-256 ECDSA key pair generated inside a
   hardware security module.  The private key never leaves the
   hardware.  The public key, combined with key metadata, forms the
   user's Decentralized Identifier (DID).

**Transport Key (TK)**:  An asymmetric key pair used for peer-to-peer
   networking.  Generated in software.  Signed by the Identity Key
   to create a Key Binding Certificate.  The algorithm depends on
   the transport implementation (e.g., Ed25519 for QUIC-based
   transports).

**Key Binding Certificate (KBC)**:  A COSE_Sign1 [RFC9052] signed
   statement binding an Identity Key to a Transport Key.  Proves that
   a specific human (who controls the IK) authorizes a specific
   network node (identified by the TK).

**Contact Payload**:  A COSE_Sign1 signed structure exchanged in
   person (via QR code, BLE, or NFC) containing the sender's
   Identity Key, Transport Key, network addressing information, and a
   signature over all fields.

**Key Verification Fingerprint**:  A short value derived from both
   peers' Identity Keys, used for optional out-of-band verification
   that no key substitution occurred during contact exchange.  The
   human-readable encoding is application-defined.

**Hardware Security Module (HSM)**:  A tamper-resistant hardware
   component that generates, stores, and uses cryptographic keys
   without exposing the private key material to software.  In this
   document, HSM refers to Apple Secure Enclave, Android StrongBox,
   Android TEE, or equivalent.

**Channel**:  A multiplexed logical stream within an H2H connection,
   typed by application (control, messaging, file transfer, audio,
   video).

**Mailbox Node**:  An optional intermediary that stores encrypted
   messages for an offline peer and delivers them when the peer
   reconnects.  A mailbox node is untrusted: it sees only ciphertext
   and cannot forge or modify messages.

**Assurance Level**:  The degree of confidence that a signing
   operation was authorized by the claimed human.  H2H defines two
   levels: PRESENCE (biometric) and DEVICE (credential fallback).

### 2.3.  Notation

```
||      Concatenation of byte strings
len(X)  Length of X in bytes
H(X)    SHA-256 hash of X
ECDSA-Sign(key, data)   P-256 ECDSA signature
ECDSA-Verify(key, data, sig)   P-256 ECDSA verification
```

### 2.4.  Encoding

All protocol structures are encoded using CBOR [RFC8949] with
deterministic encoding as defined in Section 4.2 of RFC 8949.  All
signed structures use COSE_Sign1 [RFC9052] with algorithm identifier
ES256 (ECDSA w/ SHA-256 on P-256).

Every CBOR map payload includes a structure_type field (key 0) that
uniquely identifies the payload type, preventing type confusion
attacks where a signed structure of one type is substituted for
another.

---

## 3.  Protocol Overview

### 3.1.  Protocol Phases

The H2H Protocol operates in four phases:

```
Phase 1: Identity Creation
   User creates a P-256 identity key in the hardware security module
   and a Transport Key for networking.  The identity key signs the node
   key, producing a Key Binding Certificate.  The identity is
   represented as a did:peer DID.

Phase 2: Contact Exchange (in person)
   Two users meet physically.  Each generates a Contact Payload
   containing their DID, Transport Key, network addressing, and a
   COSE_Sign1 signature.  Payloads are exchanged via QR code or
   BLE.  Each user verifies the other's signature and stores the
   contact.

Phase 3: Connection Establishment
   Using the Transport Key and addressing info from the Contact Payload,
   peers establish a direct encrypted connection.  The connection
   is authenticated by verifying the Key Binding Certificate against
   the stored Identity Key.

Phase 4: Data Transfer
   Peers exchange data over multiplexed channels within the
   encrypted connection.  The signing model (per-message, per-
   session, or per-challenge) is an implementation decision
   (Section 10.1).
```

### 3.2.  Two-Key Architecture

H2H uses two asymmetric key pairs per user, each serving a distinct
purpose:

```
+--------------------------------------------------+
|  Identity Key (P-256 ECDSA)                      |
|  - Lives in hardware security module             |
|  - Private key is non-exportable                 |
|  - Use SHOULD require biometric authentication   |
|  - Purpose: PROVE "I am this specific human"     |
|  - Represented as: did:peer DID                  |
+--------------------------------------------------+
         |
         | signs (Key Binding Certificate, COSE_Sign1)
         v
+--------------------------------------------------+
|  Transport Key (transport-specific algorithm)         |
|  - Lives in software                             |
|  - Used for peer-to-peer transport               |
|  - Purpose: ROUTE "reach me at this address"     |
+--------------------------------------------------+
```

The Identity Key is the trust anchor.  It is constrained by hardware
to require biometric or device authentication for every use, making
it a proxy for human presence.  The Transport Key handles high-frequency
networking operations (handshakes, keepalives) that cannot
practically require authentication prompts.

The Key Binding Certificate links the two: "I, the human who
controls Identity Key IK, authorize Transport Key TK to act as my
network endpoint."

### 3.3.  Assurance Levels

The protocol defines two assurance levels for signing operations:

```
Level 1: PRESENCE (biometric authentication)
   The HSM gated key access on a biometric check (Face ID,
   fingerprint, iris scan).  This provides the highest assurance
   that a specific human authorized the operation.

Level 2: DEVICE (device credential authentication)
   The HSM gated key access on a device credential (PIN, passcode,
   pattern).  This provides assurance that someone with knowledge
   of the device credential authorized the operation, but does not
   prove biometric presence.
```

The assurance level MUST be indicated in every signed structure
(Section 15).  Implementations MUST display the assurance level to
the receiving user so they can make informed trust decisions.

Level 2 (DEVICE) exists to support users who cannot use biometrics
due to disability, injury, or device limitations.  Implementations
SHOULD default to Level 1 (PRESENCE) and only fall back to Level 2
when biometric authentication is unavailable or explicitly disabled
by the user.

### 3.4.  Relationship to Passkeys

The H2H Identity Key uses the same cryptographic primitive (P-256
ECDSA in a hardware security module, gated by biometrics) as
passkeys defined in WebAuthn [WebAuthn].  The critical differences
are:

1.  **Direction**: Passkeys authenticate a user to a server.  H2H
    authenticates a human to another human.

2.  **Key residency**: Passkeys are typically synced across devices
    via cloud backup (iCloud Keychain, Google Password Manager).
    H2H Identity Keys MUST NOT leave the hardware security module
    of the device on which they were created.

3.  **Relying party**: For passkeys, the relying party is a web
    origin.  For H2H, the relying party is another human's device.

4.  **Trust anchor**: Passkeys trust the server's stored public key.
    H2H trusts the in-person contact exchange event.

---

## 4.  Cryptographic Primitives

### 4.1.  Algorithms

The following algorithms are REQUIRED for conforming implementations:

| Function           | Algorithm         | Reference    |
|--------------------|-------------------|--------------|
| Identity signing   | ECDSA on P-256    | [FIPS186-5]  |
| Identity key gen   | P-256 in HSM      | [SEC2]       |
| Hash               | SHA-256           | [FIPS180-4]  |
| Encoding           | CBOR              | [RFC8949]    |
| Signed structures  | COSE_Sign1 (ES256)| [RFC9052]    |
| Key encoding       | COSE_Key          | [RFC9053]    |
| DID representation | did:peer Method 2 | [DID-PEER]   |

The following algorithms are RECOMMENDED for implementations that
provide end-to-end encryption (specified in companion doc H2H-CRYPTO):

| Function           | Algorithm         | Reference    |
|--------------------|-------------------|--------------|
| Key agreement      | ECDH on P-256     | [RFC6090]    |
| KDF                | HKDF-SHA-256      | [RFC5869]    |
| AEAD               | AES-256-GCM       | [RFC5116]    |

The Transport Key algorithm is not mandated by this specification, as
it depends on the transport implementation.  Implementations MUST
document which Transport Key algorithm they use.

### 4.2.  Why P-256

The Identity Key MUST be P-256 ECDSA because this is the only
elliptic curve universally supported by hardware security modules
across consumer devices:

-  Apple Secure Enclave: P-256 only [Apple-SE]
-  Android StrongBox: P-256 only for EC operations [Android-KS]
-  Android TEE: P-256 (plus broader curve support) [Android-KS]
-  Titan M / M2 (Google Pixel): P-256 [Titan-M]

### 4.3.  Key Sizes

| Key                | Algorithm | Private  | Public          |
|--------------------|-----------|----------|-----------------|
| Identity Key (IK)  | P-256     | 32 bytes | 65 bytes (uncompressed) or 33 bytes (compressed) |
| Transport Key (TK)      | Transport-dependent | Varies | Varies |

Identity Key public keys MUST be encoded as COSE_Key structures
[RFC9053] with key type EC2 (kty: 2) and curve P-256 (crv: 1).

---

## 5.  Identity Layer

### 5.1.  Identity Creation

Identity creation MUST occur once per device installation and MUST
be gated by biometric enrollment or device credential.  The
procedure is:

1.  The implementation MUST verify that a hardware security module
    is available.  If no HSM is present, identity creation MUST
    fail with an error.  Software-only key generation MUST NOT be
    used for the Identity Key.

2.  Generate a P-256 key pair inside the HSM with the following
    properties:
    -  Private key is non-exportable
    -  Access control SHOULD require biometric authentication
    -  Access control MAY allow device credential fallback for
       accessibility (see Section 3.3)
    -  The key is tagged with a unique, application-scoped identifier

3.  Generate a Transport Key pair appropriate for the transport layer.

4.  Construct the Key Binding Certificate (Section 5.4).

5.  Derive the DID from the Identity Key public key (Section 5.3).

### 5.2.  Identity Key Properties

The following properties MUST hold for the Identity Key:

-  **Non-exportable**: The private key MUST never leave the HSM.
   No API call, backup mechanism, or device migration process
   SHALL be capable of extracting it.

-  **Authentication-gated**: Every signing operation MUST trigger
   an authentication prompt.  Implementations SHOULD use biometric
   authentication (producing PRESENCE assurance level).
   Implementations MAY allow device credential fallback (producing
   DEVICE assurance level) for accessibility.

-  **Invalidated on biometric change**: If biometric authentication
   is used and the device's biometric enrollment changes (e.g., a
   new fingerprint is added), the key MUST be invalidated by the
   HSM.  This prevents an attacker who gains temporary physical
   access from adding their biometric and subsequently producing
   valid PRESENCE-level signatures.

-  **Single device**: A given Identity Key exists on exactly one
   physical device.  A user with multiple devices has multiple
   Identity Keys, each independently verified through contact
   exchange.

### 5.3.  DID Representation

Each Identity Key is represented as a Decentralized Identifier
using the did:peer method [DID-PEER], specifically Method 2
(multiple key agreement and authentication keys).

The DID encodes:
-  The Identity Key public key (authentication)
-  The current Transport Key public key (key agreement / service endpoint)
-  A service endpoint for network addressing

Example:

```
did:peer:2.Ez6LSfM...authentication_key...
             .Vz6Mkr...node_key...
             .SeyJ0I...service_endpoint...
```

Using did:peer allows H2H identities to interoperate with the
broader Decentralized Identity ecosystem, including Verifiable
Credentials [VC-DATA-MODEL] and DIDComm [DIDCOMM-V2].

### 5.4.  Key Binding Certificate

The Key Binding Certificate (KBC) binds an Identity Key to a
Transport Key.  It is a COSE_Sign1 structure:

```
KBC = COSE_Sign1(IK_private, KBC_payload)

KBC_payload (CBOR map):
   0 (structure_type): uint, 0x01 (KBC)
   1 (version):       uint, 1
   2 (ik_public):     COSE_Key (P-256)
   3 (tk_public):     bstr (transport-specific encoding)
   4 (tk_algorithm):  tstr (e.g., "Ed25519", "P-256")
   5 (timestamp):     uint (Unix milliseconds)
   6 (expiry):        uint (Unix milliseconds)
   7 (assurance):     uint (1 = PRESENCE, 2 = DEVICE)

COSE_Sign1 protected header:
   1 (alg): -7 (ES256)
```

Verification of a KBC:

1.  Decode the COSE_Sign1 structure.
2.  Verify that the current time is between timestamp and expiry.
3.  Verify the ECDSA signature using the ik_public from the payload.
4.  If verification succeeds, the TK is authorized to act on behalf
    of the human who controls IK.

The expiry SHOULD be set to 30 days from the timestamp.
Implementations MUST re-sign the KBC before expiry to maintain
connectivity.

### 5.5.  Transport Key Rotation

The Transport Key MAY be rotated at any time to limit the window during
which a network observer can correlate a user's activity to a stable
identifier.

The rotation procedure is:

1.  Generate a new Transport Key pair.
2.  Sign a new KBC binding the Identity Key to the new Transport Key.
    This signing operation requires authentication (biometric or
    device credential).
3.  Notify connected peers of the new Transport Key via a KEY_ROTATION
    message on the Control Channel (Section 10.2).
4.  Update the DID document to reflect the new Transport Key.
5.  Discard the old Transport Key private material.

Peers that receive a KEY_ROTATION message MUST:

1.  Verify the new KBC signature against the stored Identity Key.
2.  If valid, update the stored Transport Key and addressing information.
3.  Reconnect using the new Transport Key.

Implementations SHOULD rotate the Transport Key at least once every 30
days.  Implementations MAY rotate more frequently (e.g., per
connection or per session) at the cost of reduced reachability
during the rotation window.

### 5.6.  Offline Rotation

When the Transport Key is rotated while a contact is offline, the
contact will be unable to locate the key holder using the previous
Transport Key.  Implementations MUST address this using one or more
of the following:

-  **Stable addressing (RECOMMENDED)**: The addressing field in the
   Contact Payload (Section 6.2, field 9) SHOULD contain addressing
   information that survives Transport Key rotation (e.g., a relay
   server address or discovery service endpoint).  When a contact
   reconnects via the stable address, the key holder presents the
   current KBC containing the new Transport Key.

-  **Mailbox deposit**: If the implementation supports mailbox nodes
   (Section 12), the key holder SHOULD deposit the new KBC at their
   mailbox node.  Offline contacts retrieve the updated KBC upon
   reconnection.

-  **Overlap period**: The key holder MAY continue to accept
   connections on the old Transport Key for a limited time after
   rotation (RECOMMENDED: no more than 24 hours).  Connections on
   the old key SHOULD be used only to deliver the new KBC and
   redirect the contact.

Implementations MUST NOT rotate the Transport Key without a
mechanism for offline contacts to discover the new key.

### 5.7.  Identity Lifecycle

```
+-------------------+
| UNINITIALIZED     |  No identity key exists on this device
+-------------------+
         |
         | createIdentity() [biometric or device credential]
         v
+-------------------+
| ACTIVE            |  Identity key exists, KBC is valid
+-------------------+
         |                    |
         | lock / app         | KBC expiry approaching
         | background         |
         v                    v
+-------------------+   +-------------------+
| LOCKED            |   | RENEWAL REQUIRED  |
| HSM key exists    |   | Re-sign KBC       |
| Auth needed       |   | [auth prompt]     |
+-------------------+   +-------------------+
         |                    |
         | auth               | auth success
         | success            |
         v                    v
+-------------------+
| ACTIVE            |
+-------------------+
         |
         | deleteIdentity() or device loss
         v
+-------------------+
| UNINITIALIZED     |  Recover via Social Recovery (Section 8)
+-------------------+
```

---

## 6.  Contact Exchange Protocol

### 6.1.  Overview

Contact exchange is the trust establishment ceremony.  It MUST
occur with both parties physically present.  This requirement is
not a limitation; it is the core security property.  The physical
meeting is the root of trust that replaces certificate authorities
and identity providers.

### 6.2.  Contact Payload Format

Each party generates a Contact Payload as a COSE_Sign1 structure
containing all information the peer needs to both verify identity
and establish a network connection:

```
Contact Payload = COSE_Sign1(IK_private, payload)

payload (CBOR map):
   0 (structure_type): uint, 0x02 (Contact Payload)
   1 (version):       uint, 1
   2 (did):           tstr (did:peer:2.Ez6LS...)
   3 (ik_public):     COSE_Key (P-256)
   4 (tk_public):     bstr (transport-specific)
   5 (tk_algorithm):  tstr (e.g., "Ed25519")
   6 (display_name):  tstr (UTF-8, max 64 bytes)
   7 (timestamp):     uint (Unix milliseconds)
   8 (nonce):         bstr (16 bytes, cryptographically random)
   9 (addressing):    bstr (transport-specific addressing info)
  10 (assurance):     uint (1 = PRESENCE, 2 = DEVICE)

COSE_Sign1 protected header:
   1 (alg): -7 (ES256)
```

The addressing field (9) contains transport-specific information
needed to establish a connection (e.g., relay addresses, direct
IP:port hints, or a serialized connection ticket).  Its format is
opaque to the protocol and defined by the transport binding
document.

### 6.3.  Exchange Procedure

```
Alice                                             Bob
  |                                                 |
  |  [both physically present, devices in hand]     |
  |                                                 |
  |  1. Generate Contact Payload                    |
  |     (triggers auth prompt)                      |
  |                                                 |
  |  2. Encode as QR code                           |
  |                                                 |
  |  3. -------  Bob scans Alice's QR  ------->     |
  |                                                 |
  |     4. Bob verifies Alice's COSE_Sign1 sig      |
  |     5. Bob holds Alice's payload (not yet stored)|
  |                                                 |
  |  6. Bob generates his Contact Payload           |
  |     (triggers auth prompt)                      |
  |                                                 |
  |  7. Bob displays QR code                        |
  |                                                 |
  |  8. <------  Alice scans Bob's QR  --------     |
  |                                                 |
  |  9. Alice verifies Bob's COSE_Sign1 sig         |
  |                                                 |
  |  [Exchange is now bidirectional — both store]   |
  |  10. Both devices store the other's contact     |
  |                                                 |
  |  [optionally compare fingerprints to confirm]   |
  |                                                 |
```

### 6.4.  Verification Rules

Upon receiving a Contact Payload, the implementation MUST:

1.  Decode the COSE_Sign1 structure.
2.  Verify that structure_type (field 0) is 0x02 (Contact Payload).
3.  Verify that the version field is supported.
4.  Verify that the timestamp is within a 5-minute window of the
    current device time.
5.  Verify that the nonce has not been seen before (within the
    5-minute window).
6.  Verify the COSE_Sign1 signature using the ik_public from the
    payload.
7.  If all checks pass AND the exchange is bidirectional (the local
    device has also transmitted its own Contact Payload to this
    peer), store the contact at Trust Tier 1 (in-person verified).

The contact exchange MUST be bidirectional: both parties MUST
exchange and verify Contact Payloads before either party stores
the other as a verified contact.

If ANY check fails, the implementation MUST reject the payload and
MUST NOT store any contact information.

### 6.5.  Exchange Mechanisms

The Contact Payload binary (serialized COSE_Sign1) MAY be
transmitted via any proximity mechanism.  This specification
defines the following:

**QR Code (REQUIRED)**:  Implementations MUST support QR code
exchange.  The serialized COSE_Sign1 is encoded as a binary-mode
QR code.  QR Version 25 supports up to 1,270 bytes in binary mode,
which is sufficient for a typical Contact Payload (~300-400 bytes).

**BLE Proximity Exchange (RECOMMENDED)**:  Both devices advertise
a service UUID specific to H2H.  Upon discovery, devices exchange
Contact Payloads over a BLE GATT characteristic.  BLE range
(~10-30m) provides a weaker proximity guarantee than QR; the
implementation SHOULD require explicit user confirmation before
accepting a BLE-discovered contact.

**NFC Tag Read (OPTIONAL)**:  A user MAY write their Contact Payload
to a physical NFC tag (e.g., an NFC business card).  The peer
reads the tag to obtain the payload.  Note: peer-to-peer NFC
(Android Beam) is deprecated on modern platforms.  NFC exchange
is one-directional per read; bidirectional exchange requires two
sequential tag reads or a combination with QR/BLE.

---

## 7.  Trust Model

### 7.1.  Trust Tiers

H2H defines a hierarchical trust model with four tiers:

```
Tier 1: In-Person Verified (PRESENCE)
   You met this person face-to-face and exchanged Contact Payloads
   with biometric-level assurance.  Highest trust.  The only tier
   that provides full Human Proof.

Tier 2: Vouched Introduction
   A Tier 1 contact introduced this person via a signed vouching
   statement.  You trust the introducer's judgment.  Reduced trust.

Tier 3: Video Verified
   You conducted a live video call with this person and compared
   Key Verification Fingerprints.  Moderate trust -- susceptible to real-time
   deepfakes.

Tier 4: Key Only
   You have this person's public key from an unverified source
   (e.g., a website, email, messaging app).  Minimal trust -- no
   human verification has occurred.
```

### 7.2.  Trust Tier Rules

-  The trust tier is assigned at the time of contact exchange and
   stored alongside the contact record.
-  The tier MUST NOT be automatically upgraded.  A contact at Tier 4
   can only reach Tier 1 through an in-person meeting.
-  The implementation MUST display the trust tier to the user in any
   context where the contact's identity is relevant.
-  If a contact exchanges at assurance level DEVICE (not biometric),
   the resulting tier SHOULD be annotated as "Tier 1 (device)" to
   distinguish from "Tier 1 (presence)".

### 7.3.  Vouched Introductions (Tier 2)

A Tier 1 contact (the "introducer") may introduce two of their
own Tier 1 contacts to each other.

The vouch is a COSE_Sign1 structure:

```
Vouch = COSE_Sign1(introducer_IK_private, vouch_payload)

vouch_payload (CBOR map):
   1 (version):             uint, 1
   2 (introducer_did):      tstr
   3 (introduced_did):      tstr
   4 (introduced_ik):       COSE_Key
   5 (introduced_nk):       bstr
   6 (introduced_addressing): bstr
   7 (timestamp):           uint (Unix milliseconds)
   8 (assurance):           uint
```

Each receiving party stores the introduced contact at Tier 2 with
a reference to the introducer.  A Tier 2 contact MAY be upgraded
to Tier 1 through a subsequent in-person meeting.

### 7.4.  Trust Degradation

If a contact's Identity Key changes (new device, recovery), the
trust tier MUST be reset to Tier 4.  The implementation MUST warn
the user: "[Contact name]'s identity key has changed.  Previous
verification is no longer valid."

Exception: if the key change occurred via Social Recovery
(Section 8), the recovered identity starts at Tier 2 with the
contacts who participated in recovery.

---

## 8.  Social Recovery

### 8.1.  Overview

When a device is lost, stolen, or destroyed, the Identity Key is
permanently lost (it cannot be extracted from the HSM).  Social
Recovery allows the user to establish a new Identity Key on a new
device and have existing contacts recognize the new key as belonging
to the same person.

### 8.2.  Recovery Configuration

A user MAY designate a set of K-of-N recovery contacts from their
Tier 1 contacts.  The designation is a signed statement stored
locally (and optionally distributed to the designated contacts):

```
RecoveryConfig = COSE_Sign1(IK_private, config_payload)

config_payload (CBOR map):
   1 (version):           uint, 1
   2 (owner_did):         tstr
   3 (threshold):         uint (K, minimum vouches required)
   4 (recovery_contacts): array of tstr (DIDs of designated contacts)
   5 (timestamp):         uint
```

RECOMMENDED defaults: K=3, N=5 (3 of 5 contacts must vouch).

### 8.3.  Recovery Procedure

```
1. User gets a new device and creates a new Identity Key (IK_new).

2. User contacts K or more recovery contacts out-of-band (phone
   call, in-person, another messaging app) and provides IK_new's
   public key.

3. Each recovery contact verifies the user's identity through
   whatever means they deem sufficient (voice recognition,
   personal knowledge, in-person meeting) and signs a Recovery
   Vouch:

   RecoveryVouch = COSE_Sign1(contact_IK_private, vouch_payload)

   vouch_payload (CBOR map):
      1 (version):       uint, 1
      2 (old_did):       tstr (the user's previous DID)
      3 (new_did):       tstr (the user's new DID)
      4 (voucher_did):   tstr
      5 (timestamp):     uint

4. The user collects K or more RecoveryVouches and presents them
   to other contacts.

5. A contact that receives K valid RecoveryVouches from designated
   recovery contacts SHOULD accept the new Identity Key at Tier 2.

6. The user MAY upgrade recovered contacts to Tier 1 through
   subsequent in-person meetings.
```

### 8.4.  Recovery Limitations

-  Social Recovery requires that recovery contacts are reachable
   through an out-of-band channel.
-  Recovery contacts verify the user's identity subjectively; the
   protocol does not prescribe how this verification occurs.
-  A compromised set of K recovery contacts could vouch for an
   attacker's key.  This is a fundamental trade-off between
   recovery and security.
-  The recovered identity starts at Tier 2, not Tier 1, reflecting
   the reduced assurance of non-in-person verification.

---

## 9.  Data Transfer Layer

### 9.1.  Transport Requirements

The H2H Protocol is transport-agnostic.  A conforming transport
implementation MUST provide:

1.  **Encrypted connections**: All data in transit MUST be encrypted.
    The transport MUST provide confidentiality and integrity.

2.  **Peer-to-peer connectivity**: The transport MUST support direct
    connections between two peers, with fallback to relay when
    direct connection is not possible.

3.  **Multiplexed streams**: The transport MUST support multiple
    concurrent logical streams (reliable, ordered) and SHOULD
    support unreliable datagrams for real-time media.

4.  **Authenticated peer identity**: The transport MUST expose the
    peer's Transport Key public key so that the H2H protocol layer can
    verify it against the stored Key Binding Certificate.

5.  **NAT traversal**: The transport SHOULD provide automatic NAT
    traversal to maximize direct connectivity.

6.  **Local discovery**: The transport SHOULD support local network
    peer discovery (e.g., mDNS [RFC6762]) for offline/local
    scenarios.

Suitable transport implementations include (but are not limited
to): iroh [iroh], libp2p-quic [libp2p], or any QUIC [RFC9000]
implementation with P2P extensions.

### 9.2.  Connection Establishment

```
Alice (initiator)                          Bob (responder)
  |                                          |
  |  1. Look up Bob's TK + addressing from   |
  |     stored contact                       |
  |                                          |
  |  2. Establish encrypted connection       |
  |     using transport layer                |
  |  ---- transport handshake -------------> |
  |                                          |
  |  3. Both sides exchange KBCs             |
  |  <--- KBC(Bob.IK -> Bob.TK) ----------- |
  |  ---- KBC(Alice.IK -> Alice.TK) ------> |
  |                                          |
  |  4. Each side verifies:                  |
  |     a. KBC signature is valid (COSE)     |
  |     b. IK matches stored contact DID     |
  |     c. TK matches the transport peer     |
  |     d. KBC has not expired               |
  |                                          |
  |  5. If all checks pass:                  |
  |     connection is AUTHENTICATED          |
  |                                          |
  |  6. Exchange supported versions          |
  |     (Section 14)                         |
  |                                          |
  |  7. Open multiplexed channels            |
  |  <========= DATA TRANSFER ============> |
  |                                          |
```

If any verification check in step 4 fails, the connection MUST be
terminated immediately.  The implementation MUST NOT fall back to
an unauthenticated mode.

### 9.3.  Connection Authentication Chain

After the transport handshake, both peers exchange Key Binding
Certificates over the encrypted connection.  The verification
chain proves:

```
Stored IK (from in-person meeting)
  = IK in KBC (verified by COSE_Sign1 signature)
    = IK that signed TK
      = TK of the transport peer (verified by transport handshake)
```

Therefore, the transport peer is controlled by the same human who
was physically present at the contact exchange.

### 9.4.  Relay Servers

When direct peer-to-peer connectivity is not achievable, the
transport layer MAY route traffic through relay servers.  Relay
servers are untrusted transport.  They:

-  CANNOT read message content (transport encryption).
-  CANNOT forge messages (no access to Identity Keys).
-  CANNOT forge Key Binding Certificates.
-  CAN observe connection metadata (source/destination Transport Keys,
   timing, data volume).
-  CAN drop or delay packets (denial of service).

### 9.5.  When to Use Signed Messages vs Transport Authentication

Once the KBC is verified and the transport connection is established,
the transport layer already provides authentication and encryption.
The question is: when do applications need the additional SSK-signed
messages defined in Section 10.1?

**Use SSK-signed messages when:**

-  **Messages are stored or forwarded**: Text messages saved in chat
   history, files delivered via mailbox nodes, or any content that
   outlives the transport session.  A signed message is self-contained
   and verifiable without the original transport connection.
-  **Non-repudiation is needed**: Approval workflows, financial
   authorizations, or any operation where the receiver needs to prove
   the sender produced the content.  Transport auth only proves "this
   came over Bob's connection" — it cannot be verified after the
   connection closes.
-  **Relay or store-and-forward delivery**: A compromised relay could
   inject or modify unsigned content.  SSK signatures protect integrity
   end-to-end regardless of the delivery path.

**Transport authentication alone is sufficient when:**

-  **Real-time media streams**: Audio and video frames over an
   authenticated, direct transport connection.  The KBC already proved
   the transport peer is the person from the ceremony.  Signing every
   frame would add latency with no practical security benefit.
-  **Ephemeral signaling**: Keepalive, typing indicators, read receipts,
   or other transient data that has no value after the session ends.

**Summary by channel type:**

| Channel | Signed Messages | Rationale |
|:--------|:----------------|:----------|
| Control (0x00) | No | Transport-authenticated, ephemeral signaling |
| Messages (0x01) | Yes (SHOULD) | Stored, forwarded, non-repudiation |
| Files (0x02) | Yes (header) | Significant operation, stored |
| Audio (0x03) | No | Real-time, latency-sensitive |
| Video (0x04) | No | Real-time, latency-sensitive |

For channels that do not use SSK signatures, human presence during the
session is verified via Human Proof Challenges on the Control Channel
(Section 11).

---

## 10.  Channel Types

### 10.1.  Session Credentials

A Session Credential is a short-lived signing delegation from the
Identity Key to an ephemeral Session Signing Key (SSK).  It enables
per-message authentication without requiring an authentication event
for each signing operation.  This pattern is analogous to TLS
Delegated Credentials [RFC9345].

The delegation creates a verifiable chain from any signed message
back to the Identity Key established during the in-person contact
exchange:

```
Stored IK_public (from in-person meeting)
  |
  v  verifies
Session Credential (COSE_Sign1 signed by IK)
  |  contains SSK_public
  v  verifies
Signed Message (COSE_Sign1 signed by SSK)
```

#### Session Credential Format

```
SessionCredential = COSE_Sign1(IK_private, SC_payload)

SC_payload (CBOR map):
   0 (structure_type):  uint, 0x03 (SESSION_CREDENTIAL)
   1 (version):         uint, 1
   2 (ssk_public):      COSE_Key (ephemeral P-256)
   3 (ik_public):       COSE_Key (issuing Identity Key)
   4 (peer_ik_hash):    bstr (SHA-256 of peer's IK_public)
   5 (timestamp):       uint (Unix milliseconds)
   6 (expiry):          uint (Unix milliseconds, application-defined)
   7 (assurance):       uint (1 = PRESENCE, 2 = DEVICE)
```

Creation requires an authentication event (biometric or device
credential).  The SSK is generated in software, held in memory
only, and MUST NOT be persisted to disk.

#### Signed Message Format

```
SignedMessage = COSE_Sign1(SSK_private, SM_payload)

SM_payload (CBOR map):
   0 (structure_type):  uint, 0x04 (SIGNED_MESSAGE)
   1 (message_id):      uint (unique per sender, monotonic)
   2 (timestamp):       uint (Unix milliseconds)
   3 (content_type):    uint (application-defined)
   4 (content):         bstr or tstr
```

SSK signing does NOT require an authentication event.  This enables
signing every message without user interaction.

#### Verification Chain

To verify a Signed Message, the verifier:

1.  Verifies the Session Credential: structure_type is 0x03;
    signature is valid against the sender's stored IK_public;
    peer_ik_hash matches the verifier's own IK; channel_binding
    matches the current connection; credential has not expired.
2.  Extracts ssk_public from the Session Credential.
3.  Verifies the Signed Message: structure_type is 0x04; signature
    is valid against ssk_public.

The Session Credential is verified once per session; ssk_public
is cached for subsequent message verifications.

#### Credential Lifecycle

-  A Session Credential SHOULD be created at connection
   establishment, immediately after KBC exchange.
-  The credential MUST have a finite expiry.  Duration is
   application-defined (RECOMMENDED default: 1 hour).  Shorter for
   high-risk operations, longer for extended sessions.
-  When expired, the sender creates a new credential (requires
   fresh authentication).
-  When the connection closes, the SSK MUST be discarded.
-  For operations requiring the strongest assurance (e.g., file
   transfer authorization, financial transactions), implementations
   SHOULD sign directly with the Identity Key rather than the SSK.

All signed structures MUST include the assurance level (PRESENCE or
DEVICE) so that the receiving party knows what level of
authentication was performed.

### 10.2.  Channel Multiplexing

A single H2H connection supports multiple concurrent channels:

```
H2H Connection (one transport connection)
|
+-- Channel 0x00: Control (reliable, ordered)
|     Signaling, version negotiation, capability advertisement,
|     keepalive, Human Proof challenges, key rotation
|
+-- Channel 0x01: Messages (reliable, ordered)
|     Text messages, reactions, read receipts
|
+-- Channel 0x02: Files (reliable, ordered)
|     File transfer with progress reporting
|
+-- Channel 0x03: Audio (unreliable datagrams)
|     Real-time audio frames
|
+-- Channel 0x04: Video (unreliable datagrams)
|     Real-time video frames
|
+-- Channel 0x05-0xFE: Reserved for future use
|
+-- Channel 0xFF: Extension (reliable)
      Custom application-defined protocols
```

### 10.3.  Control Channel (0x00)

The Control Channel is REQUIRED and is opened immediately after
connection authentication.  It carries:

-  **Version negotiation** (Section 14)
-  **Capability advertisement**: Each peer advertises supported
   channel types.
-  **Channel open/close requests**: A peer MUST request to open a
   channel before sending data on it.
-  **Keepalive**: RECOMMENDED interval of 30 seconds.
-  **Human Proof challenges** (Section 11)
-  **Key rotation notifications** (Section 5.5)

### 10.4.  Message Channel (0x01)

The Message Channel carries text messages and associated metadata
using reliable, ordered streams.  The message format is a CBOR map:

```
Message (CBOR map):
   1 (message_id):    uint (unique per sender, monotonic)
   2 (timestamp):     uint (Unix milliseconds)
   3 (content_type):  uint (1 = text/plain, 2 = text/markdown)
   4 (content):       tstr (UTF-8)
```

Messages SHOULD be signed using the Signed Message format defined
in Section 10.1 (COSE_Sign1 signed by the Session Signing Key).
This provides per-message authentication with a verifiable chain
back to the Identity Key from the in-person contact exchange,
without requiring an authentication event for each message.

### 10.5.  File Channel (0x02)

```
FileHeader (CBOR map):
   1 (transfer_id):   bstr (16 bytes, random)
   2 (filename):      tstr
   3 (file_size):     uint
   4 (sha256):        bstr (32 bytes)

FileChunk (CBOR map):
   1 (transfer_id):   bstr
   2 (chunk_index):   uint
   3 (chunk_data):    bstr
```

File transfer initiation SHOULD require authentication (the
FileHeader is wrapped in COSE_Sign1) regardless of the signing
model, as file transfers represent a significant operation.

### 10.6.  Audio Channel (0x03)

Real-time audio using unreliable datagrams.

```
AudioFrame (CBOR map):
   1 (seq):           uint
   2 (timestamp_us):  uint (microseconds)
   3 (codec_data):    bstr
```

Audio frames are NOT individually signed (latency constraint).
Human presence during calls is verified via Human Proof
challenges on the Control Channel (Section 11).

### 10.7.  Video Channel (0x04)

Real-time video using unreliable datagrams.

```
VideoFrame (CBOR map):
   1 (seq):           uint
   2 (timestamp_us):  uint (microseconds)
   3 (frame_type):    uint (1 = keyframe, 2 = delta)
   4 (frag_index):    uint
   5 (frag_count):    uint
   6 (codec_data):    bstr
```

Like audio, video frames are not individually signed.

---

## 11.  Human Proof Challenges

### 11.1.  Overview

During a long-lived connection (e.g., an ongoing call), either
peer MAY send a Human Proof Challenge on the Control Channel,
requesting the other peer to produce a fresh Human Proof.

### 11.2.  Challenge-Response

```
Challenge (CBOR map):
   0 (structure_type):    uint, 0x10 (PROOF_CHALLENGE)
   1 (challenge_nonce):   bstr (32 bytes, random)
   2 (required_assurance): uint (1 = PRESENCE required, 2 = DEVICE ok)

Response = COSE_Sign1(IK_private, response_payload)

response_payload (CBOR map):
   0 (structure_type):    uint, 0x11 (PROOF_RESPONSE)
   1 (challenge_nonce):   bstr (echoed from challenge)
   2 (assurance):         uint (1 = PRESENCE, 2 = DEVICE)
   3 (channel_binding):   bstr (SHA-256 of connection context)
```

The required_assurance field allows the challenger to demand a
specific assurance level.  If set to 1 (PRESENCE), the responder
MUST authenticate via biometric; a DEVICE-level response MUST be
rejected.

The channel_binding field MUST contain the SHA-256 hash of a
connection-specific value (e.g., the concatenation of both peers'
Transport Key public components).  This prevents relay attacks where an
attacker forwards a challenge to the victim and relays the response
to a different connection.

### 11.3.  Timing Constraints

-  Implementations SHOULD NOT issue challenges more frequently than
   once per 5 minutes, to avoid excessive authentication prompts.
-  The challenged peer MUST respond within 60 seconds.
-  If no valid response is received, the challenger SHOULD warn the
   user: "[Contact name] did not respond to presence verification."
-  Failure to respond MUST NOT automatically terminate the
   connection.  The user decides how to proceed.

---

## 12.  Mailbox Nodes

### 12.1.  Overview

H2H is peer-to-peer: both peers must be online simultaneously
for real-time communication.  For asynchronous messaging, the
protocol defines an OPTIONAL mailbox node role.

A mailbox node is an always-on network node that accepts messages
on behalf of an offline peer and delivers them when the peer
reconnects.

### 12.2.  Trust Model

A mailbox node is UNTRUSTED infrastructure.  It:

-  Stores only opaque byte blobs (encrypted messages).
-  CANNOT read, modify, or forge message content.
-  CANNOT determine sender or recipient identity beyond their
   Transport Keys (which may be rotated).
-  MAY observe message timing and size metadata.
-  MUST delete messages after successful delivery.

### 12.3.  Mailbox Registration

A user registers with a mailbox node by providing their current
Transport Key.  The mailbox node stores messages addressed to that
Transport Key.  Registration does not require Identity Key disclosure.

```
MailboxRegister (CBOR map):
   1 (tk_public):     bstr
   2 (expiry):        uint (Unix ms, registration validity)
```

### 12.4.  Message Deposit

A sender who cannot reach a peer directly MAY deposit a message
at the peer's registered mailbox node:

```
MailboxDeposit (CBOR map):
   1 (recipient_nk):  bstr (recipient's Transport Key)
   2 (payload):       bstr (opaque encrypted message)
   3 (timestamp):     uint
   4 (ttl):           uint (seconds until message expires)
```

### 12.5.  Message Retrieval

When the recipient reconnects, it contacts its mailbox node and
retrieves pending messages:

```
MailboxRetrieve request:
   1 (tk_public):     bstr (prove ownership via transport auth)

MailboxRetrieve response:
   1 (messages):      array of MailboxDeposit
```

The mailbox node MUST delete delivered messages after the recipient
acknowledges receipt.

### 12.6.  Mailbox Selection

A user MAY advertise their mailbox node address in their Contact
Payload (as part of the addressing field) or DID document.  A user
MAY operate their own mailbox node, use a community-operated node,
or use a commercial mailbox service.

---

## 13.  Key Verification Fingerprint

### 13.1.  Overview

The Key Verification Fingerprint enables two users to verify
out-of-band that they hold each other's correct Identity Keys,
detecting man-in-the-middle key substitution.

Key verification is OPTIONAL.  Its necessity depends on the
contact exchange mechanism:

-  Physical proximity exchange (visual scanning, NFC tap): key
   verification MAY be skipped — MITM is not feasible.
-  Server-mediated or remote exchange: both parties MUST verify
   fingerprints through a separate trusted channel.
-  Exchange involving an intermediary (e.g., a reference/URL that
   points to a server-hosted payload): key verification SHOULD
   be performed regardless of physical proximity.

### 13.2.  Computation

```
sorted_keys = sort([IK_public_A, IK_public_B])  // lexicographic
digest = SHA-256(
   "H2H-KeyFingerprint-v1" || sorted_keys[0] || sorted_keys[1]
)
```

This protocol defines the digest computation but does NOT mandate
a specific human-readable encoding.  Applications choose the
encoding appropriate for their user experience.

RECOMMENDED encodings (implementations MUST support at least one):

**Numeric (8 digits)**:
```
fingerprint = sprintf("%08d", BE_uint(digest[0..3]) mod 100000000)
```
Example: "48371205"

**Numeric (6 digits)**:
```
fingerprint = sprintf("%06d", BE_uint(digest[0..3]) mod 1000000)
```
Example: "371205"

**Visual comparison**: Display the full digest as a visual pattern
(identicon, color grid) on both devices for side-by-side comparison.

### 13.3.  Properties

-  The fingerprint is deterministic: the same pair of Identity Keys
   always produces the same fingerprint.
-  The fingerprint is symmetric: computed identically by both parties
   (due to the lexicographic sort).
-  The fingerprint changes if and only if either party's Identity
   Key changes.

### 13.4.  Key Change Notification

Implementations SHOULD alert the user when a stored contact's
Identity Key changes.  This may indicate a new device (benign) or
a man-in-the-middle attack.

---

## 14.  Version Negotiation

### 14.1.  Procedure

During connection establishment, immediately after KBC exchange
(Section 9.2, step 6), both peers advertise supported protocol
versions on the Control Channel:

```
VersionAdvertise (CBOR map):
   1 (supported):     array of uint (e.g., [1, 2])
   2 (preferred):     uint (highest supported version)
```

Both peers select the highest version present in both supported
arrays.  If no common version exists, the connection MUST be
terminated with a VERSION_MISMATCH error.

### 14.2.  Version Numbers

| Version | Status        | Description                          |
|---------|---------------|--------------------------------------|
| 1       | This document | Initial H2H protocol specification   |

Future versions will be defined in updates to this document or
companion documents.

---

## 15.  Wire Formats

### 15.1.  General Frame Format

All H2H protocol messages on channels are wrapped in a CBOR-encoded
frame:

```
Frame (CBOR map):
   1 (channel):       uint (channel type, 0x00-0xFF)
   2 (msg_type):      uint (message type within channel)
   3 (payload):       bstr or CBOR map (channel-specific)
```

### 15.2.  Control Channel Message Types

| Type | Name              | Direction |
|------|-------------------|-----------|
| 0x01 | CHANNEL_OPEN      | Either    |
| 0x02 | CHANNEL_ACK       | Either    |
| 0x03 | CHANNEL_CLOSE     | Either    |
| 0x04 | KEEPALIVE         | Either    |
| 0x05 | CAPABILITY_ADV    | Either    |
| 0x06 | VERSION_ADV       | Either    |
| 0x07 | VERSION_SELECT    | Either    |
| 0x08 | KEY_ROTATION      | Either    |
| 0x09 | ERROR             | Either    |
| 0x10 | PROOF_CHALLENGE   | Either    |
| 0x11 | PROOF_RESPONSE    | Either    |
| 0x20 | KBC_EXCHANGE      | Either    |

### 15.3.  Error Codes

```
Error (CBOR map):
   1 (code):          uint
   2 (message):       tstr

Error codes:
   1   VERSION_MISMATCH     No common protocol version
   2   KBC_EXPIRED          Key Binding Certificate has expired
   3   KBC_INVALID          KBC signature verification failed
   4   IDENTITY_MISMATCH    IK does not match stored contact
   5   CHANNEL_UNSUPPORTED  Requested channel type not supported
   6   PAYLOAD_TOO_LARGE    Payload exceeds maximum size
   7   RATE_LIMITED         Too many requests
```

---

## 16.  Security Considerations

### 16.1.  Hardware Security Module Trust

The Human Proof property depends entirely on the integrity of
the hardware security module.  If an HSM is compromised (e.g., via
a hardware exploit or firmware vulnerability), the attacker can
sign arbitrary data without biometric authentication.

Implementations SHOULD check for known HSM vulnerabilities and
refuse to create identities on devices with compromised security
modules when feasible.

### 16.2.  Biometric Spoofing

The protocol is only as strong as the device's biometric
authentication.  An attacker who can spoof the biometric sensor
can produce valid Human Proofs.  This is a device-level
vulnerability, not a protocol-level one.

The protocol mitigates this through the in-person contact
exchange: identity is established based on physical presence, not
biometric appearance.  An attacker who can spoof the biometric
sensor still needs physical access to the target's device.

### 16.3.  Device Credential Fallback

When assurance level DEVICE is used (PIN/passcode fallback), the
security guarantees are weaker: anyone with knowledge of the device
credential can produce valid signatures.  Implementations MUST
clearly display the assurance level to receiving users.  Users
SHOULD treat DEVICE-level signatures with lower confidence than
PRESENCE-level signatures.

### 16.4.  Non-Repudiation

H2H signatures provide non-repudiation: a signed message is
permanent cryptographic proof that the holder of a specific
Identity Key authorized that specific message.  This is by design
and is a feature of the protocol.

Users and implementers should be aware that:

-  Signed messages can be presented as evidence in legal,
   compliance, or dispute-resolution contexts.
-  A sender cannot later deny having sent a signed message.
-  This property persists even after the Identity Key is destroyed.

Applications that require deniability (where neither party can
prove what the other said) should use the session-level signing
model (Section 10.1) where messages are protected by transport
encryption rather than individual signatures.

### 16.5.  Relay Server Threats

See Section 9.4.

### 16.6.  QR Code Replay

A captured QR code contains a valid Contact Payload.  Mitigations:

-  The timestamp field limits validity to 5 minutes.
-  The nonce prevents reuse of the same payload.
-  The attacker gains the public key and addressing info but cannot
   sign as the captured identity (they lack the HSM private key).
-  The attacker can attempt to connect to the captured peer but
   cannot authenticate (they lack a KBC for a trusted IK).

### 16.7.  Device Loss or Theft

If a device is lost or stolen:

-  The Identity Key requires authentication.  An attacker without
   the owner's biometric/credential cannot sign.
-  If the device was unlocked at time of theft, the attacker can
   sign until the device is wiped.
-  The owner SHOULD initiate Social Recovery (Section 8) on a new
   device.
-  Contacts SHOULD remove the compromised IK and accept the
   recovered identity through Social Recovery.

### 16.8.  Social Recovery Attacks

An attacker who compromises K recovery contacts can vouch for a
fraudulent replacement key.  Mitigations:

-  Users SHOULD choose recovery contacts carefully (geographically
   distributed, high-trust, difficult to simultaneously compromise).
-  Implementations SHOULD alert the user when a recovery is
   initiated and provide a grace period before the old key is
   replaced.
-  The recovered identity starts at Tier 2, not Tier 1, limiting
   the attacker's trust level.

### 16.9.  Deepfake Attacks on Video Verification (Tier 3)

Trust Tier 3 (video verified) is explicitly weaker than Tier 1
because real-time deepfakes exist.  Implementations MUST clearly
communicate that Tier 3 provides lower assurance.

### 16.10. Type Confusion

Without the structure_type discriminator (field 0), an attacker
could substitute a signed Contact Payload for a Key Binding
Certificate or vice versa, since both are COSE_Sign1 structures
signed by the same Identity Key.  Implementations MUST verify that
the structure_type matches the expected value before processing any
other fields.

### 16.11. Relay Attack on Human Proof

Without channel binding, an attacker between two peers could relay a
Human Proof Challenge to the victim's device and forward the
response to the challenger, falsely demonstrating presence.  The
channel_binding field in the Human Proof Response (Section 11)
binds the response to the specific connection, preventing this
attack.

### 16.12. Assurance Level Downgrade

An attacker who compromises the application layer (but not the
hardware key store) may attempt to produce DEVICE-level signatures
while claiming PRESENCE.  Since the assurance field is inside the
COSE_Sign1 payload, it is covered by the signature and cannot be
modified without invalidation.  However, a compromised application
could request DEVICE authentication and set assurance to PRESENCE
before signing.  This requires application-level compromise on the
signer's device.

### 16.13. Identity Key Revocation

This protocol does not define a formal key revocation mechanism.
If an Identity Key is compromised, the key holder SHOULD notify all
contacts via an out-of-band channel.  Contacts MUST remove the
compromised Identity Key and treat future signatures from that key
as unverified.  A formal revocation mechanism is deferred to a
companion document.

### 16.14. Quantum Computing

P-256 ECDSA is vulnerable to quantum computers running Shor's
algorithm.  When post-quantum signature schemes are standardized
for hardware security modules, the protocol SHOULD be updated.
The version negotiation mechanism (Section 14) enables this
migration.

---

## 17.  Privacy Considerations

### 17.1.  Metadata Exposure

Connection metadata is visible to:

-  Relay servers: source and destination Transport Keys, timing, volume.
-  Network observers: IP addresses of peers or relay connections.

The protocol does not specify metadata protection mechanisms.
Implementations requiring metadata privacy SHOULD layer additional
protections (VPN, Tor, mixnets).

### 17.2.  Identity Key in Cleartext Contact Payloads

The Contact Payload transmits the Identity Key public component in
cleartext via the proximity exchange mechanism.  Any party that
observes or captures a Contact Payload obtains a persistent, unique
identifier for the user.

Mitigations:

-  Contact Payloads are exchanged only during deliberate, in-person
   ceremonies, limiting the exposure surface.
-  The 5-minute validity window limits the useful lifetime of a
   captured payload.
-  The Transport Key (used for networking) is a separate key and does
   not directly reveal the Identity Key to network observers.

Applications with heightened privacy requirements SHOULD consider
exchanging Contact Payloads over an encrypted proximity channel.

### 17.3.  Transport Key Correlation

The Transport Key is visible to relay servers and network observers.
If the Transport Key is long-lived, observers can correlate a user's
activity across connections and time.

Implementations SHOULD rotate the Transport Key periodically
(Section 5.5) to limit the correlation window.  The trade-off is
that frequent rotation reduces reachability (contacts must learn
the new TK before they can connect).

### 17.4.  Contact Graph

A compromised device reveals the owner's contact list (stored
Identity Keys and DIDs).  This is inherent to contact-based
communication systems.

---

## 18.  References

### 18.1.  Normative References

[RFC2119]    Bradner, S., "Key words for use in RFCs to Indicate
             Requirement Levels", BCP 14, RFC 2119,
             DOI 10.17487/RFC2119, March 1997.

[RFC8174]    Leiba, B., "Ambiguity of Uppercase vs Lowercase in
             RFC 2119 Key Words", BCP 14, RFC 8174,
             DOI 10.17487/RFC8174, May 2017.

[FIPS186-5]  National Institute of Standards and Technology,
             "Digital Signature Standard (DSS)", FIPS PUB 186-5,
             February 2023.

[FIPS180-4]  National Institute of Standards and Technology,
             "Secure Hash Standard (SHS)", FIPS PUB 180-4,
             August 2015.

[RFC8949]    Bormann, C. and P. Hoffman, "Concise Binary Object
             Representation (CBOR)", RFC 8949,
             DOI 10.17487/RFC8949, December 2020.

[RFC9052]    Schaad, J., "CBOR Object Signing and Encryption
             (COSE): Structures and Process", RFC 9052,
             DOI 10.17487/RFC9052, August 2022.

[RFC9053]    Schaad, J., "CBOR Object Signing and Encryption
             (COSE): Initial Algorithms", RFC 9053,
             DOI 10.17487/RFC9053, August 2022.

[RFC6090]    McGrew, D., Igoe, K., and M. Salter, "Fundamental
             Elliptic Curve Cryptography Algorithms", RFC 6090,
             DOI 10.17487/RFC6090, February 2011.

[RFC5869]    Krawczyk, H. and P. Eronen, "HMAC-based
             Extract-and-Expand Key Derivation Function (HKDF)",
             RFC 5869, DOI 10.17487/RFC5869, May 2010.

[RFC5116]    McGrew, D., "An Interface and Algorithms for
             Authenticated Encryption", RFC 5116,
             DOI 10.17487/RFC5116, January 2008.

[RFC9000]    Iyengar, J., Ed. and M. Thomson, Ed., "QUIC: A
             UDP-Based Multiplexed and Secure Transport",
             RFC 9000, DOI 10.17487/RFC9000, May 2021.

[SEC2]       Certicom Research, "SEC 2: Recommended Elliptic
             Curve Domain Parameters", Version 2.0, January 2010.

[DID-CORE]   Sporny, M., Guy, A., Sabadello, M., and
             D. Reed, "Decentralized Identifiers (DIDs) v1.0",
             W3C Recommendation, July 2022.

[DID-PEER]   Hardman, D., "Peer DID Method Specification",
             https://identity.foundation/peer-did-method-spec/,
             2023.

### 18.2.  Informative References

[WebAuthn]   Balfanz, D., et al., "Web Authentication: An API for
             accessing Public Key Credentials Level 2", W3C
             Recommendation, April 2021.

[Apple-SE]   Apple Inc., "Apple Platform Security: Secure Enclave",
             https://support.apple.com/guide/security/
             secure-enclave-sec59b0b31ff, 2024.

[Android-KS] Google, "Android Keystore System",
             https://developer.android.com/privacy-and-security/
             keystore, 2024.

[Titan-M]    Google, "Titan M2 Security Chip",
             https://security.googleblog.com/, 2022.

[iroh]       n0, Inc., "iroh: peer-to-peer that just works",
             https://iroh.computer/, 2026.

[libp2p]     Protocol Labs, "libp2p",
             https://libp2p.io/, 2024.

[Signal]     Marlinspike, M. and T. Perrin, "The X3DH Key
             Agreement Protocol", 2016.

[DblRatchet] Marlinspike, M. and T. Perrin, "The Double Ratchet
             Algorithm", 2016.

[RFC6762]    Cheshire, S. and M. Krochmal, "Multicast DNS",
             RFC 6762, DOI 10.17487/RFC6762, February 2013.

[RFC6716]    Valin, JM., Vos, K., and T. Terriberry, "Definition
             of the Opus Audio Codec", RFC 6716,
             DOI 10.17487/RFC6716, September 2012.

[VC-DATA-MODEL] Sporny, M., et al., "Verifiable Credentials Data
             Model v1.1", W3C Recommendation, March 2022.

[DIDCOMM-V2] Curren, S., et al., "DIDComm Messaging v2.0",
             Decentralized Identity Foundation, 2022.

[RFC9420]    Barnes, R., et al., "The Messaging Layer Security
             (MLS) Protocol", RFC 9420, DOI 10.17487/RFC9420,
             July 2023.

---

## Appendix A.  P-256 Parameter Summary

| Parameter     | Value                                        |
|---------------|----------------------------------------------|
| Curve         | NIST P-256 (secp256r1, prime256v1)           |
| Field         | GF(p), p = 2^256 - 2^224 + 2^192 + 2^96 - 1|
| Order (n)     | 0xFFFFFFFF00000000FFFFFFFFFFFFFFFFBCE6FAADA7179E84F3B9CAC2FC632551 |
| Cofactor      | 1                                            |
| Key size      | 256 bits (32 bytes)                          |
| Signature     | DER-encoded (r, s), 70-72 bytes typically    |
| Public key    | 65 bytes uncompressed (0x04 || x || y)       |
| COSE alg id   | -7 (ES256)                                   |
| COSE kty      | 2 (EC2)                                      |
| COSE crv      | 1 (P-256)                                    |

---

## Appendix B.  Test Vectors

Test vectors for conformance testing will be published in a
companion document upon protocol stabilization.  Implementers
SHOULD generate interoperability test vectors for:

1.  COSE_Sign1 Contact Payload creation and verification
2.  COSE_Sign1 Key Binding Certificate creation and verification
3.  Key Verification Fingerprint computation for known key pairs
4.  COSE_Sign1 Human Proof challenge-response
5.  CBOR frame encoding and decoding

---

## Appendix C.  Relationship to Existing Standards

### C.1.  Passkeys (WebAuthn / FIDO2)

H2H uses the same cryptographic primitive as passkeys: P-256
ECDSA in a hardware security module, gated by biometrics.  The
distinction is directional.  Passkeys authenticate a user to a
server.  H2H authenticates a human to another human.

Passkeys are synced across devices for convenience.  H2H Identity
Keys are hardware-bound and non-exportable for non-repudiation.

### C.2.  Signal Protocol

The Signal Protocol provides end-to-end encryption and forward
secrecy.  H2H does not replace Signal Protocol; it addresses an
orthogonal problem.  Signal proves that a message was encrypted to
a specific key.  H2H proves that a specific human authorized the
message.  A companion document (H2H-CRYPTO) specifies how X3DH and
Double Ratchet can be layered on H2H.

### C.3.  Decentralized Identifiers (DIDs)

H2H adopts did:peer [DID-PEER] for identity representation,
enabling interoperability with the W3C Verifiable Credentials
ecosystem and DIDComm messaging.  H2H adds what DIDs alone do not
provide: hardware binding, biometric gating, and in-person trust
establishment.

### C.4.  ISO 18013-5 (Mobile Driving License)

ISO 18013-5 defines device engagement via QR/NFC + BLE for
presenting mobile identity documents.  H2H's contact exchange
transport patterns are similar, but H2H differs in being
bidirectional (both parties exchange identity) and self-sovereign
(no government issuer).

### C.5.  Enterprise Identity (Okta, Entra, Ping)

Enterprise identity providers authenticate users to services via
centralized directories.  H2H is complementary: an enterprise
deployment could bind H2H Identity Keys (via DIDs) to enterprise
identity records.

### C.6.  TLS (RFC 8446)

TLS authenticates servers to clients using X.509 certificates
issued by certificate authorities.  H2H authenticates humans to
humans using self-certified keys rooted in hardware and physical
presence.

### C.7.  KERI (Key Event Receipt Infrastructure)

KERI provides self-certifying identifiers with key rotation
history.  H2H's Transport Key rotation (Section 5.5) serves a similar
purpose but is scoped to the networking key only, while the
Identity Key is immutable (bound to hardware).

---

## Appendix D.  Future Work

The following topics are acknowledged but deferred to future
versions or companion documents:

1.  **Multi-device identity linking**: Allowing a user's multiple
    devices (each with their own Identity Key) to be recognized as
    belonging to the same person.

2.  **Group communication (H2H-GROUP)**: Group formation, membership
    management, and fan-out over pairwise H2H connections.
    Integration with MLS [RFC9420] for group key agreement.

3.  **End-to-end encryption (H2H-CRYPTO)**: X3DH and Double Ratchet
    adapted for P-256, layered on the H2H Data Transfer Layer.

4.  **Hardware attestation and verified assurance levels**: The base
    protocol's assurance values (DEVICE, PRESENCE) are application-
    reported claims.  A future extension would define higher assurance
    levels backed by hardware attestation evidence from the platform's
    root of trust (e.g., Android Key Attestation, Apple App Attest).
    This would enable peers to cryptographically verify:

    -  The Identity Key was generated inside a genuine hardware
       security module and is non-exportable (hardware-enforced).
    -  The biometric/device credential check actually occurred on
       real hardware, not merely reported by the application.
    -  At ceremony time, the Contact Object could include attestation
       evidence, allowing the receiving peer to verify IK hardware
       properties before accepting the contact.

    Hardware attestation introduces a trust dependency on the platform
    vendor (Apple, Google, etc.) for the specific claim that key store
    properties are genuine.  This does not replace the peer-to-peer
    ceremony for identity -- the ceremony remains the sole trust anchor
    for "who is this person."  The vendor trust is limited to "is this
    real hardware doing what it claims."  Applications choose whether
    to require hardware attestation based on their threat model.

5.  **Post-quantum migration**: Adoption of post-quantum signature
    schemes when standardized for hardware security modules.

6.  **Transport binding documents**: Formal bindings of the abstract
    transport requirements to specific implementations (iroh,
    libp2p, custom QUIC).

7.  **Metadata protection**: Onion routing or mixnet integration
    for connection metadata privacy.

---

## Authors' Addresses

Gamil Rodriguez
Rolab
Email: gamil@rolabtech.com
