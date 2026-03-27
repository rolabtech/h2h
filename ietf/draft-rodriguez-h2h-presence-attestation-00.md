---
title: "Peer-to-Peer Presence Verification for Relationship-Bound Authorization"
abbrev: "P2P Presence Verification"
category: exp
docname: draft-rodriguez-h2h-presence-attestation-00
ipr: trust200902
submissiontype: independent
number:
date: 2026-03-26
v: 3
keyword:
  - peer-to-peer authentication
  - relationship-bound identity
  - presence attestation
  - COSE
  - CBOR
venue:
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
  RFC6973:
  RFC8610:
  RFC9345:
  RFC9420:
  RFC8446:
  RFC6347:
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

---

--- abstract

Existing protocols authenticate users to services, negotiate session
keys, or protect message content from eavesdroppers.  None verify
that a remote party is the same individual whose key was accepted
during an earlier in-person exchange and has just now physically
authorized a signature on the device holding that key.  Advances in
synthetic media increase the value of mechanisms that do not rely on
unauthenticated audio, video, or text for sensitive authorizations.

This document defines transport-independent CBOR- and COSE-based
objects that chain every remote interaction back to a bilateral
in-person contact exchange -- not a certificate authority, identity
provider, or key server.  The protocol specifies Key Binding Objects,
short-lived Session Credentials, replay-protected Signed Messages, a
Presence Challenge requiring fresh platform-mediated user
verification, and a Relationship Fingerprint for out-of-band key
confirmation.  It does not claim to prove biological humanness, legal
identity, or voluntary action.

--- middle

# Introduction

## Problem Statement

Existing protocols authenticate users to services (WebAuthn
{{WebAuthn}}, FIDO2 {{FIDO2}}), services to users (TLS
{{RFC8446}}), or protect message content from eavesdroppers
(Signal Protocol {{Signal}}, MLS {{RFC9420}}).  However, some
deployments require a narrower but different property: one peer
wishes to verify that the remote party is continuing a relationship
previously anchored by an in-person exchange, and that the remote
party has recently produced a local authorization event on the
device bound to that relationship.

Examples include high-trust messaging, approval workflows, operator
handoffs, and other interactions where account-level authentication
alone is insufficient.

Recent advances in synthetic media and automated impersonation
increase the value of mechanisms that reduce reliance on
unauthenticated audio, video, or text interactions for sensitive
approvals.  This document does not attempt to detect synthetic
media or prove biological humanness.  Instead, it specifies
interoperable signed objects and verification rules that bind
sensitive authorizations to a prior relationship ceremony and to a
fresh, platform-mediated local user-verification event.

## Design Goal

The goal of this protocol is not to prove a universally valid human
identity.  The goal is to let a relying peer verify all of the
following:

1.  A public key was previously accepted during an in-person exchange.
2.  The corresponding private key remains non-exportable and device-
    bound according to the local platform's key store semantics.
3.  The remote device recently produced a signature using that key,
    or a key delegated by it, following a local user-verification
    event reported by the platform.

## Overview

The protocol operates in four logical phases:

~~~
Phase 1           Phase 2            Phase 3           Phase 4
CONTACT           KEY BINDING        SESSION           DATA
EXCHANGE          PRESENTATION       DELEGATION        TRANSFER

+----------+    +----------+       +----------+     +----------+
| Meet in  |    | Connect  |       | IK signs |     | SSK signs|
| person,  |--->| remotely,|------>| ephemeral|---->| app      |
| exchange |    | present  |       | SSK via  |     | units    |
| Contact  |    | Key      |       | Session  |     |          |
| Objects  |    | Binding  |       |Credential|     |          |
+----------+    +----------+       +----------+     +----------+
 [user-verify]   [verify chain]    [user-verify]    [automatic]

At any point: Presence Challenge --> fresh IK signature
~~~

Phases 1 and 3 require platform-mediated user verification (e.g.,
biometric or PIN) because they use the hardware-bound Identity Key.
Phase 4 is automatic because the SSK is a software key delegated
during Phase 3 — no user interaction is needed for signing.

1.  **Contact Exchange**: Two peers meet in person and exchange
    Contact Objects -- signed structures containing their Identity
    Keys, Transport Keys, and addressing information.

2.  **Key Binding Presentation**: When connecting remotely, peers
    present Key Binding Objects and verify the trust chain: stored
    IK (from the in-person ceremony) = IK in binding = key that
    signed TK = TK of the transport peer.

3.  **Session Delegation**: Peers create Session Credentials (IK
    signs an ephemeral SSK) for efficient signing without repeated
    user-verification prompts.

4.  **Data Transfer**: Peers exchange Signed Messages (SSK signs
    application-defined units).  Every signed unit chains back to
    the in-person ceremony.

At any point, either peer MAY issue a Presence Challenge requesting
a fresh IK signature that requires a local user-verification event
on the challenged device.

## Scope

This document specifies:

-  The Contact Object format and verification rules ({{contact-object}})
-  The Key Binding Object format and verification rules ({{key-binding-object}})
-  The Session Credential format and verification rules
   ({{session-credential}})
-  The Signed Message format and verification rules
   ({{signed-message}})
-  The Presence Challenge and Presence Response formats
   ({{presence-challenge}})
-  A Relationship Fingerprint computation ({{relationship-fingerprint}})
-  Transport Key rotation guidance ({{tk-rotation}})
-  Protocol constants for object types, assurance values, and transport
   key algorithms

This document does not specify:

-  Proof-of-personhood
-  Legal, civil, or government identity
-  Transport protocol selection or configuration
-  End-to-end encryption
-  Multi-party relationships
-  Revocation or recovery
-  Application-layer trust policy
-  UX requirements beyond protocol-visible semantics

## Reuse and Interoperability Scope

The objects and verification procedures in this document are intended
for reuse across multiple peer-to-peer applications and transports
that require relationship-bound authorization.  This document does not
define a single application protocol or user experience.  Instead, it
standardizes compact object formats and verification rules that can be
embedded into messaging, approval, rendezvous, or comparable systems.

## Object Relationship Overview

The protocol objects have distinct roles and chain together as
follows:

~~~
Contact Object
  -> introduces IK + TK + rendezvous hints

Key Binding Object
  -> proves the current TK is bound to the stored IK

Session Credential
  -> allows the IK to delegate short-lived signing authority to an SSK

Signed Message
  -> carries application content signed by the delegated SSK

Presence Response
  -> re-asserts fresh, direct IK-based authorization on demand
~~~

## Trust Model Summary

This protocol relies on two external trust assumptions.

First, peers are assumed to have completed an earlier in-person
ceremony and to have accepted each other's Contact Objects during
that ceremony.

Second, relying parties trust the local platform's key store and
user-verification semantics.  A valid signature under the stored
identity key indicates only that the platform permitted use of that
key under its configured policy.  It does not, by itself, prove
biological humanness, voluntary action, or legal identity.

## End-to-End Message Flow {#e2e-flow}

The following ladder diagram shows a complete post-contact session
between two peers, Alice and Bob, who have already exchanged Contact
Objects in person.  Phases 2 through 4 occur each time the peers
reconnect remotely.

~~~
 Alice                                               Bob
   |                                                   |
   |  ============ Phase 2: Key Binding ============   |
   |                                                   |
   |  --- KBO(IK_A signs TK_A) ---------------------->|
   |                                                   |
   |<--------------------- KBO(IK_B signs TK_B) ---   |
   |                                                   |
   |  Both sides verify:                               |
   |    - IK in KBO matches stored IK from ceremony    |
   |    - TK in KBO matches transport peer key         |
   |    - KBO signature valid, not expired             |
   |                                                   |
   |  ========= Phase 3: Session Delegation ========   |
   |                                                   |
   |  Alice generates ephemeral SSK_A                  |
   |  IK_A signs Session Credential(SSK_A)             |
   |    [user-verification: biometric/PIN]             |
   |                                                   |
   |  --- SessionCredential(SSK_A_pub) -------------->|
   |                                                   |
   |          Bob generates ephemeral SSK_B            |
   |          IK_B signs Session Credential(SSK_B)     |
   |            [user-verification: biometric/PIN]     |
   |                                                   |
   |<------------- SessionCredential(SSK_B_pub) ---   |
   |                                                   |
   |  Both sides verify:                               |
   |    - SC signed by stored IK                       |
   |    - peer_ik_hash matches own IK                  |
   |    - SC not expired                               |
   |    - Cache SSK_pub for message verification       |
   |                                                   |
   |  =========== Phase 4: Data Transfer ===========   |
   |                                                   |
   |  --- SignedMessage(id=1, SSK_A signs content) --->|
   |                                                   |
   |      Bob verifies:                                |
   |        - SSK_A signature valid                    |
   |        - message_id > last seen (replay check)    |
   |        - Trust chain: SSK_A -> SC -> IK_A         |
   |                         -> ceremony               |
   |                                                   |
   |<--- SignedMessage(id=1, SSK_B signs content) ---  |
   |                                                   |
   |  --- SignedMessage(id=2, SSK_A signs content) --->|
   |                                                   |
   |<--- SignedMessage(id=2, SSK_B signs content) ---  |
   |                                                   |
   |  ====== Optional: Presence Challenge ==========   |
   |                                                   |
   |  --- PRESENCE_CHALLENGE {nonce} ----------------->|
   |                                                   |
   |          Bob's device prompts biometric/PIN       |
   |          IK_B signs nonce + channel_binding       |
   |                                                   |
   |<--------------- PRESENCE_RESPONSE {nonce, sig} ---|
   |                                                   |
   |  Alice verifies:                                  |
   |    - sig valid against stored IK_B                |
   |    - nonce matches                                |
   |    - assurance >= required                        |
   |                                                   |
   |  ====== Session Credential Renewal ============   |
   |                                                   |
   |  (When SC expires, peer creates a new SC          |
   |   requiring fresh user-verification; old SSK      |
   |   is discarded from memory)                       |
   |                                                   |
~~~

The diagram above is informative.  Normative requirements for each
object are specified in their respective sections ({{contact-object}},
{{key-binding-object}}, {{session-credential}}, {{signed-message}},
{{presence-challenge}}).


# Conventions and Definitions

{::boilerplate bcp14-tagged}

## Terminology

Identity Key (IK):
: A long-term P-256 ECDSA key pair {{FIPS186-5}} generated in a
  platform key store.  The private key is non-exportable.  Use of the
  private key requires local user verification according to platform
  policy.

Transport Key (TK):
: A transport-facing key pair used by a peer-to-peer transport or
  rendezvous mechanism.  The TK is bound to the IK by a Key Binding
  Object.  The algorithm is determined by the transport layer and is
  not constrained by this document.

Session Signing Key (SSK):
: A short-lived P-256 ECDSA key pair generated in software, delegated
  by the IK for signing application-defined units.  The SSK private key MUST
  be held in memory only and MUST NOT be persisted to disk.  Not all
  application data requires SSK signing; see {{applicability}} for
  guidance on when Signed Messages add value beyond transport
  authentication.

Platform Key Store:
: A platform component that generates, stores, and uses cryptographic
  keys without exposing private key material to application software.
  Examples include secure enclaves, secure elements, TPMs, and
  trusted execution environments.

Contact Object:
: A COSE_Sign1 object exchanged during an in-person ceremony that
  conveys the sender's Identity Key, Transport Key, and related
  metadata.

Key Binding Object (KBO):
: A COSE_Sign1 object that binds a Transport Key to a previously
  exchanged Identity Key.

Session Credential:
: A COSE_Sign1 object delegating signing authority from the IK to a
  short-lived SSK.  Analogous to TLS Delegated Credentials
  {{?RFC9345}}.

Signed Message:
: A COSE_Sign1 object wrapping application content, signed by the
  SSK and verifiable via the Session Credential trust chain.

Presence Response:
: A COSE_Sign1 object produced in response to a challenge and bound
  to a connection context.

Assurance Value:
: A protocol-visible value reported by the signer describing the type
  of local verification event used for a signing operation.  Assurance
  values are platform-mediated claims, not universal truth statements.

## Assurance Values

This document defines two assurance values.  Higher numeric values
indicate stronger assurance:

| Value | Label    | Meaning |
|------:|:---------|:--------|
|     1 | DEVICE   | The platform reported device credential or equivalent local verification for this signing operation. |
|     2 | PRESENCE | The platform reported biometric or equivalent presence-oriented local verification for this signing operation. |

A response with assurance value N satisfies a requirement for
assurance value M if and only if N >= M.

A verifier MUST treat these values as claims produced by the signing
platform.  A verifier MUST NOT interpret them as independent proof of
humanness, lack of coercion, or legal attribution.  A verifier MUST
treat the assurance value as an input to local policy, not as a
portable cross-platform security equivalence class.  Two
implementations that both emit PRESENCE are not necessarily providing
equivalent user-verification guarantees.

Relying parties (receiving peers) MUST be informed of the assurance
value and SHOULD make it visible to the user.

## CBOR and COSE Conventions

All protocol payloads in this document are encoded as CBOR maps
{{RFC8949}} with integer keys.  All signed objects are encoded as
COSE_Sign1 {{RFC9052}} with algorithm ES256 (ECDSA w/ SHA-256
using P-256, COSE algorithm identifier -7).

Implementations MUST use deterministic CBOR encoding as defined in
Section 4.2 of {{RFC8949}} (Core Deterministic Encoding
Requirements).  This ensures that the same payload always produces
the same byte sequence, which is critical for signature
verification.

Each payload includes a structure_type discriminator at key 0.  A
verifier MUST check this value before processing any other payload
semantics.  This prevents type confusion attacks where a signed
structure of one type is substituted for another.

Identity Key public keys are encoded as COSE_Key structures
{{RFC9053}} with kty=2 (EC2) and crv=1 (P-256).


# Identity and Binding Model

## Two-Key Model

The protocol uses a long-term Identity Key and a rotatable Transport
Key per peer.

~~~
+--------------------------------------------------+
|  Identity Key (IK)                               |
|  Algorithm: P-256 ECDSA                          |
|  Storage: Platform key store (non-exportable)    |
|  Access: User-verification-gated                 |
|  Lifetime: Permanent (until device loss/reset)   |
|  Purpose: Relationship anchor and trust root     |
+--------------------------------------------------+
         |
         | signs (Key Binding Object)
         v
+--------------------------------------------------+
|  Transport Key (TK)                              |
|  Algorithm: Transport-determined                 |
|  Storage: Software                               |
|  Access: No per-operation user verification      |
|  Lifetime: Rotatable                             |
|  Purpose: Peer-to-peer transport identity        |
+--------------------------------------------------+
~~~

The Identity Key is the relationship anchor.  It is exchanged during
the in-person ceremony and later used to verify continuity.  Its
private key never leaves the platform key store, and every use
requires a local user-verification event.

The Transport Key handles high-frequency transport operations
(handshakes, keepalives) that cannot practically require
per-operation user verification.  A Key Binding Object
({{key-binding-object}}) links the TK to the IK, extending
the trust anchor to the transport layer.

## Identity Key Requirements

An implementation claiming conformance for IK generation MUST
ensure:

1.  Non-exportable: No interface SHALL allow extraction of the
    private key material, including backup, sync, or migration
    interfaces.

2.  User-verification-gated: The platform MUST enforce local user
    verification before each IK signing operation, or before each
    operation within a bounded and explicitly documented platform
    policy window.  The platform MUST NOT cache verification
    results indefinitely.

3.  Enrollment-sensitive: If the device's biometric enrollment data
    changes (e.g., a new fingerprint is added), the platform SHOULD
    invalidate the Identity Key.  This prevents an attacker who
    gains temporary physical access from adding their biometric and
    subsequently producing valid PRESENCE-level signatures.

4.  The platform MUST expose sufficient information for the
    implementation to label the resulting signature with one of
    the defined assurance values.

This document does not require any specific hardware architecture.
It intentionally abstracts over secure enclaves, secure elements,
TPMs, TEEs, and comparable mechanisms.

## Transport Key Rotation {#tk-rotation}

The Transport Key MAY be rotated to limit network-level correlation.
Rotation comprises:

1.  Generate a new Transport Key pair.
2.  Sign a new Key Binding Object binding the IK to the new TK
    ({{key-binding-object}}).
3.  Distribute the new KBO to connected peers.
4.  Discard old TK private material.

Peers that receive a new KBO for a known IK MUST verify the KBO
signature.  If valid, the peer updates its stored TK for that
contact.

### Offline Rotation

When the Transport Key is rotated while a contact is offline, the
contact will be unable to locate the key holder on the network
using the previous Transport Key.  Implementations MUST address
this using one or more of the following mechanisms:

**Stable addressing (RECOMMENDED)**: The addressing field in the
Contact Object ({{contact-object}}, field 8) SHOULD contain
addressing information that survives Transport Key rotation (e.g.,
a relay server address or discovery service endpoint).  When a
contact reconnects using the stable address, the key holder
presents the current KBO containing the new Transport Key.

**Mailbox deposit**: If the implementation supports a mailbox or
store-and-forward mechanism, the key holder SHOULD deposit the
new KBO at their mailbox.  Offline contacts retrieve the updated
KBO upon reconnection.

**Overlap period**: The key holder MAY continue to accept
connections on the old Transport Key for a limited time after
rotation (RECOMMENDED: no more than 24 hours).  Connections on
the old Transport Key SHOULD be used only to deliver the new KBO
and redirect the contact.  After the overlap period, the old
Transport Key MUST be discarded.

Implementations MUST NOT rotate the Transport Key without a
mechanism for offline contacts to discover the new key.


# Contact Object {#contact-object}

## Purpose

The Contact Object is exchanged during an in-person ceremony and
anchors the long-term relationship.

## Format

The Contact Object is a COSE_Sign1 structure:

~~~
ContactObject = COSE_Sign1(
  protected: { 1 (alg): -7 (ES256) },
  unprotected: {},
  payload: Contact_payload
)
~~~

Contact_payload is a CBOR map:

| Key | Name           | CBOR Type | Description                       |
|----:|:---------------|:----------|:----------------------------------|
|   0 | structure_type | uint      | MUST be 0x02 (CONTACT_OBJECT)     |
|   1 | version        | uint      | MUST be 1                         |
|   2 | ik_public      | COSE_Key  | Sender Identity Key (P-256)       |
|   3 | tk_public      | bstr      | Sender Transport Key public bytes |
|   4 | tk_algorithm   | int       | Algorithm identifier for tk_public|
|   5 | display_name   | tstr      | UTF-8, max 64 bytes of UTF-8 content |
|   6 | timestamp      | uint      | Unix time in milliseconds         |
|   7 | nonce          | bstr      | 16 bytes, cryptographically random|
|   8 | addressing     | bstr      | Transport-specific, max 1024 bytes|
|   9 | assurance      | uint      | Assurance value for signing       |

The tk_public field MUST contain the raw public key bytes in the
canonical encoding for the algorithm identified by tk_algorithm.
See the transport key algorithm table ({{iana}}) for defined
encodings.

The addressing field is opaque to this specification.  It MUST carry
only the minimum information needed for rendezvous or re-contact in
the embedding application.  Implementations SHOULD prefer
application-scoped or short-lived addressing values where practical
and SHOULD avoid embedding identifiers that are broader or more stable
than operationally required.

## Creation

The sender MUST:

1.  Generate a 16-byte nonce (field 7) using a cryptographically
    secure random number generator (CSPRNG).
2.  Set timestamp (field 6) to the current time.
3.  Populate all other fields.
4.  Serialize Contact_payload as deterministic CBOR.
5.  Sign using COSE_Sign1 with the Identity Key.  This operation
    triggers a local user-verification event.
6.  Record the assurance value reported for that signing event in
    field 9.

## Verification {#contact-verification}

Upon receiving a Contact Object during the in-person ceremony, the
receiver MUST perform the following checks.  Checks are ordered so
that inexpensive validations precede cryptographic signature
verification:

1.  Decode the COSE_Sign1 structure.  If decoding fails, reject.

2.  Extract the Contact_payload.

3.  Verify that structure_type (field 0) is 0x02 (CONTACT_OBJECT).
    If not, reject.

4.  Verify that version (field 1) is supported.  If unsupported,
    reject.

5.  Verify that timestamp (field 6) is within a 5-minute window of
    the receiver's current time: abs(now - timestamp) <= 300000
    milliseconds.  If outside the window, reject.

6.  Verify that nonce (field 7) has not been seen in a previous
    Contact Object within the current 5-minute window.  If
    duplicate, reject.

7.  Verify the COSE_Sign1 signature using ik_public (field 2).  If
    signature verification fails, reject.

8.  If all checks pass and the exchange is bidirectional (the local
    device has also transmitted its own Contact Object to this
    peer), store the contact: ik_public, tk_public, tk_algorithm,
    display_name, addressing, assurance value, and the timestamp of
    the exchange.

A verifier MAY compute and compare a Relationship Fingerprint
({{relationship-fingerprint}}) as an additional defense if the
exchange path includes intermediaries.

## Ceremony Requirement

The Contact Object exchange MUST be bidirectional: both parties
MUST exchange and verify Contact Objects before either party stores
the other as a verified contact.  An implementation MUST NOT store
a contact from a unidirectional exchange.

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

The choice of proximity mechanism is an implementation decision.

## Size Considerations

A typical Contact Object serializes to 300-500 bytes depending on
the display name length and addressing information size.
Implementations SHOULD ensure their chosen proximity mechanism
supports payloads of at least 512 bytes.

## Clock Skew

The 5-minute timestamp window accommodates reasonable clock skew
between devices.  Implementations SHOULD use network time
synchronization (NTP) when available but MUST NOT reject payloads
solely due to minor clock differences within the window.

## Replay Protection

The nonce field prevents replay of a captured Contact Object within
its validity window.  After the 5-minute window expires, the
timestamp check prevents replay.  Implementations MUST maintain a
set of seen nonces for at least 5 minutes and SHOULD discard them
after 10 minutes.


# Key Binding Object {#key-binding-object}

## Purpose

The Key Binding Object proves continuity between the stored IK and
the currently offered TK.

## Format

The Key Binding Object is a COSE_Sign1 structure:

~~~
KeyBindingObject = COSE_Sign1(
  protected: { 1 (alg): -7 (ES256) },
  unprotected: {},
  payload: KBO_payload
)
~~~

KBO_payload is a CBOR map:

| Key | Name           | CBOR Type | Description                       |
|----:|:---------------|:----------|:----------------------------------|
|   0 | structure_type | uint      | MUST be 0x01 (KEY_BINDING)        |
|   1 | version        | uint      | MUST be 1                         |
|   2 | ik_public      | COSE_Key  | Identity Key (P-256)              |
|   3 | tk_public      | bstr      | Transport Key public bytes        |
|   4 | tk_algorithm   | int       | Algorithm identifier              |
|   5 | timestamp      | uint      | Unix time in milliseconds         |
|   6 | expiry         | uint      | Unix time in milliseconds         |
|   7 | assurance      | uint      | Assurance value for signing       |

The tk_public and tk_algorithm fields follow the same encoding
rules as in the Contact Object ({{contact-object}}).

## Creation

The signer MUST:

1.  Populate all fields of KBO_payload.
2.  Set expiry (field 6) to no more than 30 days (2,592,000,000
    milliseconds) after timestamp (field 5).
3.  Serialize KBO_payload as deterministic CBOR.
4.  Sign using COSE_Sign1 with the Identity Key.  This operation
    triggers a local user-verification event.
5.  Record the assurance value of the verification event in
    field 7.

## Verification

The verifier MUST perform the following checks.  Checks are ordered
so that inexpensive validations precede cryptographic signature
verification:

1.  Decode the COSE_Sign1 structure.  If decoding fails, reject.

2.  Verify that structure_type (field 0) is 0x01 (KEY_BINDING).
    If not, reject.

3.  Verify that the current time is >= timestamp (field 5) and
    < expiry (field 6).  If outside the window, reject.

4.  Extract ik_public (field 2) from the payload.

5.  Verify that ik_public matches the expected Identity Key for this
    contact (from a prior contact exchange, {{contact-object}}).
    If no match, reject.

6.  Verify that tk_public (field 3) matches the Transport Key of
    the transport peer as observed during the transport handshake.
    If no match, reject.

7.  Verify the COSE_Sign1 signature using ik_public.  If signature
    verification fails, reject.

If ANY check fails, the verifier MUST reject the KBO and terminate
the connection.

## Renewal

Implementations MUST re-sign the KBO before its expiry.  Renewal
requires a fresh user-verification event, providing periodic
evidence of continued platform-authorized use.


# Session Credential {#session-credential}

## Purpose

The Session Credential allows efficient signing of application-
defined units without requiring a fresh IK operation for each unit.
Not all data types require this signing; embedding applications
determine which content warrants Signed Message protection (see
{{applicability}}).  The delegation pattern is analogous to TLS
Delegated Credentials
{{?RFC9345}}: a long-term trust anchor (the IK) signs a short-lived
operational credential (the Session Credential), which in turn
authenticates individual operations (messages, file transfers).

This creates a verifiable chain from any signed message back to the
Identity Key established during the in-person contact exchange:

~~~
Stored IK_public (from in-person ceremony)
  |
  v  verifies
Session Credential (COSE_Sign1 signed by IK)
  |  contains SSK_public
  v  verifies
Signed Message (COSE_Sign1 signed by SSK)
~~~

## Format

The Session Credential is a COSE_Sign1 structure:

~~~
SessionCredential = COSE_Sign1(
  protected: { 1 (alg): -7 (ES256) },
  unprotected: {},
  payload: SC_payload
)
~~~

SC_payload is a CBOR map:

| Key | Name           | CBOR Type | Description                       |
|----:|:---------------|:----------|:----------------------------------|
|   0 | structure_type | uint      | MUST be 0x03 (SESSION_CREDENTIAL) |
|   1 | version        | uint      | MUST be 1                         |
|   2 | ssk_public     | COSE_Key  | Ephemeral P-256 public key        |
|   3 | ik_public      | COSE_Key  | Issuing Identity Key (reference)  |
|   4 | peer_ik_hash   | bstr      | SHA-256 of peer's IK_public       |
|   5 | timestamp      | uint      | Unix time in milliseconds         |
|   6 | expiry         | uint      | Unix time in milliseconds         |
|   7 | assurance      | uint      | Assurance value                   |

## Creation

The signer MUST:

1.  Generate an ephemeral P-256 key pair (the Session Signing Key).
    The SSK is generated in software; it is not required to reside
    in the platform key store.
2.  Compute peer_ik_hash as SHA-256 of the peer's IK_public encoded
    as an uncompressed P-256 point (65 bytes starting with 0x04).
    This encoding MUST match the encoding used in the Relationship
    Fingerprint computation ({{relationship-fingerprint}}).
3.  Set expiry to a value after timestamp.  Implementations MUST
    set a finite expiry (unlimited Session Credentials are
    prohibited).  A RECOMMENDED default is 1 hour.  Applications
    with different risk profiles MAY choose shorter durations
    (e.g., 5 minutes for financial transactions) or longer
    durations (e.g., 8 hours for a workday session).
4.  Serialize SC_payload as deterministic CBOR.
5.  Sign using COSE_Sign1 with the Identity Key.  This operation
    triggers a local user-verification event.
6.  Record the assurance value of the verification event in
    field 7.

The SSK private key MUST be held in memory only and MUST NOT be
persisted to disk.  It MUST be discarded when the session expires
or the connection closes.  Implementations SHOULD store the SSK
in a memory region that is not swappable to disk (e.g., mlock on
Linux).

## Verification

The verifier MUST:

1.  Decode the COSE_Sign1 structure.  If decoding fails, reject.

2.  Verify that structure_type (field 0) is 0x03
    (SESSION_CREDENTIAL).  If not, reject.

3.  Verify that the current time is between timestamp (field 5)
    and expiry (field 6).  If outside the window, reject.

4.  Verify that ik_public (field 3) matches the stored Identity
    Key for the peer.  If no match, reject.

5.  Verify that peer_ik_hash (field 4) matches SHA-256 of the
    verifier's own IK_public (confirms the credential is scoped
    to this relationship).  If no match, reject.

6.  Verify the COSE_Sign1 signature using the stored IK_public.
    If signature verification fails, reject.

The Session Credential is typically verified once per session
(when first received) and the ssk_public is cached for subsequent
message verifications.

## Credential Lifecycle

-  A Session Credential SHOULD be created at connection
   establishment, immediately after KBO exchange.
-  The credential MUST have a finite expiry.  The maximum duration
   is application-defined.  RECOMMENDED default: 1 hour.
-  When a credential expires, the sender MUST create a new one,
   which requires a fresh user-verification event.
-  If the connection closes, the SSK private material MUST be
   discarded immediately.
-  A peer MAY reject an expired Session Credential and request
   that the sender re-authenticate by issuing a Presence Challenge
   ({{presence-challenge}}).

## Security Note

Use of the SSK is weaker than direct use of the IK because the SSK
is a software key.  An attacker who extracts the SSK from memory
can forge messages for the remaining validity period of the Session
Credential.  The short expiry limits this exposure.

This document treats Session Credentials as an optimization and
not as an equivalent substitute for direct IK use.  Applications
SHOULD require direct IK-based Presence Responses for high-risk
actions such as key changes, privileged approvals, or other
operations where fresh platform-mediated authorization is
materially more valuable than signing throughput.

Mitigations for SSK compromise:

-  The Session Credential MUST have a finite expiry, limiting the
   forgery window.
-  The SSK MUST be discarded when the session ends.
-  The peer_ik_hash field prevents the stolen SSK from being used
   with a different peer.
-  Implementations SHOULD store the SSK in a non-swappable memory
   region.
-  For operations requiring the strongest assurance, implementations
   SHOULD sign directly with the Identity Key rather than delegating
   to the SSK.


# Signed Message {#signed-message}

## Applicability {#applicability}

Not all application data requires Signed Message wrapping.  When a
transport connection has been authenticated via Key Binding Object
verification ({{key-binding-object}}), the transport layer already
provides authentication for data in transit.  Signed Messages add
value when content must be verifiable independent of the transport
session -- for example, stored messages, forwarded content, or
operations requiring non-repudiation.  Real-time streams (audio,
video) over an authenticated direct connection typically do not
require per-unit signatures; the KBO-verified transport and
Presence Challenges provide sufficient assurance.

Embedding applications define which data types require Signed
Messages and which rely solely on transport authentication.

## Format

Application-defined units authenticated by a Session Credential are
wrapped in a COSE_Sign1 structure signed by the SSK.  A unit MAY be
a text message, a video keyframe, a file, a batch of events, or any
other application-meaningful boundary.  The signing granularity is
determined by the embedding application's security requirements:

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
|   1 | message_id     | uint      | See message ordering below         |
|   2 | timestamp      | uint      | Unix time in milliseconds          |
|   3 | content_type   | uint      | Application-defined (e.g., 1=text) |
|   4 | content        | bstr/tstr | Message content                    |

The SSK signing operation does NOT require a user-verification
event (it uses the software-held ephemeral key).  This enables
signing application units without user interaction.

## Message Ordering and Replay Protection {#message-ordering-and-replay-protection}

The message_id field MUST be unique among all Signed Messages
produced by the same sender within the scope of a single Session
Credential.  The sender MUST assign message_id values as a strictly
increasing sequence starting from 1.

### Reliable Transports

When the underlying transport guarantees in-order delivery (e.g.,
TCP, QUIC streams), the receiver MUST reject any Signed Message
whose message_id is less than or equal to the highest message_id
previously accepted from the same Session Credential.

### Unreliable Transports

When the underlying transport does not guarantee ordering (e.g.,
UDP, DTLS, WebRTC data channels in unordered mode), messages MAY
arrive out of order or be lost in transit.  In this case, the
receiver MUST maintain a sliding replay window of at least 64
entries and apply the following rules.  Implementations SHOULD
size the window to accommodate the expected maximum reordering
delay of the transport (e.g., a larger window for high-throughput
video streams than for infrequent text messages):

1.  If message_id > highest_seen: accept, update highest_seen.
2.  If message_id < highest_seen - window_size: reject (too old).
3.  If message_id is within the window and has NOT been seen:
    accept, mark as seen.
4.  If message_id is within the window and HAS been seen:
    reject (replay).

This approach follows the anti-replay mechanism defined in
Section 4.1.2.6 of DTLS {{RFC6347}}.

### Replay State Model

~~~
Reliable transport (strict):

  Session Credential SC1
     highest_message_id = 41

  Incoming message_id:
    42 -> accept, store 42
    42 -> reject (replay)
    40 -> reject (stale)

Unreliable transport (sliding window, size=64):

  Session Credential SC1
     highest_message_id = 100
     window covers 37..100

  Incoming message_id:
    101 -> accept, update highest to 101
     85 -> in window, not seen -> accept, mark seen
     85 -> in window, already seen -> reject (replay)
     30 -> below window -> reject (too old)
~~~

## Replay State

Verifiers MUST maintain replay state scoped to the validated Session
Credential.  At minimum, a verifier MUST remember the highest
accepted message_id (and, for unreliable transports, the sliding
window bitmap) for each active Session Credential until that
credential expires or is otherwise discarded.

If replay state is lost, the verifier MUST treat subsequently
received Signed Messages under the affected Session Credential as
potentially replayed.  In that case, the verifier MUST either reject
those messages or require a new Session Credential, or a direct
Identity-Key-based re-synchronization step, before accepting further
messages in that session.

## Verification

The full trust chain from a signed message back to the in-person
ceremony:

~~~
 In-person ceremony
       |
       v
 Stored IK_public -----> verifies Session Credential signature
                                |
                                v
                          SSK_public (extracted from SC)
                                |
                                v
                          verifies Signed Message signature
                                |
                                v
                          Message content is authenticated
                          as originating from the peer whose
                          IK was accepted in person
~~~

To verify a Signed Message, the verifier MUST:

1.  Obtain the sender's Session Credential for this connection.
2.  Verify the Session Credential (see {{session-credential}}).
3.  Extract ssk_public (field 2) from the Session Credential.
4.  Decode the Signed Message COSE_Sign1 structure.  If decoding
    fails, reject.
5.  Verify that structure_type (field 0) is 0x04 (SIGNED_MESSAGE).
    If not, reject.
6.  Apply the replay check for message_id (field 1) according to
    the transport type: strict monotonic for reliable transports,
    or sliding window for unreliable transports (see
    {{message-ordering-and-replay-protection}}).  If the check
    fails, reject.
7.  Verify the COSE_Sign1 signature using ssk_public.  If signature
    verification fails, reject.

If any check fails, the verifier MUST reject the message.


# Presence Challenge and Presence Response {#presence-challenge}

## Purpose

A Presence Challenge requests fresh evidence that the peer can
perform an IK operation on the current connection.

After a connection is established and authenticated via KBO exchange,
either peer MAY issue a Presence Challenge to request on-demand
evidence that the remote device has recently produced a locally
authorized signature.

~~~
 Alice (challenger)                        Bob (responder)
   |                                          |
   |  1. Generate 32-byte random nonce        |
   |                                          |
   |  --- PRESENCE_CHALLENGE {nonce, req} --> |
   |                                          |
   |          2. Bob's device prompts         |
   |             local user verification      |
   |                                          |
   |          3. IK signs nonce + binding     |
   |             [user-verification event]    |
   |                                          |
   |  <-- PRESENCE_RESPONSE {nonce, sig} ---  |
   |                                          |
   |  4. Alice verifies:                      |
   |     - sig valid against Bob's IK         |
   |     - nonce matches                      |
   |     - channel_binding matches            |
   |     - assurance >= required              |
   |                                          |
   |  Result: Bob's platform authorized       |
   |  an IK operation on this connection      |
   |                                          |
~~~

## Challenge Format

The Presence Challenge is an unsigned CBOR map:

| Key | Name               | CBOR Type | Description                        |
|----:|:-------------------|:----------|:-----------------------------------|
|   0 | structure_type     | uint      | MUST be 0x10 (PRESENCE_CHALLENGE)  |
|   1 | challenge_nonce    | bstr      | 32 bytes, cryptographically random |
|   2 | required_assurance | uint      | Minimum required assurance value   |

The challenger MUST generate challenge_nonce using a
cryptographically secure random number generator.

The required_assurance field allows the challenger to demand a
minimum assurance level.  If set to 2 (PRESENCE), the responder
MUST authenticate via biometric or equivalent; a DEVICE-level
response to a PRESENCE-required challenge MUST be rejected by the
challenger.  If set to 1 (DEVICE), either assurance level is
acceptable (since PRESENCE satisfies the DEVICE requirement).

## Response Format

The Presence Response is a COSE_Sign1 structure:

~~~
PresenceResponse = COSE_Sign1(
  protected: { 1 (alg): -7 (ES256) },
  unprotected: {},
  payload: PR_payload
)
~~~

PR_payload is a CBOR map:

| Key | Name            | CBOR Type | Description                        |
|----:|:----------------|:----------|:-----------------------------------|
|   0 | structure_type  | uint      | MUST be 0x11 (PRESENCE_RESPONSE)   |
|   1 | challenge_nonce | bstr      | Echoed from challenge              |
|   2 | assurance       | uint      | Reported assurance value           |
|   3 | channel_binding | bstr      | SHA-256 of connection context      |
|   4 | timestamp       | uint      | Unix time in milliseconds          |

The responder signs the response with their Identity Key.  This
signing operation triggers a local user-verification event.

The channel_binding field (key 3) MUST contain the SHA-256 hash of
a connection-specific value agreed upon during the transport
handshake.  This prevents relay attacks where an attacker forwards
a challenge to the victim's device and relays the response to a
different connection.

The mandatory-to-implement channel binding computation is:

~~~
cb_input = sort_lexicographic(TK_local, TK_remote)
channel_binding = SHA-256(cb_input[0] || cb_input[1])
~~~

where TK_local and TK_remote are the raw Transport Key public
bytes (as carried in tk_public fields) and sort_lexicographic
orders them by byte-wise lexicographic comparison.

Transport profiles MAY define alternative channel binding
computations (e.g., a transport-provided session identifier or TLS
Exporter value).  Both peers MUST agree on the computation method
before issuing or responding to challenges.  When no transport
profile is in effect, implementations MUST use the mandatory-to-
implement computation above.

## Verification

The challenger MUST:

1.  Decode the COSE_Sign1 structure.  If decoding fails, reject.

2.  Verify that structure_type (field 0) is 0x11
    (PRESENCE_RESPONSE).  If not, reject.

3.  Verify that challenge_nonce (field 1) matches the nonce sent.
    If no match, reject.

4.  Verify that timestamp (field 4) is within 60 seconds of the
    challenge issuance time.  If outside the window, reject.

5.  Verify that channel_binding (field 3) matches the expected
    SHA-256 hash of the current connection context.  If no match,
    reject.

6.  Verify that assurance (field 2) is >= required_assurance from
    the challenge.  If not, reject.

7.  Verify the COSE_Sign1 signature using the stored Identity Key
    for the peer.  If signature verification fails, reject.

## Timing

-  Implementations SHOULD NOT issue challenges more frequently than
   once per 300 seconds (5 minutes).
-  The responder MUST produce a response within 60 seconds.
-  If no valid response is received within 60 seconds, the
   challenger SHOULD inform the user.
-  Failure to respond MUST NOT automatically terminate the
   connection.

A successful Presence Response demonstrates only that the peer was
able to use the stored IK under the platform's configured policy on
the current connection.  It does not prove voluntariness or lack of
coercion.


# Relationship Fingerprint {#relationship-fingerprint}

## Overview

The Relationship Fingerprint enables two peers to verify out-of-band
that they hold each other's correct Identity Keys, detecting
man-in-the-middle key substitution.

Fingerprint verification is OPTIONAL.  Its necessity depends on the
contact exchange mechanism:

-  When Contact Objects are exchanged via a physical proximity
   mechanism where both parties can visually confirm the exchange,
   man-in-the-middle substitution is not feasible and fingerprint
   verification MAY be skipped.

-  When Contact Objects are exchanged via a channel that does not
   guarantee physical proximity (e.g., via a server, email, or
   messaging application), both parties MUST verify fingerprints
   through a separate trusted channel.

-  When the exchange mechanism involves an intermediary, fingerprint
   verification SHOULD be performed regardless of physical proximity.

## Computation

Given two Identity Key public keys, IK_A and IK_B, encoded as
uncompressed P-256 points (65 bytes each, starting with 0x04):

~~~
sorted = sort_lexicographic(IK_A, IK_B)
prefix = "H2H-RelationshipFingerprint-v1"
digest = SHA-256(prefix || sorted[0] || sorted[1])
~~~

SHA-256 is as defined in {{FIPS180-4}}.

This document defines only the digest computation.  Any human-
readable rendering is application-defined.

RECOMMENDED encodings (implementations MUST support at least one):

**Numeric (8 digits)**: Take the first 4 bytes of the digest,
interpret as a big-endian unsigned integer, reduce modulo
100,000,000, and zero-pad to 8 digits:

~~~
fingerprint = sprintf("%08d", BE_uint(digest[0..3]) mod 100000000)
~~~

Example: "48371205"

**Numeric (6 digits)**: Same approach, modulo 1,000,000:

~~~
fingerprint = sprintf("%06d", BE_uint(digest[0..3]) mod 1000000)
~~~

Example: "371205"

**Visual comparison**: Display the full digest as a visual pattern
on both devices for side-by-side comparison.

## Properties

-  The fingerprint is deterministic: the same pair of Identity Keys
   always produces the same fingerprint.
-  The fingerprint is symmetric: it is the same regardless of which
   party computes it (due to the lexicographic sort).
-  The fingerprint changes if and only if either party's Identity
   Key changes.

## Key Change Notification

Implementations SHOULD alert the user when a stored contact's
Identity Key changes (and therefore the fingerprint changes).  This
may indicate a new device (benign) or a man-in-the-middle attack.


# Security Considerations

## What This Protocol Does Provide

This protocol provides relationship continuity, binding of a
transport key to a previously accepted identity key, freshness
through challenge-response, optional delegated message
authentication, and per-unit replay protection via monotonic
message identifiers.

## What This Protocol Does Not Provide

This protocol does not provide:

-  proof-of-personhood
-  universal identity proof
-  protection against coercion
-  protection against a compromised platform key store
-  protection against a compromised application falsely describing
   the platform's assurance event unless the deployment adds
   attestation
-  unlinkability across contacts that receive the same IK
-  revocation or recovery

## Platform Trust

All meaningful security properties of this protocol depend on
platform behavior when using the Identity Key.  If the platform
allows the key operation without the expected local verification
event, the protocol cannot detect that failure by itself.

## Compromised Applications and Trusted UI

A compromised or malicious application can alter what is being
signed, misrepresent why a signing operation is requested, or
present application-specific content to the user in a misleading
way.  Unless the deployment provides trusted display, platform
attestation, or other out-of-band confirmation of signing intent,
a verifier MUST NOT assume that a valid protocol object implies
that the signer saw or understood the application semantics
associated with that object.

For the same reason, deployments SHOULD distinguish between
cryptographic authorization of a protocol object and user consent
to higher-level application meaning.  This document standardizes
only the former.

## Stolen, Unlocked, or Coerced Device State

If an attacker obtains control of an unlocked device, or if the
platform permits repeated key use within a local user-verification
window, this protocol may permit valid signatures that do not
reflect a new conscious act by the device holder.  Likewise, if
the device holder is coerced into satisfying local verification,
the protocol cannot detect that condition.

Accordingly, verifiers MUST treat successful verification as
evidence only that the platform authorized key use under its
current policy.  Verifiers MUST NOT infer voluntariness, freedom
from duress, or the absence of local compromise.

## Contact Object Capture

A captured Contact Object (e.g., photographed visual encoding or
intercepted wireless transmission) gives the attacker the victim's
public Identity Key and addressing information but the attacker
cannot:

-  Sign as the victim (lacks the device-bound private key).
-  Replay the payload after 5 minutes (timestamp check).
-  Replay within 5 minutes if the nonce was already seen.

The attacker CAN attempt to connect to the victim's transport
address but cannot complete authentication (they lack a KBO for
an Identity Key the victim trusts).

## Key Binding Object Expiry

A 30-day KBO expiry limits the window during which a compromised
Transport Key can be used.  Frequent TK rotation ({{tk-rotation}})
reduces this window further.

## Assurance Semantics and Policy

The assurance values defined by this document are signer-reported,
platform-mediated claims.  They are intentionally coarse and do not
capture the full variety of operating-system behavior, biometric
modes, passcode fallback rules, trusted-path properties, or reuse
windows.

An attacker who compromises the application layer (but not the
platform key store) may attempt to forge the assurance field in
signed structures, claiming PRESENCE (2) when only DEVICE (1)
verification occurred.  The assurance field is included in the
COSE_Sign1 payload and therefore covered by the signature; it
cannot be modified without invalidating the signature.  However, a
compromised application could request DEVICE-level verification
from the platform and then set the assurance field to PRESENCE in
the payload before signing.

Deployments that require stronger semantics SHOULD define local
policy for acceptable platforms, maximum verification age, session
credential lifetimes, and when direct IK use is required instead
of delegated signing.

## Relay Attack on Presence Response

Without channel binding, an attacker positioned between two peers
could relay a Presence Challenge from Peer A to the victim's device,
collect the signed response, and forward it to Peer A.  The
channel_binding field in the Presence Response ({{presence-challenge}})
prevents this by binding the response to the specific connection
context.

## Type Confusion

Without the structure_type discriminator (field 0), an attacker
could potentially substitute a signed Contact Object for a Key
Binding Object or vice versa, since both are COSE_Sign1 structures
signed by the same Identity Key.  The structure_type field prevents
this: verifiers MUST check that the structure_type matches the
expected value before processing any other fields.

## Relationship Fingerprint Limitations

The fingerprint is optional and its security depends on the encoding
chosen by the application.  An 8-digit numeric encoding provides
approximately 26 bits of collision resistance: an attacker would
need to generate approximately 67 million key pairs to find a
collision.  A 6-digit encoding provides approximately 20 bits.

For applications requiring stronger assurance, implementations
SHOULD support machine-readable comparison which can compare the
full 256-bit digest without human error.

## Privacy Considerations

### Identity Key as a Stable Identifier

The Contact Object transmits the Identity Key public component in
cleartext.  Any party that observes or captures a Contact Object
obtains a persistent, unique identifier for the user.

Mitigations:

-  Contact Objects are exchanged only during deliberate, in-person
   ceremonies, limiting the exposure surface.
-  The 5-minute validity window limits the useful lifetime of a
   captured object.
-  The Transport Key (used for networking) is a separate key that
   does not reveal the Identity Key to network observers.

### Relationship Correlation

If the same Identity Key is reused across multiple relationships or
applications, receivers that exchange identifiers or protocol
transcripts can determine that the same long-term key participated
in multiple relationships.  This may reveal social-graph structure
or cross-context linkage.

Deployments that treat such correlation as sensitive SHOULD consider
future profiles that use pairwise-derived relationship identifiers
or per-relationship keys.  Those mechanisms are not defined by this
document.

### Metadata Exposure

The Contact Object can also reveal metadata through fields such as
display_name, addressing, timestamps, and transport key continuity.
Even when application payloads are encrypted elsewhere, these fields
may reveal account aliases, transport reachability hints, device
migration timing, or relationship existence.

Applications using this protocol SHOULD minimize metadata
disclosure, avoid unnecessary stable identifiers, and avoid
embedding data in the addressing field that is broader than required
for rendezvous.

### Fingerprint Handling

Relationship Fingerprints are intended for optional out-of-band
comparison.  Applications SHOULD avoid displaying full high-entropy
fingerprints in contexts where shoulder surfing, screenshots, or
logging may create unnecessary privacy leakage.

See {{RFC6973}} for a general discussion of privacy considerations
in Internet protocols.

## Denial of Service via Presence Challenges

An authenticated peer could issue excessive Presence Challenges,
forcing repeated user-verification prompts.  The 300-second rate
limit ({{presence-challenge}}) mitigates this but does not eliminate
it.  Implementations SHOULD allow users to disable Presence Challenge
responses from specific contacts.

## Post-Quantum Considerations

P-256 ECDSA is vulnerable to quantum computers implementing Shor's
algorithm.  When post-quantum signature algorithms are available in
platform key stores, the protocol SHOULD migrate.  The version field
in all structures enables this transition.

## Forward Secrecy of Authentication

This protocol does not provide forward secrecy of authentication.
If the Identity Key private key is ever extracted from the platform
key store (despite hardware protections), all Key Binding Objects,
Session Credentials, and Presence Responses previously signed by
that key can be verified by the attacker.  Past signed objects
remain verifiable indefinitely.

Deployments that require forward secrecy of authentication evidence
SHOULD layer an ephemeral key agreement protocol (e.g., TLS 1.3
{{RFC8446}} or a Diffie-Hellman-based transport) beneath this
protocol.  This document's objects provide authentication, not
confidentiality or forward secrecy.

## Session Credential Expiry and In-Flight Messages

A Session Credential has a finite expiry.  Messages signed by the
SSK before the credential expires but received after expiry present
a time-of-check/time-of-use consideration.  Verifiers SHOULD accept
a Signed Message if the message's own timestamp (field 2) falls
within the Session Credential's validity window, even if the message
arrives after the credential has expired, provided the arrival delay
is within a locally configured grace period (RECOMMENDED: no more
than 60 seconds past expiry).  After the grace period, verifiers
MUST reject the message and require a new Session Credential.

## Device Loss and Recovery

When a device is lost or destroyed, the Identity Key is permanently
unrecoverable (non-exportable from the platform key store).  All
contacts verified against that Identity Key become orphaned.

Revocation and recovery are out of scope for this document, but that
omission has concrete deployment consequences.  In such cases,
deployments will typically require an out-of-band re-verification
ceremony, explicit peer approval of a new Identity Key, or an
application-defined recovery mechanism.

Similarly, if an Identity Key is believed to be compromised, this
document does not define a protocol-native revocation object or
distribution method.  Applications MUST therefore define local policy
for suspending trust in a stored key and for handling replacement or
termination of the affected relationship.

## Terminology Caution

Implementations and profiles built on this document SHOULD avoid
using terms such as "proves human" or "non-repudiation" without
additional legal and deployment analysis.  The protocol produces
cryptographic attribution artifacts within the protocol trust model;
broader claims may not hold.


# IANA Considerations {#iana}

This document has no IANA actions.

This document defines protocol constants for structure types,
assurance values, and transport key algorithms.  These values are
defined in the body of this document and in the CDDL schema
({{cddl}}).  They are summarized here for convenience.

## Structure Type Values

| Value | Name               |
|------:|:-------------------|
|  0x01 | KEY_BINDING        |
|  0x02 | CONTACT_OBJECT     |
|  0x03 | SESSION_CREDENTIAL |
|  0x04 | SIGNED_MESSAGE     |
|  0x10 | PRESENCE_CHALLENGE |
|  0x11 | PRESENCE_RESPONSE  |

## Assurance Values

| Value | Name     |
|------:|:---------|
|     1 | DEVICE   |
|     2 | PRESENCE |

## Transport Key Algorithms

| COSE alg | Name    | tk_public Encoding                             |
|---------:|:--------|:-----------------------------------------------|
|       -8 | EdDSA   | 32 bytes, Ed25519 public key (RFC 8032 S5.1.5) |
|       -7 | ES256   | 65 bytes, uncompressed P-256 point (0x04 \|\| x \|\| y) |

Implementations MUST support at least one of the above transport
key algorithms.

A future Standards Track document may request the creation of IANA
registries for these value spaces if broader interoperability
requires coordinated allocation.

--- back

# CDDL Schema {#cddl}

The following CDDL {{?RFC8610}} schema formally defines all protocol
structures.  This schema is normative.

~~~cddl
; ============================================================
; P2P Presence Verification CDDL Schema
; Reference: draft-rodriguez-h2h-presence-attestation-00
; ============================================================

; -- Common types --

assurance_value = 1 / 2       ; 1=DEVICE, 2=PRESENCE

; P-256 public key as COSE_Key (kty=EC2, crv=P-256)
P256_COSE_Key = {
  1  => 2,                    ; kty: EC2
  -1 => 1,                   ; crv: P-256
  -2 => bstr .size 32,       ; x coordinate
  -3 => bstr .size 32,       ; y coordinate
}

; ============================================================
; COSE_Sign1 wrappers
; Each signed structure is a COSE_Sign1 with ES256 algorithm.
; The payload is a CBOR-encoded map defined below.
; ============================================================

H2H_COSE_Sign1<PayloadType> = #6.18([
  protected: bstr .cbor {1 => -7},  ; alg: ES256
  unprotected: {},
  payload: bstr .cbor PayloadType,
  signature: bstr,
])

KeyBindingObject    = H2H_COSE_Sign1<KBO_payload>
ContactObject       = H2H_COSE_Sign1<Contact_payload>
SessionCredential   = H2H_COSE_Sign1<SC_payload>
SignedMessage        = H2H_COSE_Sign1<SM_payload>
PresenceResponse    = H2H_COSE_Sign1<PR_payload>

; ============================================================
; Payload definitions (no additional keys permitted)
; ============================================================

; -- Key Binding Object --

KBO_payload = {
  0 => 1,                    ; structure_type: KEY_BINDING (fixed)
  1 => 1,                    ; version (fixed for v1)
  2 => P256_COSE_Key,        ; ik_public
  3 => bstr,                 ; tk_public: raw key bytes
  4 => int,                  ; tk_algorithm: COSE alg id
  5 => uint,                 ; timestamp: Unix ms
  6 => uint,                 ; expiry: Unix ms, max +30d
  7 => assurance_value,      ; assurance
}

; -- Contact Object --

Contact_payload = {
  0 => 2,                    ; structure_type: CONTACT_OBJECT (fixed)
  1 => 1,                    ; version (fixed for v1)
  2 => P256_COSE_Key,        ; ik_public
  3 => bstr,                 ; tk_public: raw key bytes
  4 => int,                  ; tk_algorithm: COSE alg id
  5 => tstr .size (0..64),   ; display_name: UTF-8, max 64 bytes
  6 => uint,                 ; timestamp: Unix ms
  7 => bstr .size 16,        ; nonce: 16 random bytes
  8 => bstr .size (0..1024), ; addressing: transport-specific
  9 => assurance_value,      ; assurance
}

; -- Session Credential --

SC_payload = {
  0 => 3,                    ; structure_type: SESSION_CREDENTIAL
  1 => 1,                    ; version (fixed for v1)
  2 => P256_COSE_Key,        ; ssk_public: ephemeral P-256 key
  3 => P256_COSE_Key,        ; ik_public: issuing Identity Key
  4 => bstr .size 32,        ; peer_ik_hash: SHA-256 of peer IK
  5 => uint,                 ; timestamp: Unix ms
  6 => uint,                 ; expiry: Unix ms
  7 => assurance_value,      ; assurance
}

; -- Signed Message --

SM_payload = {
  0 => 4,                    ; structure_type: SIGNED_MESSAGE (fixed)
  1 => uint,                 ; message_id: >= 1, strictly increasing
  2 => uint,                 ; timestamp: Unix ms
  3 => uint,                 ; content_type: application-defined
  4 => bstr / tstr,          ; content
}

; -- Presence Challenge (not signed, plain CBOR) --

PresenceChallenge = {
  0 => 16,                   ; structure_type: PRESENCE_CHALLENGE
  1 => bstr .size 32,        ; challenge_nonce: 32 random bytes
  2 => assurance_value,      ; required_assurance
}

; -- Presence Response --

PR_payload = {
  0 => 17,                   ; structure_type: PRESENCE_RESPONSE
  1 => bstr .size 32,        ; challenge_nonce: echoed
  2 => assurance_value,      ; assurance
  3 => bstr .size 32,        ; channel_binding: SHA-256
  4 => uint,                 ; timestamp: Unix ms
}
~~~


# Implementation Considerations

The following implementation guidance is informative and does not alter
the protocol requirements in this document.

## Platform Key Store Mapping

The abstract "platform key store" requirement commonly maps to
mechanisms such as Secure Enclave, StrongBox or TEE-backed Android
keys, TPM-backed Windows keys, or TPM- and PKCS#11-based Linux
deployments.  Implementations MUST verify that their chosen mechanism
meets the Identity Key requirements in Section 3.2 regardless of the
platform name used in product documentation.

## Payload Size Limits

Implementations MUST enforce maximum payload sizes to prevent
resource-exhaustion attacks.  A deployment SHOULD set explicit limits
for each structure type before CBOR decoding or signature verification.
If a profile does not define stricter limits, a reasonable default is
4,096 bytes for Contact Objects, Key Binding Objects, and Session
Credentials; 262,144 bytes for Signed Messages; and 1,024 bytes for
Presence Challenges and Presence Responses.

# Acknowledgements
{:numbered="false"}

This work builds on COSE {{RFC9052}}, CBOR {{RFC8949}}, WebAuthn
{{WebAuthn}}, the FIDO2 framework {{FIDO2}}, delegated-credential
patterns {{?RFC9345}}, the Signal Protocol {{Signal}}, and prior
secure messaging designs.
