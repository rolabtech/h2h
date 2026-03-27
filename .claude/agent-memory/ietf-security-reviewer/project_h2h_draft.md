---
name: H2H Presence Attestation Internet-Draft
description: IETF I-D draft-rodriguez-h2h-presence-attestation (renamed from presence-proof) defining peer-to-peer relationship-bound authorization using hardware-bound keys and COSE_Sign1 signing
type: project
---

H2H Presence Attestation is an individual submission (Experimental, category: exp) to the ISE, authored by Gamil Rodriguez / Rolab.

**Why:** Addresses the gap in peer-to-peer relationship-bound authorization without trusted third parties, motivated by generative AI impersonation threats. Renamed from "presence-proof" to "presence-attestation" between drafts.

**How to apply:** Review context includes:
- Protocol defines: KBO (IK signs TK), Contact Object (in-person exchange), Session Credentials (delegated per-message signing), Signed Messages, Presence Challenge/Response, Relationship Fingerprint
- All structures use COSE_Sign1 with ES256, CBOR maps with integer keys
- Two-key architecture: hardware-bound Identity Key (P-256) + software Transport Key
- Structure types: 0x01 KBO, 0x02 Contact, 0x03 SC, 0x04 SM, 0x10 Challenge, 0x11 Response
- CDDL schema in Appendix A is declared normative
- Docname/filename version mismatch: docname says -07 but filename says -00
- Three IANA registries requested: Structure Type, Assurance Value, Transport Key Algorithm
- Companion doc H2H-FULL covers transport, encryption, groups, recovery
- Second review performed 2026-03-26 for ISE submission
