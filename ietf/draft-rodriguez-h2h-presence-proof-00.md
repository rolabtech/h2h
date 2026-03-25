# Presence Proof: Peer-to-Peer Identity Authentication Using Hardware-Bound Keys

**Document**: draft-rodriguez-h2h-presence-proof-00
**Status**: Individual Internet-Draft
**Intended Status**: Experimental
**Authors**: G. Rodriguez, Rolab
**Date**: 2026-03-24

---

## Abstract

This document defines a protocol for peer-to-peer human identity
authentication using hardware-bound cryptographic keys with
biometric or credential-gated access control.  The protocol
introduces the Presence Proof: a COSE_Sign1 signed artifact that
provides cryptographic evidence a specific, previously verified
individual authorized a specific operation.

The protocol specifies: a two-key architecture binding a hardware-
bound identity key to a software networking key via a Key Binding
Certificate; a Contact Payload format for in-person identity
exchange; a challenge-response mechanism for on-demand presence
verification; and a Safety Number scheme for out-of-band key
confirmation.

The protocol does not depend on any central authority, certificate
issuer, or identity provider.  Trust is rooted in physical human
presence during an in-person key exchange ceremony.

---

## Status of This Memo

This Internet-Draft is submitted in full conformance with the
provisions of BCP 78 and BCP 79.

Internet-Drafts are working documents of the Internet Engineering
Task Force (IETF).  Note that other groups may also distribute
working documents as Internet-Drafts.

This Internet-Draft will expire on September 24, 2026.

---

## Copyright Notice

Copyright (c) 2026 IETF Trust and the persons identified as the
document authors.  All rights reserved.

---

## Table of Contents

1.  Introduction
2.  Conventions and Definitions
3.  Architecture
4.  Key Binding Certificate
5.  Contact Payload
6.  Contact Exchange Verification
7.  Presence Proof Challenge
8.  Safety Numbers
9.  Security Considerations
10. IANA Considerations
11. References

---

## 1.  Introduction

### 1.1.  Problem Statement

Existing identity authentication protocols authenticate users to
servers (WebAuthn [WebAuthn], FIDO2 [FIDO2]), servers to clients
(TLS [RFC8446]), or protect message content from eavesdroppers
(Signal Protocol [Signal], MLS [RFC9420]).  None provide a
mechanism for one individual to cryptographically prove their
identity to another individual in a peer-to-peer manner without
dependence on a trusted third party.

The increasing capability of generative AI to synthesize realistic
voice, video, and text impersonation creates an urgent need for
such a mechanism.  This document specifies a protocol that fills
this gap using cryptographic keys bound to hardware security modules
and gated by biometric or credential authentication, capabilities
already present in billions of consumer devices.

### 1.2.  Overview

The protocol operates as follows:

1.  Each user generates an Identity Key inside a hardware key store
    with non-exportable private material and authentication-gated
    access control.

2.  Each user generates a Node Key in software for peer-to-peer
    networking and signs a Key Binding Certificate (KBC) linking
    the Identity Key to the Node Key.

3.  Two users meet in person and exchange Contact Payloads: signed
    structures containing their Identity Keys, Node Keys, and
    network addressing information.

4.  Later, when connecting remotely, peers exchange KBCs and verify
    the chain: stored Identity Key (from in-person meeting) =
    Identity Key in KBC = key that signed the Node Key = Node Key
    of the transport peer.

5.  At any time, either peer may issue a Presence Proof Challenge
    requesting a fresh signed response that requires authentication
    on the challenged device.

### 1.3.  Scope

This document specifies:

-  The Key Binding Certificate format (Section 4)
-  The Contact Payload format (Section 5)
-  Contact exchange verification rules (Section 6)
-  The Presence Proof challenge-response mechanism (Section 7)
-  Safety Number computation (Section 8)

This document does not specify:

-  Trust policy (how applications interpret verification results)
-  Transport layer selection or configuration
-  End-to-end message encryption
-  Group communication
-  Key recovery mechanisms
-  Application-layer data formats (messaging, file transfer, etc.)

These topics are addressed in companion documents.

---

## 2.  Conventions and Definitions

### 2.1.  Requirements Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY",
and "OPTIONAL" in this document are to be interpreted as described
in BCP 14 [RFC2119] [RFC8174] when, and only when, they appear in
all capitals, as shown here.

### 2.2.  Terminology

**Identity Key (IK)**: A P-256 ECDSA key pair [FIPS186-5] generated
   within a hardware key store.  The private key is non-exportable.
   Access to signing operations requires authentication (biometric
   or device credential).

**Node Key (NK)**: An asymmetric key pair used for peer-to-peer
   transport.  Generated in software.  The algorithm is determined
   by the transport layer and is not constrained by this document.

**Hardware Key Store**: A hardware component that generates, stores,
   and uses cryptographic keys without exposing private key material
   to software.  The key store MUST support P-256 key generation
   with non-exportable private keys and per-operation authentication
   gating.  Examples include (but are not limited to) secure
   enclaves, secure elements, and trusted execution environments.

**Authentication Event**: A user verification performed by the
   hardware key store or operating system immediately prior to a
   cryptographic operation.  An authentication event has one of two
   assurance levels:

   -  PRESENCE (1): Biometric verification (facial recognition,
      fingerprint, iris scan).  Provides evidence that a specific
      individual was physically present.

   -  DEVICE (2): Device credential verification (PIN, passcode,
      pattern).  Provides evidence that someone with knowledge of
      the credential was present, but does not identify a specific
      individual.

**Key Binding Certificate (KBC)**: A COSE_Sign1 [RFC9052] structure
   binding an Identity Key to a Node Key.

**Contact Payload (CP)**: A COSE_Sign1 structure containing a user's
   Identity Key, Node Key, and addressing information, exchanged
   during an in-person meeting.

**Presence Proof**: A COSE_Sign1 structure produced in response to a
   challenge, demonstrating that an authentication event occurred on
   the signing device at the time of signing.

**Safety Number**: A human-readable numeric string derived from two
   Identity Keys, used for out-of-band verification.

### 2.3.  CBOR and COSE Conventions

All structures in this document are encoded as CBOR [RFC8949] maps
with integer keys.  All signed structures use COSE_Sign1 [RFC9052]
with algorithm ES256 (ECDSA w/ SHA-256 using P-256, COSE algorithm
identifier -7).

Identity Key public keys are encoded as COSE_Key [RFC9053]
structures with kty=2 (EC2), crv=1 (P-256).

The CBOR map keys used in this document are summarized in
Section 10 (IANA Considerations).

---

## 3.  Architecture

### 3.1.  Two-Key Design

The protocol employs two asymmetric key pairs per user:

```
+--------------------------------------------------+
|  Identity Key (IK)                               |
|  Algorithm: P-256 ECDSA                          |
|  Storage: Hardware key store (non-exportable)    |
|  Access: Authentication-gated (PRESENCE/DEVICE)  |
|  Lifetime: Permanent (until device loss/reset)   |
|  Purpose: Prove "I am this specific individual"  |
+--------------------------------------------------+
         |
         | signs (Key Binding Certificate)
         v
+--------------------------------------------------+
|  Node Key (NK)                                   |
|  Algorithm: Transport-determined                 |
|  Storage: Software                               |
|  Access: No per-operation authentication         |
|  Lifetime: Rotatable                             |
|  Purpose: Peer-to-peer network identity          |
+--------------------------------------------------+
```

The Identity Key serves as the long-term trust anchor.  Its
private key never leaves the hardware key store, and every use
requires an authentication event.  These constraints make it a
proxy for human presence: a valid IK signature implies that a
human completed an authentication event on the device that holds
the IK.

The Node Key handles high-frequency transport operations
(handshakes, keepalives) that cannot practically require
per-operation authentication.  The Key Binding Certificate
(Section 4) links the NK to the IK, extending the trust anchor
to the transport layer.

### 3.2.  Assurance Levels

Every signed structure in this protocol includes an assurance
level indicating the type of authentication event that authorized
the signing operation:

| Value | Label    | Meaning                                    |
|-------|----------|--------------------------------------------|
| 1     | PRESENCE | Biometric authentication was performed     |
| 2     | DEVICE   | Device credential authentication performed |

Relying parties (receiving peers) MUST be informed of the assurance
level and SHOULD make it visible to the user.

### 3.3.  Identity Key Requirements

The hardware key store MUST enforce the following properties for
Identity Keys:

1.  **Non-exportable**: No interface SHALL allow extraction of the
    private key material, including backup, sync, or migration
    interfaces.

2.  **Authentication-gated**: Each signing operation MUST be
    preceded by a successful authentication event.  The key store
    MUST NOT cache authentication results across signing operations.

3.  **Enrollment-sensitive** (RECOMMENDED): If the device's
    biometric enrollment data changes (e.g., a new fingerprint is
    added), the key store SHOULD invalidate the Identity Key.

### 3.4.  Node Key Rotation

The Node Key MAY be rotated to limit network-level correlation.
Rotation comprises:

1.  Generate a new Node Key pair.
2.  Sign a new KBC binding the IK to the new NK (Section 4).
3.  Distribute the new KBC to connected peers.
4.  Discard old NK private material.

Peers that receive a new KBC for a known IK MUST verify the KBC
signature.  If valid, the peer updates its stored NK for that
contact.

---

## 4.  Key Binding Certificate

### 4.1.  Format

The Key Binding Certificate is a COSE_Sign1 structure:

```
KBC = COSE_Sign1(
  protected: {
    1 (alg): -7 (ES256)
  },
  unprotected: {},
  payload: KBC_payload
)
```

KBC_payload is a CBOR map:

```
KBC_payload = {
  1 : uint,       ; version (MUST be 1)
  2 : COSE_Key,   ; ik_public (P-256, kty=2, crv=1)
  3 : bstr,       ; nk_public (transport-specific encoding)
  4 : tstr,       ; nk_algorithm (e.g., "Ed25519", "P-256")
  5 : uint,       ; timestamp (Unix time in milliseconds)
  6 : uint,       ; expiry (Unix time in milliseconds)
  7 : uint        ; assurance (1=PRESENCE, 2=DEVICE)
}
```

### 4.2.  Creation

The signer MUST:

1.  Populate all fields of KBC_payload.
2.  Set expiry to no more than 30 days after timestamp.
3.  Serialize KBC_payload as CBOR.
4.  Sign using COSE_Sign1 with the Identity Key.
    This operation triggers an authentication event on the device.
5.  Record the assurance level of the authentication event in
    field 7.

### 4.3.  Verification

The verifier MUST:

1.  Decode the COSE_Sign1 structure.
2.  Extract ik_public (field 2) from the payload.
3.  Verify the COSE_Sign1 signature using ik_public.
4.  Verify that the current time is >= timestamp (field 5) and
    < expiry (field 6).
5.  Verify that ik_public matches the expected Identity Key for
    this contact (from a prior contact exchange, Section 5).
6.  Verify that nk_public (field 3) matches the Node Key of the
    transport peer as observed during the transport handshake.

If ANY check fails, the verifier MUST reject the KBC and
terminate the connection.

### 4.4.  Renewal

Implementations MUST re-sign the KBC before its expiry.  Renewal
requires a fresh authentication event, providing periodic proof
of continued human control.

---

## 5.  Contact Payload

### 5.1.  Format

The Contact Payload is a COSE_Sign1 structure:

```
CP = COSE_Sign1(
  protected: {
    1 (alg): -7 (ES256)
  },
  unprotected: {},
  payload: CP_payload
)
```

CP_payload is a CBOR map:

```
CP_payload = {
  1  : uint,       ; version (MUST be 1)
  2  : COSE_Key,   ; ik_public (P-256)
  3  : bstr,       ; nk_public
  4  : tstr,       ; nk_algorithm
  5  : tstr,       ; display_name (UTF-8, max 64 bytes)
  6  : uint,       ; timestamp (Unix ms)
  7  : bstr,       ; nonce (16 bytes, cryptographically random)
  8  : bstr,       ; addressing (transport-specific, opaque)
  9  : uint        ; assurance (1=PRESENCE, 2=DEVICE)
}
```

### 5.2.  Creation

The signer MUST:

1.  Generate a 16-byte cryptographically random nonce (field 7).
2.  Set timestamp (field 6) to the current time.
3.  Populate all other fields.
4.  Serialize CP_payload as CBOR.
5.  Sign using COSE_Sign1 with the Identity Key.
    This operation triggers an authentication event.
6.  Record the assurance level in field 9.

### 5.3.  Encoding for Exchange

The serialized COSE_Sign1 structure (a CBOR byte string) is
transmitted to the peer via a proximity mechanism.

Implementations MUST support encoding the serialized Contact
Payload as a binary-mode QR code for visual scanning.

Implementations MAY additionally support Bluetooth Low Energy
(BLE) proximity exchange, NFC tag reading, or other proximity
mechanisms.

### 5.4.  Size Considerations

A typical Contact Payload serializes to 300-500 bytes depending
on the display name length and addressing information size.  QR
code binary mode (Version 25, error correction level M) supports
up to 1,269 bytes, which is sufficient.

---

## 6.  Contact Exchange Verification

### 6.1.  Procedure

Upon receiving a serialized Contact Payload, the verifier MUST:

1.  Decode the CBOR byte string as a COSE_Sign1 structure.
    If decoding fails, reject.

2.  Extract the CP_payload.

3.  Verify that version (field 1) is supported.
    If unsupported, reject.

4.  Verify that timestamp (field 6) is within a 5-minute window
    of the verifier's current time:
    abs(now - timestamp) <= 300000 milliseconds.
    If outside the window, reject.

5.  Verify that nonce (field 7) has not been seen in a previous
    Contact Payload within the current 5-minute window.
    If duplicate, reject.

6.  Verify the COSE_Sign1 signature using ik_public (field 2).
    If signature verification fails, reject.

7.  If all checks pass, store the contact: ik_public, nk_public,
    nk_algorithm, display_name, addressing, assurance level, and
    the timestamp of the exchange.

### 6.2.  Clock Skew

The 5-minute timestamp window accommodates reasonable clock skew
between devices.  Implementations SHOULD use network time
synchronization (NTP) when available but MUST NOT reject payloads
solely due to minor clock differences within the window.

### 6.3.  Replay Protection

The nonce field prevents replay of a captured Contact Payload
within its validity window.  After the 5-minute window expires,
the timestamp check prevents replay.  Implementations MUST
maintain a set of seen nonces for at least 5 minutes and SHOULD
discard them after 10 minutes.

---

## 7.  Presence Proof Challenge

### 7.1.  Overview

After a connection is established and authenticated via KBC
exchange, either peer MAY issue a Presence Proof Challenge to
request on-demand evidence that a human is currently present at
the remote device.

### 7.2.  Challenge Format

```
Challenge = {
  1 : uint,   ; type (MUST be 0x10)
  2 : bstr    ; challenge_nonce (32 bytes, random)
}
```

The challenger MUST generate challenge_nonce using a
cryptographically secure random number generator.

### 7.3.  Response Format

```
Response = COSE_Sign1(
  protected: {
    1 (alg): -7 (ES256)
  },
  unprotected: {},
  payload: {
    1 : uint,   ; type (MUST be 0x11)
    2 : bstr,   ; challenge_nonce (echoed from challenge)
    3 : uint    ; assurance (1=PRESENCE, 2=DEVICE)
  }
)
```

The responder signs the response with their Identity Key.  This
signing operation triggers an authentication event on the
responder's device.

### 7.4.  Verification

The challenger MUST:

1.  Decode the COSE_Sign1 structure.
2.  Verify the signature using the stored Identity Key for the peer.
3.  Verify that challenge_nonce in the response matches the
    challenge_nonce that was sent.

### 7.5.  Timing

-  Implementations SHOULD NOT issue challenges more frequently than
   once per 300 seconds (5 minutes).
-  The responder MUST produce a response within 60 seconds.
-  If no valid response is received within 60 seconds, the
   challenger SHOULD inform the user.
-  Failure to respond MUST NOT automatically terminate the
   connection.

---

## 8.  Safety Numbers

### 8.1.  Computation

Safety Numbers enable two users to verify out-of-band that they
hold each other's correct Identity Keys.

Given two Identity Key public keys, IK_A and IK_B, encoded as
uncompressed P-256 points (65 bytes each, starting with 0x04):

```
sorted = sort_lexicographic(IK_A, IK_B)
digest = SHA-256(
  "H2H-SafetyNumber-v1" || sorted[0] || sorted[1]
)
```

The first 30 bytes of the digest are divided into six 5-byte
groups.  Each group is interpreted as a big-endian unsigned integer
and reduced modulo 100000.  The result is formatted as six groups
of zero-padded 5-digit decimal numbers separated by spaces:

```
for i in 0..5:
  group[i] = BE_uint(digest[i*5 .. (i+1)*5]) mod 100000
output = sprintf("%05d %05d %05d %05d %05d %05d", group[0..5])
```

Example: "34521 08837 65092 18374 90125 63748"

### 8.2.  Properties

-  The Safety Number is deterministic: the same pair of Identity
   Keys always produces the same Safety Number.
-  The Safety Number is symmetric: it is the same regardless of
   which party computes it (due to the lexicographic sort).
-  The Safety Number changes if and only if either party's Identity
   Key changes.

### 8.3.  Verification Procedure

After contact exchange, both users SHOULD compare Safety Numbers
by reading them aloud or by scanning a QR code encoding.

Implementations MUST provide a user interface for displaying
Safety Numbers.  Implementations SHOULD alert the user when a
stored contact's Safety Number changes.

---

## 9.  Security Considerations

### 9.1.  Attacker Model

This protocol considers the following attacker capabilities:

-  **Network attacker**: Can observe, modify, drop, replay, and
   inject messages on the network.  Mitigated by transport-layer
   encryption (assumed) and COSE_Sign1 signatures.

-  **Remote attacker**: Can attempt to impersonate a user without
   physical access to their device.  Mitigated by the hardware
   key store's non-exportable property: a valid IK signature
   requires the physical device.

-  **Proximity attacker**: Can observe or photograph a QR code
   during contact exchange.  Mitigated by the 5-minute timestamp
   window, nonce uniqueness, and the fact that the captured Contact
   Payload does not contain the private key.

This protocol does NOT protect against:

-  An attacker with physical access to the unlocked device.
-  Compromise of the hardware key store itself (e.g., via hardware
   exploit or firmware vulnerability).
-  Biometric spoofing that defeats the device's biometric sensor.

### 9.2.  Hardware Key Store Trust

The Presence Proof property depends entirely on the integrity of
the hardware key store.  If the key store is compromised, an
attacker can produce valid signatures without authentication.
This is the fundamental trust assumption of the protocol.

### 9.3.  Authentication Bypass

The DEVICE assurance level (credential-based authentication)
provides weaker guarantees than PRESENCE (biometric).  An
attacker who learns the device credential can produce valid
DEVICE-level signatures.  Relying parties SHOULD treat DEVICE-
level signatures with lower confidence and MUST display the
assurance level to the user.

### 9.4.  QR Code Capture

A photographed QR code contains a valid Contact Payload.  The
attacker gains the victim's public Identity Key and addressing
information but cannot:

-  Sign as the victim (lacks the hardware-bound private key).
-  Replay the payload after 5 minutes (timestamp check).
-  Replay the payload within 5 minutes if the victim's device has
   already seen the nonce (nonce check).

The attacker CAN attempt to connect to the victim's transport
address, but cannot complete H2H authentication (they lack a KBC
for an Identity Key the victim trusts).

### 9.5.  Key Binding Certificate Expiry

A 30-day KBC expiry limits the window during which a compromised
Node Key can be used.  However, an attacker who compromises the NK
private material can impersonate the user's network endpoint until
the KBC expires.  Frequent NK rotation (Section 3.4) reduces this
window further.

### 9.6.  Non-Repudiation

COSE_Sign1 signatures produced by an Identity Key are non-
repudiable: they constitute cryptographic proof that the key
holder authorized the signed operation.  This is a deliberate
property of the protocol.  Applications should inform users that
signed artifacts may be presented as evidence.

### 9.7.  Safety Number Limitations

Safety Numbers are derived from a 30-byte truncation of a SHA-256
hash, yielding 240 bits of collision resistance.  The human-
readable format (30 decimal digits) provides approximately 99.7
bits of entropy.  This is considered sufficient for manual
comparison but implementations SHOULD also support machine-
readable comparison (e.g., QR code scanning) for higher assurance.

### 9.8.  Post-Quantum Considerations

P-256 ECDSA is vulnerable to quantum computers implementing Shor's
algorithm.  When post-quantum signature algorithms are available in
hardware key stores, the protocol SHOULD migrate.  The version
field in all structures enables this transition.

---

## 10.  IANA Considerations

### 10.1.  COSE Header Parameters

This document requests no new COSE header parameters.  It uses
the existing ES256 algorithm identifier (-7) defined in [RFC9053].

### 10.2.  H2H CBOR Map Keys

This document defines CBOR map keys for use in H2H protocol
structures.  A future version of this document may request
registration of these keys in an appropriate IANA registry.

KBC_payload keys:

| Key | Name          | Type     |
|-----|---------------|----------|
| 1   | version       | uint     |
| 2   | ik_public     | COSE_Key |
| 3   | nk_public     | bstr     |
| 4   | nk_algorithm  | tstr     |
| 5   | timestamp     | uint     |
| 6   | expiry        | uint     |
| 7   | assurance     | uint     |

CP_payload keys:

| Key | Name          | Type     |
|-----|---------------|----------|
| 1   | version       | uint     |
| 2   | ik_public     | COSE_Key |
| 3   | nk_public     | bstr     |
| 4   | nk_algorithm  | tstr     |
| 5   | display_name  | tstr     |
| 6   | timestamp     | uint     |
| 7   | nonce         | bstr     |
| 8   | addressing    | bstr     |
| 9   | assurance     | uint     |

Presence Proof message types:

| Value | Name            |
|-------|-----------------|
| 0x10  | PROOF_CHALLENGE |
| 0x11  | PROOF_RESPONSE  |

Assurance levels:

| Value | Name     |
|-------|----------|
| 1     | PRESENCE |
| 2     | DEVICE   |

---

## 11.  References

### 11.1.  Normative References

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

### 11.2.  Informative References

[WebAuthn]   Balfanz, D., et al., "Web Authentication: An API for
             accessing Public Key Credentials Level 2", W3C
             Recommendation, April 2021.

[FIDO2]      FIDO Alliance, "FIDO2: Web Authentication (WebAuthn)",
             https://fidoalliance.org/fido2/, 2024.

[RFC8446]    Rescorla, E., "The Transport Layer Security (TLS)
             Protocol Version 1.3", RFC 8446,
             DOI 10.17487/RFC8446, August 2018.

[Signal]     Marlinspike, M. and T. Perrin, "The X3DH Key
             Agreement Protocol", 2016.

[RFC9420]    Barnes, R., et al., "The Messaging Layer Security
             (MLS) Protocol", RFC 9420, DOI 10.17487/RFC9420,
             July 2023.

[H2H-FULL]   Rodriguez, G., "Human-to-Human (H2H) Authentication
             and Data Transfer Protocol", H2H-PROTOCOL-02,
             March 2026.
             https://github.com/rolabtech/h2h

---

## Appendix A.  Implementation Considerations

### A.1.  Platform Key Store Mapping

The abstract "hardware key store" requirement (Section 2.2) maps
to the following platform-specific implementations:

| Platform | Key Store              | P-256 Support | Auth Gating                          |
|----------|------------------------|---------------|--------------------------------------|
| iOS      | Secure Enclave         | Yes           | kSecAccessControlBiometryCurrentSet  |
| Android  | StrongBox Keymaster    | Yes           | setUserAuthenticationRequired(true)  |
| Android  | TEE (fallback)         | Yes           | setUserAuthenticationParameters()    |
| Windows  | TPM 2.0                | Yes           | Windows Hello                        |
| Linux    | TPM 2.0                | Yes           | PKCS#11 with PIN                     |

### A.2.  QR Code Parameters

For the Contact Payload:

-  Mode: Binary
-  Minimum version: 15 (for ~350 byte payload)
-  Recommended error correction: M (15% recovery)
-  Maximum payload at Version 25, EC level M: 1,269 bytes

### A.3.  BLE Exchange Parameters

For Bluetooth Low Energy contact exchange:

-  Service UUID: To be assigned (use a v4 UUID during development)
-  Characteristic: Single read/write characteristic containing the
   serialized COSE_Sign1 Contact Payload
-  MTU: Negotiate maximum MTU; the payload may span multiple ATT
   packets if necessary

---

## Authors' Addresses

Gamil Rodriguez
Rolab
Email: dev@rolabtech.com
URI: https://github.com/rolabtech/h2h
