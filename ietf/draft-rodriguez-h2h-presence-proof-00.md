---
title: "Presence Proof: Peer-to-Peer Identity Authentication Using Hardware-Bound Keys"
abbrev: "H2H Presence Proof"
category: exp
docname: draft-rodriguez-h2h-presence-proof-00
submissiontype: independent
number:
date: 2026-03-25
v: 3
area: Security
keyword:
  - presence proof
  - peer-to-peer authentication
  - hardware security module
  - biometric
  - COSE
venue:
  group:
  type:
  mail:
  arch:
  github: "rolabtech/h2h"
  latest: "https://github.com/rolabtech/h2h"

author:
  - fullname: Gamil Rodriguez
    organization: Rolab
    email: gamil@rolabtech.com

normative:
  RFC2119:
  RFC8174:
  RFC8949:
  RFC9052:
  RFC9053:
  FIPS186-5:
    title: "Digital Signature Standard (DSS)"
    author:
      org: National Institute of Standards and Technology
    date: 2023-02
    seriesinfo:
      FIPS: PUB 186-5
  FIPS180-4:
    title: "Secure Hash Standard (SHS)"
    author:
      org: National Institute of Standards and Technology
    date: 2015-08
    seriesinfo:
      FIPS: PUB 180-4

informative:
  RFC8446:
  RFC9420:
  WebAuthn:
    title: "Web Authentication: An API for accessing Public Key Credentials Level 2"
    author:
      - name: D. Balfanz
      - name: A. Czeskis
      - name: J. Hodges
      - name: J.C. Jones
      - name: M.B. Jones
      - name: A. Kumar
      - name: A. Liao
      - name: R. Lindemann
      - name: E. Lundberg
    date: 2021-04
    target: https://www.w3.org/TR/webauthn-2/
    seriesinfo:
      W3C: Recommendation
  FIDO2:
    title: "FIDO2: Web Authentication (WebAuthn)"
    author:
      org: FIDO Alliance
    date: 2024
    target: https://fidoalliance.org/fido2/
  Signal:
    title: "The X3DH Key Agreement Protocol"
    author:
      - name: M. Marlinspike
      - name: T. Perrin
    date: 2016
    target: https://signal.org/docs/specifications/x3dh/
  H2H-FULL:
    title: "Human-to-Human (H2H) Authentication and Data Transfer Protocol"
    author:
      - name: G. Rodriguez
    date: 2026-03
    target: https://github.com/rolabtech/h2h

---

--- abstract

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

--- middle

# Introduction

## Problem Statement

Existing identity authentication protocols authenticate users to
servers (WebAuthn {{WebAuthn}}, FIDO2 {{FIDO2}}), servers to
clients (TLS {{RFC8446}}), or protect message content from
eavesdroppers (Signal Protocol {{Signal}}, MLS {{RFC9420}}).  None
provide a mechanism for one individual to cryptographically prove
their identity to another individual in a peer-to-peer manner
without dependence on a trusted third party.

The increasing capability of generative AI to synthesize realistic
voice, video, and text impersonation creates an urgent need for
such a mechanism.  This document specifies a protocol that fills
this gap using cryptographic keys bound to hardware security modules
and gated by biometric or credential authentication, capabilities
already present in billions of consumer devices.

## Overview

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
    the chain: stored Identity Key (from in-person meeting) equals
    the Identity Key in the KBC, which signed the Node Key, which
    matches the Node Key of the transport peer.

5.  At any time, either peer may issue a Presence Proof Challenge
    requesting a fresh signed response that requires authentication
    on the challenged device.

## Scope

This document specifies:

-  The Key Binding Certificate format ({{kbc}})
-  The Contact Payload format ({{contact-payload}})
-  Contact exchange verification rules ({{contact-verification}})
-  Session Credentials for delegated per-message signing
   ({{session-credentials}})
-  The Presence Proof challenge-response mechanism ({{presence-proof}})
-  Safety Number computation ({{safety-numbers}})

This document does not specify:

-  Trust policy (how applications interpret verification results)
-  Transport layer selection or configuration
-  End-to-end message encryption
-  Group communication
-  Key recovery mechanisms
-  Application-layer data formats (messaging, file transfer, etc.)

These topics are addressed in companion documents; see {{H2H-FULL}}
for the complete protocol specification.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

## Terminology

Identity Key (IK):
: A P-256 ECDSA key pair {{FIPS186-5}} generated within a hardware
  key store.  The private key is non-exportable.  Access to signing
  operations requires authentication (biometric or device credential).

Node Key (NK):
: An asymmetric key pair used for peer-to-peer transport.  Generated
  in software.  The algorithm is determined by the transport layer
  and is not constrained by this document.

Hardware Key Store:
: A hardware component that generates, stores, and uses cryptographic
  keys without exposing private key material to software.  The key
  store MUST support P-256 key generation with non-exportable private
  keys and per-operation authentication gating.  Examples include
  secure enclaves, secure elements, and trusted execution
  environments.

Authentication Event:
: A user verification performed by the hardware key store or operating
  system immediately prior to a cryptographic operation.  An
  authentication event has one of two assurance levels:
  PRESENCE (biometric verification) or DEVICE (credential
  verification).

Key Binding Certificate (KBC):
: A COSE_Sign1 {{RFC9052}} structure binding an Identity Key to a
  Node Key.

Contact Payload (CP):
: A COSE_Sign1 structure containing a user's Identity Key, Node Key,
  and addressing information, exchanged during an in-person meeting.

Presence Proof:
: A COSE_Sign1 structure produced in response to a challenge,
  demonstrating that an authentication event occurred on the signing
  device at the time of signing.

Safety Number:
: A human-readable numeric string derived from two Identity Keys,
  used for out-of-band verification.

## CBOR and COSE Conventions

All structures in this document are encoded as CBOR {{RFC8949}} maps
with integer keys.  All signed structures use COSE_Sign1 {{RFC9052}}
with algorithm ES256 (ECDSA w/ SHA-256 using P-256, COSE algorithm
identifier -7).

Implementations MUST use deterministic CBOR encoding as defined in
Section 4.2 of {{RFC8949}} (Core Deterministic Encoding Requirements).
This ensures that the same payload always produces the same byte
sequence, which is critical for signature verification.

Every CBOR map payload in this protocol includes a structure_type
field (key 0) that uniquely identifies the payload type.  This
prevents type confusion attacks where a signed structure of one type
is substituted for another.

Identity Key public keys are encoded as COSE_Key {{RFC9053}}
structures with kty=2 (EC2), crv=1 (P-256).


# Architecture

## Two-Key Design

The protocol employs two asymmetric key pairs per user:

~~~
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
~~~

The Identity Key serves as the long-term trust anchor.  Its private
key never leaves the hardware key store, and every use requires an
authentication event.  These constraints make it a proxy for human
presence: a valid IK signature implies that a human completed an
authentication event on the device that holds the IK.

The Node Key handles high-frequency transport operations (handshakes,
keepalives) that cannot practically require per-operation
authentication.  The Key Binding Certificate ({{kbc}}) links the NK
to the IK, extending the trust anchor to the transport layer.

## Assurance Levels

Every signed structure in this protocol includes an assurance level
indicating the type of authentication event that authorized the
signing operation:

| Value | Label    | Meaning                                    |
|------:|----------|:-------------------------------------------|
|     1 | PRESENCE | Biometric authentication was performed     |
|     2 | DEVICE   | Device credential authentication performed |

Relying parties (receiving peers) MUST be informed of the assurance
level and SHOULD make it visible to the user.

## Identity Key Requirements

The hardware key store MUST enforce the following properties for
Identity Keys:

1.  Non-exportable: No interface SHALL allow extraction of the
    private key material, including backup, sync, or migration
    interfaces.

2.  Authentication-gated: Each signing operation MUST be preceded by
    a successful authentication event.  The key store MUST NOT cache
    authentication results across signing operations.

3.  Enrollment-sensitive: If the device's biometric enrollment data
    changes (e.g., a new fingerprint is added), the key store MUST
    invalidate the Identity Key.  This prevents an attacker who gains
    temporary physical access from adding their biometric and
    subsequently producing valid PRESENCE-level signatures.

## Node Key Rotation

The Node Key MAY be rotated to limit network-level correlation.
Rotation comprises:

1.  Generate a new Node Key pair.
2.  Sign a new KBC binding the IK to the new NK ({{kbc}}).
3.  Distribute the new KBC to connected peers.
4.  Discard old NK private material.

Peers that receive a new KBC for a known IK MUST verify the KBC
signature.  If valid, the peer updates its stored NK for that contact.


# Key Binding Certificate {#kbc}

## Format

The Key Binding Certificate is a COSE_Sign1 structure:

~~~
KBC = COSE_Sign1(
  protected: { 1 (alg): -7 (ES256) },
  unprotected: {},
  payload: KBC_payload
)
~~~

KBC_payload is a CBOR map:

| Key | Name           | CBOR Type | Description                       |
|----:|:---------------|:----------|:----------------------------------|
|   0 | structure_type | uint      | MUST be 0x01 (KBC)                |
|   1 | version        | uint      | MUST be 1                         |
|   2 | ik_public      | COSE_Key  | P-256 public key (kty=2, crv=1)   |
|   3 | nk_public      | bstr      | Transport-specific encoding       |
|   4 | nk_algorithm   | tstr      | e.g., "Ed25519", "P-256"          |
|   5 | timestamp      | uint      | Unix time in milliseconds         |
|   6 | expiry         | uint      | Unix time in milliseconds         |
|   7 | assurance      | uint      | 1=PRESENCE, 2=DEVICE              |

## Creation

The signer MUST:

1.  Populate all fields of KBC_payload.
2.  Set expiry to no more than 30 days after timestamp.
3.  Serialize KBC_payload as CBOR.
4.  Sign using COSE_Sign1 with the Identity Key.  This operation
    triggers an authentication event on the device.
5.  Record the assurance level of the authentication event in
    field 7.

## Verification

The verifier MUST:

1.  Decode the COSE_Sign1 structure.
2.  Verify that structure_type (field 0) is 0x01 (KBC).  If not,
    reject.  This prevents type confusion with other signed
    structures.
3.  Extract ik_public (field 2) from the payload.
4.  Verify the COSE_Sign1 signature using ik_public.
5.  Verify that the current time is >= timestamp (field 5) and
    < expiry (field 6).
6.  Verify that ik_public matches the expected Identity Key for this
    contact (from a prior contact exchange, {{contact-payload}}).
7.  Verify that nk_public (field 3) matches the Node Key of the
    transport peer as observed during the transport handshake.

If ANY check fails, the verifier MUST reject the KBC and terminate
the connection.

## Renewal

Implementations MUST re-sign the KBC before its expiry.  Renewal
requires a fresh authentication event, providing periodic proof of
continued human control.


# Contact Payload {#contact-payload}

## Format

The Contact Payload is a COSE_Sign1 structure:

~~~
CP = COSE_Sign1(
  protected: { 1 (alg): -7 (ES256) },
  unprotected: {},
  payload: CP_payload
)
~~~

CP_payload is a CBOR map:

| Key | Name           | CBOR Type | Description                       |
|----:|:---------------|:----------|:----------------------------------|
|   0 | structure_type | uint      | MUST be 0x02 (Contact Payload)    |
|   1 | version        | uint      | MUST be 1                         |
|   2 | ik_public      | COSE_Key  | P-256 public key                  |
|   3 | nk_public      | bstr      | Transport-specific encoding       |
|   4 | nk_algorithm   | tstr      | e.g., "Ed25519"                   |
|   5 | display_name   | tstr      | UTF-8, max 64 bytes               |
|   6 | timestamp      | uint      | Unix time in milliseconds         |
|   7 | nonce          | bstr      | 16 bytes, cryptographically random|
|   8 | addressing     | bstr      | Transport-specific, opaque        |
|   9 | assurance      | uint      | 1=PRESENCE, 2=DEVICE              |

## Creation

The signer MUST:

1.  Generate a 16-byte cryptographically random nonce (field 7).
2.  Set timestamp (field 6) to the current time.
3.  Populate all other fields.
4.  Serialize CP_payload as CBOR.
5.  Sign using COSE_Sign1 with the Identity Key.  This operation
    triggers an authentication event.
6.  Record the assurance level in field 9.

## Encoding for Exchange

The serialized COSE_Sign1 structure (a CBOR byte string) is
transmitted to the peer via a proximity exchange mechanism.

The proximity mechanism MUST satisfy the following requirements:

1.  Both parties are physically co-present during the exchange.
2.  The mechanism can transfer at least 512 bytes of binary data.
3.  The mechanism provides a human-verifiable indication that the
    exchange is occurring with the intended party (e.g., visual
    confirmation, physical proximity constraint).
4.  For mechanisms that do not provide visual confirmation of the
    peer (e.g., wireless discovery), the implementation MUST require
    explicit user confirmation of the peer's display_name before
    storing the contact.

The contact exchange MUST be bidirectional: both parties MUST
exchange and verify Contact Payloads before either party stores
the other as a verified contact.  An implementation MUST NOT store
a contact from a unidirectional exchange (where only one party
scanned the other's payload).

The choice of proximity mechanism is an implementation decision.
See Appendix A for examples.

## Size Considerations

A typical Contact Payload serializes to 300-500 bytes depending on
the display name length and addressing information size.
Implementations SHOULD ensure their chosen proximity mechanism
supports payloads of at least 512 bytes.


# Contact Exchange Verification {#contact-verification}

## Procedure

Upon receiving a serialized Contact Payload, the verifier MUST:

1.  Decode the CBOR byte string as a COSE_Sign1 structure.  If
    decoding fails, reject.

2.  Extract the CP_payload.

3.  Verify that structure_type (field 0) is 0x02 (Contact Payload).
    If not, reject.

4.  Verify that version (field 1) is supported.  If unsupported,
    reject.

5.  Verify that timestamp (field 6) is within a 5-minute window of
    the verifier's current time: abs(now - timestamp) <= 300000
    milliseconds.  If outside the window, reject.

6.  Verify that nonce (field 7) has not been seen in a previous
    Contact Payload within the current 5-minute window.  If
    duplicate, reject.

7.  Verify the COSE_Sign1 signature using ik_public (field 2).  If
    signature verification fails, reject.

8.  If all checks pass and the exchange is bidirectional (the local
    device has also transmitted its own Contact Payload to this
    peer), store the contact: ik_public, nk_public, nk_algorithm,
    display_name, addressing, assurance level, and the timestamp
    of the exchange.

## Clock Skew

The 5-minute timestamp window accommodates reasonable clock skew
between devices.  Implementations SHOULD use network time
synchronization (NTP) when available but MUST NOT reject payloads
solely due to minor clock differences within the window.

## Replay Protection

The nonce field prevents replay of a captured Contact Payload within
its validity window.  After the 5-minute window expires, the
timestamp check prevents replay.  Implementations MUST maintain a
set of seen nonces for at least 5 minutes and SHOULD discard them
after 10 minutes.


# Session Credentials {#session-credentials}

## Overview

A Session Credential is a short-lived signing delegation from the
Identity Key to an ephemeral Session Signing Key (SSK).  It enables
per-message authentication without requiring an authentication event
for each signing operation.

The delegation pattern is analogous to TLS Delegated Credentials
{{?RFC9345}}: a long-term trust anchor (the Identity Key) signs a
short-lived operational credential (the Session Credential), which
in turn authenticates individual operations (messages, file
transfers).

This creates a verifiable chain from any signed message back to
the Identity Key established during the in-person contact exchange:

~~~
Stored IK_public (from in-person meeting)
  |
  v  verifies
Session Credential (COSE_Sign1 signed by IK)
  |  contains SSK_public
  v  verifies
Signed Message (COSE_Sign1 signed by SSK)
~~~

## Session Credential Format

The Session Credential is a COSE_Sign1 structure signed by the
Identity Key:

~~~
SessionCredential = COSE_Sign1(
  protected: { 1 (alg): -7 (ES256) },
  unprotected: {},
  payload: SC_payload
)
~~~

SC_payload is a CBOR map:

| Key | Name            | CBOR Type | Description                           |
|----:|:----------------|:----------|:--------------------------------------|
|   0 | structure_type  | uint      | MUST be 0x03 (SESSION_CREDENTIAL)     |
|   1 | version         | uint      | MUST be 1                             |
|   2 | ssk_public      | COSE_Key  | Ephemeral P-256 public key            |
|   3 | ik_public       | COSE_Key  | Issuing Identity Key (for reference)  |
|   4 | peer_ik_hash    | bstr      | SHA-256 of peer's IK_public (scope)   |
|   5 | channel_binding | bstr      | SHA-256 of connection context         |
|   6 | timestamp       | uint      | Unix time in milliseconds             |
|   7 | expiry          | uint      | Unix time in milliseconds             |
|   8 | assurance       | uint      | 1=PRESENCE, 2=DEVICE                  |

## Creation

The signer MUST:

1.  Generate an ephemeral P-256 key pair (the Session Signing Key).
    The SSK is generated in software; it is not required to reside
    in the hardware key store.
2.  Compute peer_ik_hash as SHA-256 of the peer's stored IK_public.
3.  Compute channel_binding as SHA-256 of the connection context
    (e.g., concatenation of both peers' Node Key public components).
4.  Set expiry to no more than 1 hour after timestamp.
5.  Serialize SC_payload as deterministic CBOR.
6.  Sign using COSE_Sign1 with the Identity Key.  This operation
    triggers an authentication event on the device.
7.  Record the assurance level of the authentication event in
    field 8.

The SSK private key MUST be held in memory only and MUST NOT be
persisted to disk.  It MUST be discarded when the session expires
or the connection closes.

## Signed Message Format {#signed-message}

Messages authenticated by a Session Credential are wrapped in a
COSE_Sign1 structure signed by the SSK:

~~~
SignedMessage = COSE_Sign1(
  protected: { 1 (alg): -7 (ES256) },
  unprotected: {},
  payload: SM_payload
)
~~~

SM_payload is a CBOR map:

| Key | Name           | CBOR Type | Description                        |
|----:|:---------------|:----------|:-----------------------------------|
|   0 | structure_type | uint      | MUST be 0x04 (SIGNED_MESSAGE)      |
|   1 | message_id     | uint      | Unique per sender, monotonic       |
|   2 | timestamp      | uint      | Unix time in milliseconds          |
|   3 | content_type   | uint      | Application-defined (e.g., 1=text) |
|   4 | content        | bstr/tstr | Message content                    |

The SSK signing operation does NOT require an authentication event
(it uses the software-held ephemeral key).  This enables signing
every message without user interaction.

## Verification

To verify a Signed Message, the verifier MUST:

1.  Obtain the sender's Session Credential for this connection.
2.  Verify the Session Credential:
    a.  Decode the COSE_Sign1 structure.
    b.  Verify structure_type (field 0) is 0x03 (SESSION_CREDENTIAL).
    c.  Verify the signature using the sender's stored IK_public
        (from the contact exchange).
    d.  Verify that peer_ik_hash (field 4) matches SHA-256 of the
        verifier's own IK_public (confirms the credential is scoped
        to this relationship).
    e.  Verify that channel_binding (field 5) matches the current
        connection context.
    f.  Verify that the current time is between timestamp and expiry.
3.  Extract ssk_public (field 2) from the Session Credential.
4.  Verify the Signed Message:
    a.  Decode the COSE_Sign1 structure.
    b.  Verify structure_type (field 0) is 0x04 (SIGNED_MESSAGE).
    c.  Verify the signature using ssk_public.

If any check fails, the verifier MUST reject the message.

The Session Credential is typically verified once per session
(when first received) and the ssk_public is cached for subsequent
message verifications.

## Credential Lifecycle

-  A Session Credential SHOULD be created at connection
   establishment, immediately after KBC exchange.
-  The credential MUST expire no more than 1 hour after creation.
-  When a credential expires, the sender MUST create a new one,
   which requires a fresh authentication event.
-  If the connection closes, the SSK private material MUST be
   discarded immediately.
-  A peer MAY reject an expired Session Credential and request
   that the sender re-authenticate by issuing a Presence Proof
   Challenge ({{presence-proof}}).

## Security Properties

The Session Credential mechanism provides:

-  **Per-message authentication**: Every message has a standalone
   COSE_Sign1 signature verifiable against the Identity Key trust
   chain.
-  **Non-repudiation**: A signed message, combined with its Session
   Credential, constitutes independently verifiable proof that the
   Identity Key holder authorized the session during which the
   message was produced.
-  **Scope limitation**: The peer_ik_hash and channel_binding fields
   prevent a Session Credential from being used on a different
   connection or with a different peer.
-  **Time limitation**: The 1-hour expiry bounds the damage window
   if the SSK is compromised.
-  **Single authentication event**: Only one biometric or credential
   verification is needed per hour-long session, regardless of the
   number of messages signed.

The SSK is a software key and is therefore weaker than the
hardware-bound Identity Key.  An attacker who extracts the SSK
from memory can forge messages for the remaining validity period
of the Session Credential.  The short expiry (1 hour maximum)
limits this exposure.  Implementations that require stronger
per-message guarantees SHOULD sign messages directly with the
Identity Key, accepting the per-operation authentication cost.


# Presence Proof Challenge {#presence-proof}

## Overview

After a connection is established and authenticated via KBC exchange,
either peer MAY issue a Presence Proof Challenge to request on-demand
evidence that a human is currently present at the remote device.

## Challenge Format

The challenge is a CBOR map:

| Key | Name               | CBOR Type | Description                        |
|----:|:-------------------|:----------|:-----------------------------------|
|   0 | structure_type     | uint      | MUST be 0x10 (PROOF_CHALLENGE)     |
|   1 | challenge_nonce    | bstr      | 32 bytes, cryptographically random |
|   2 | required_assurance | uint      | 1=PRESENCE required, 2=DEVICE ok   |

The challenger MUST generate challenge_nonce using a
cryptographically secure random number generator.

The required_assurance field allows the challenger to demand a
specific assurance level.  If set to 1 (PRESENCE), the responder
MUST authenticate via biometric; a DEVICE-level response to a
PRESENCE-required challenge MUST be rejected by the challenger.
If set to 2 (DEVICE), either assurance level is acceptable.

## Response Format

The response is a COSE_Sign1 structure:

~~~
Response = COSE_Sign1(
  protected: { 1 (alg): -7 (ES256) },
  unprotected: {},
  payload: {
    0 : 0x11,              / structure_type: PROOF_RESPONSE /
    1 : challenge_nonce,    / echoed from challenge /
    2 : assurance,          / 1=PRESENCE, 2=DEVICE /
    3 : channel_binding     / SHA-256 hash of connection context /
  }
)
~~~

The responder signs the response with their Identity Key.  This
signing operation triggers an authentication event on the
responder's device.

The channel_binding field (key 3) MUST contain the SHA-256 hash of
a connection-specific value agreed upon during the transport
handshake (e.g., the concatenation of both peers' Node Key public
components, or a transport-provided session identifier).  This
prevents relay attacks where an attacker forwards a challenge to the
victim's device and relays the response to a different connection.

## Verification

The challenger MUST:

1.  Decode the COSE_Sign1 structure.
2.  Verify that structure_type (field 0) is 0x11 (PROOF_RESPONSE).
3.  Verify the signature using the stored Identity Key for the peer.
4.  Verify that challenge_nonce in the response matches the
    challenge_nonce that was sent.
5.  Verify that channel_binding (field 3) matches the expected
    SHA-256 hash of the current connection context.
6.  If required_assurance in the challenge was 1 (PRESENCE), verify
    that assurance (field 2) in the response is also 1.  If the
    response is DEVICE when PRESENCE was required, reject.

## Timing

-  Implementations SHOULD NOT issue challenges more frequently than
   once per 300 seconds (5 minutes).
-  The responder MUST produce a response within 60 seconds.
-  If no valid response is received within 60 seconds, the challenger
   SHOULD inform the user.
-  Failure to respond MUST NOT automatically terminate the connection.


# Safety Numbers {#safety-numbers}

## Computation

Safety Numbers enable two users to verify out-of-band that they hold
each other's correct Identity Keys.

Given two Identity Key public keys, IK_A and IK_B, encoded as
uncompressed P-256 points (65 bytes each, starting with 0x04):

~~~
sorted = sort_lexicographic(IK_A, IK_B)
digest = SHA-256("H2H-SafetyNumber-v1" || sorted[0] || sorted[1])
~~~

SHA-256 is as defined in {{FIPS180-4}}.

The first 30 bytes of the digest are divided into six 5-byte groups.
Each group is interpreted as a big-endian unsigned integer and reduced
modulo 100000.  The result is formatted as six groups of zero-padded
5-digit decimal numbers separated by spaces:

~~~
for i in 0..5:
  group[i] = BE_uint(digest[i*5 .. (i+1)*5]) mod 100000
output = sprintf("%05d %05d %05d %05d %05d %05d", group[0..5])
~~~

Example: "34521 08837 65092 18374 90125 63748"

## Properties

-  The Safety Number is deterministic: the same pair of Identity Keys
   always produces the same Safety Number.
-  The Safety Number is symmetric: it is the same regardless of which
   party computes it (due to the lexicographic sort).
-  The Safety Number changes if and only if either party's Identity
   Key changes.

## Verification Procedure

After contact exchange, both users SHOULD compare Safety Numbers
via an out-of-band channel (e.g., reading aloud, or comparing a
visual encoding displayed on both devices).

Implementations MUST provide a user interface for displaying Safety
Numbers.  Implementations SHOULD alert the user when a stored
contact's Safety Number changes.


# Security Considerations

## Attacker Model

This protocol considers the following attacker capabilities:

-  Network attacker: Can observe, modify, drop, replay, and inject
   messages on the network.  Mitigated by transport-layer encryption
   (assumed) and COSE_Sign1 signatures.

-  Remote attacker: Can attempt to impersonate a user without physical
   access to their device.  Mitigated by the hardware key store's
   non-exportable property.

-  Proximity attacker: Can observe or capture the Contact Payload
   during exchange (e.g., by photographing a visual encoding or
   intercepting a wireless transmission).  Mitigated by the 5-minute
   timestamp window, nonce uniqueness, and the absence of private key
   material in the Contact Payload.

This protocol does NOT protect against:

-  An attacker with physical access to an unlocked device.
-  Compromise of the hardware key store (hardware exploit, firmware
   vulnerability).
-  Biometric spoofing that defeats the device's biometric sensor.

## Hardware Key Store Trust

The Presence Proof property depends entirely on the integrity of the
hardware key store.  If the key store is compromised, an attacker can
produce valid signatures without authentication.  This is the
fundamental trust assumption of the protocol.

## Authentication Bypass

The DEVICE assurance level provides weaker guarantees than PRESENCE.
An attacker who learns the device credential can produce valid
DEVICE-level signatures.  Relying parties SHOULD treat DEVICE-level
signatures with lower confidence and MUST display the assurance level
to the user.

## Contact Payload Capture

A captured Contact Payload (e.g., photographed visual encoding or
intercepted wireless transmission) gives the attacker the victim's
public Identity Key and addressing information but the attacker
cannot:

-  Sign as the victim (lacks the hardware-bound private key).
-  Replay the payload after 5 minutes (timestamp check).
-  Replay within 5 minutes if the nonce was already seen.

The attacker CAN attempt to connect to the victim's transport
address but cannot complete H2H authentication (they lack a KBC
for an Identity Key the victim trusts).

## Key Binding Certificate Expiry

A 30-day KBC expiry limits the window during which a compromised
Node Key can be used.  Frequent NK rotation ({{architecture}})
reduces this window further.

## Non-Repudiation

COSE_Sign1 signatures produced by an Identity Key are non-repudiable:
they constitute cryptographic proof that the key holder authorized the
signed operation.  This is a deliberate property.  Applications should
inform users that signed artifacts may be presented as evidence.

## Safety Number Limitations

Safety Numbers are derived from a 30-byte truncation of SHA-256,
yielding 240 bits of collision resistance.  The human-readable format
(30 decimal digits) provides approximately 99.7 bits of entropy.
This is sufficient for manual comparison.  Implementations SHOULD
also support machine-readable comparison for higher assurance.

## Assurance Level Downgrade

An attacker who compromises the application layer (but not the
hardware key store) may attempt to forge the assurance field in
signed structures, claiming PRESENCE when only DEVICE authentication
occurred.  The assurance field is included in the COSE_Sign1 payload
and therefore covered by the signature; it cannot be modified without
invalidating the signature.

However, a compromised application could request DEVICE-level
authentication from the key store and then set the assurance field
to PRESENCE in the payload before signing.  This attack requires
compromising the application on the signer's device.  It is
mitigated by platform-level attestation mechanisms (out of scope
for this document) that can independently verify the authentication
type.

## Type Confusion

Without the structure_type discriminator (field 0), an attacker
could potentially substitute a signed Contact Payload for a Key
Binding Certificate or vice versa, since both are COSE_Sign1
structures signed by the same Identity Key.  The structure_type
field prevents this: verifiers MUST check that the structure_type
matches the expected value before processing any other fields.

## Identity Key Revocation

This document does not define a key revocation mechanism.  If an
Identity Key is compromised, the key holder SHOULD notify all
contacts via an out-of-band channel that the key is no longer
trustworthy.  Contacts MUST remove the compromised Identity Key
and treat any future signatures from that key as unverified.

A formal revocation mechanism (e.g., signed revocation statements,
revocation lists) is deferred to a companion document.

## Session Signing Key Compromise

The Session Signing Key (SSK) is a software key held in application
memory.  An attacker who can read process memory (e.g., via a memory
disclosure vulnerability or debugging access) can extract the SSK
and forge Signed Messages for the remaining validity of the Session
Credential.

Mitigations:

-  The Session Credential expiry MUST NOT exceed 1 hour, limiting
   the forgery window.
-  The SSK MUST be discarded when the session ends.
-  The peer_ik_hash and channel_binding fields prevent the stolen
   SSK from being used on a different connection or with a different
   peer.
-  Implementations SHOULD store the SSK in a memory region that is
   not swappable to disk (e.g., mlock on Linux, SecureEnclave
   memory on iOS if available for symmetric keys).
-  For operations requiring the strongest assurance, implementations
   SHOULD sign directly with the Identity Key rather than delegating
   to the SSK.

## Relay Attack on Presence Proof

Without channel binding, an attacker positioned between two peers
could relay a Presence Proof Challenge from Peer A to the victim's
device, collect the signed response, and forward it to Peer A,
making Peer A believe the victim is present on the attacker's
connection.  The channel_binding field in the Presence Proof
Response ({{presence-proof}}) prevents this by binding the response
to the specific connection context.

## Post-Quantum Considerations

P-256 ECDSA is vulnerable to quantum computers implementing Shor's
algorithm.  When post-quantum signature algorithms are available in
hardware key stores, the protocol SHOULD migrate.  The version field
in all structures enables this transition.

## Privacy of Identity Key in Contact Payloads

The Contact Payload transmits the Identity Key public component in
cleartext.  Any party that observes or captures a Contact Payload
obtains a persistent, unique identifier for the user.  This is an
inherent trade-off: the recipient needs the Identity Key to verify
the signature and to identify the sender in future connections.

Mitigations:

-  Contact Payloads are exchanged only during deliberate, in-person
   ceremonies, limiting the exposure surface.
-  The 5-minute validity window limits the useful lifetime of a
   captured payload.
-  The Node Key (used for networking) is a separate key that does
   not reveal the Identity Key to network observers.

Applications with heightened privacy requirements SHOULD consider
exchanging Contact Payloads over an encrypted proximity channel
(e.g., BLE with encryption enabled) to prevent passive observation.


# IANA Considerations

## H2H CBOR Map Keys

This document defines CBOR integer map keys for use in H2H protocol
structures.  A future revision may request establishment of an IANA
registry for these keys.

### KBC_payload Keys

| Key | Name           | CBOR Type |
|----:|:---------------|:----------|
|   0 | structure_type | uint      |
|   1 | version        | uint      |
|   2 | ik_public      | COSE_Key  |
|   3 | nk_public      | bstr      |
|   4 | nk_algorithm   | tstr      |
|   5 | timestamp      | uint      |
|   6 | expiry         | uint      |
|   7 | assurance      | uint      |

### CP_payload Keys

| Key | Name           | CBOR Type |
|----:|:---------------|:----------|
|   0 | structure_type | uint      |
|   1 | version        | uint      |
|   2 | ik_public      | COSE_Key  |
|   3 | nk_public      | bstr      |
|   4 | nk_algorithm   | tstr      |
|   5 | display_name   | tstr      |
|   6 | timestamp      | uint      |
|   7 | nonce          | bstr      |
|   8 | addressing     | bstr      |
|   9 | assurance      | uint      |

### Presence Proof Challenge Keys

| Key | Name               | CBOR Type |
|----:|:-------------------|:----------|
|   0 | structure_type     | uint      |
|   1 | challenge_nonce    | bstr      |
|   2 | required_assurance | uint      |

### Presence Proof Response Keys

| Key | Name            | CBOR Type |
|----:|:----------------|:----------|
|   0 | structure_type  | uint      |
|   1 | challenge_nonce | bstr      |
|   2 | assurance       | uint      |
|   3 | channel_binding | bstr      |

### SC_payload Keys (Session Credential)

| Key | Name            | CBOR Type |
|----:|:----------------|:----------|
|   0 | structure_type  | uint      |
|   1 | version         | uint      |
|   2 | ssk_public      | COSE_Key  |
|   3 | ik_public       | COSE_Key  |
|   4 | peer_ik_hash    | bstr      |
|   5 | channel_binding | bstr      |
|   6 | timestamp       | uint      |
|   7 | expiry          | uint      |
|   8 | assurance       | uint      |

### SM_payload Keys (Signed Message)

| Key | Name           | CBOR Type |
|----:|:---------------|:----------|
|   0 | structure_type | uint      |
|   1 | message_id     | uint      |
|   2 | timestamp      | uint      |
|   3 | content_type   | uint      |
|   4 | content        | bstr/tstr |

### Structure Type Values

| Value | Name               |
|------:|:-------------------|
|  0x01 | KBC                |
|  0x02 | CONTACT_PAYLOAD    |
|  0x03 | SESSION_CREDENTIAL |
|  0x04 | SIGNED_MESSAGE     |
|  0x10 | PROOF_CHALLENGE    |
|  0x11 | PROOF_RESPONSE     |

### Assurance Levels

| Value | Name     |
|------:|:---------|
|     1 | PRESENCE |
|     2 | DEVICE   |

--- back

# Implementation Considerations

## Platform Key Store Mapping

The abstract "hardware key store" requirement maps to the following
platform-specific implementations:

| Platform | Key Store           | P-256 | Auth Gating              |
|:---------|:--------------------|:------|:-------------------------|
| iOS      | Secure Enclave      | Yes   | BiometryCurrentSet       |
| Android  | StrongBox Keymaster | Yes   | UserAuthenticationRequired |
| Android  | TEE (fallback)      | Yes   | UserAuthenticationParameters |
| Windows  | TPM 2.0             | Yes   | Windows Hello            |
| Linux    | TPM 2.0             | Yes   | PKCS#11 with PIN         |

## QR Code Parameters

For the Contact Payload:

-  Mode: Binary
-  Minimum version: 15 (for ~350 byte payload)
-  Recommended error correction: M (15% recovery)
-  Maximum payload at Version 25, EC level M: 1,269 bytes

## BLE Exchange Parameters

For Bluetooth Low Energy contact exchange:

-  Service UUID: To be assigned (use a v4 UUID during development)
-  Characteristic: Single read/write characteristic containing the
   serialized COSE_Sign1 Contact Payload
-  MTU: Negotiate maximum MTU; the payload may span multiple ATT
   packets if necessary

# Acknowledgements
{:numbered="false"}

The H2H protocol builds on foundational work in FIDO2/WebAuthn
{{WebAuthn}}, the Signal Protocol {{Signal}}, and the CBOR/COSE
encoding standards {{RFC8949}} {{RFC9052}}.
