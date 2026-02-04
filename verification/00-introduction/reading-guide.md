# Reading Guide

This documentation is structured for different reading paths depending on your role and goals.

---

## By Role

### Mobile Developer (Holder Wallet)

You are building the wallet application that holds credentials and presents them to Verifiers.

**Start here:**
1. [01 Architecture](../01-architecture/) — Understand module structure
2. [02 Presentation Request](../02-presentation-request/) — How requests arrive at the wallet
3. [06 Credential Selection](../06-credential-selection/) — Matching and user consent
4. [07 Holder Binding](../07-holder-binding/) — Cryptographic proofs for presentation
5. [08 Presentation Response](../08-presentation-response/) — Building and sending the VP

**Key implementation files (EUDI Android):**
- `WalletCorePresentationController.kt` — Central presentation orchestrator
- `PresentationRequestInteractor.kt` — Request handling and document filtering
- `RequestTransformer.kt` — Credential/claim transformation

### Backend Developer (Verifier Service)

You are building the service that requests and validates credential presentations.

**Start here:**
1. [04 Authorization Request](../04-authorization-request/) — Constructing OID4VP requests
2. [05 Presentation Definition](../05-presentation-definition/) — Specifying credential requirements
3. [09 Verifier Validation](../09-verifier-validation/) — Validating received presentations
4. [03 Verifier Metadata](../03-verifier-metadata/) — Trust establishment

### Architect / Technical Lead

You are making decisions about which implementation approach to follow.

**Start here:**
1. [01 Architecture](../01-architecture/) — High-level comparison
2. [12 Comparative Summary](../12-comparative-summary/) — Decision framework
3. [10 Transport Modes](../10-transport-modes/) — Remote vs proximity trade-offs
4. [11 Cryptography](../11-cryptography/) — Security properties

### Security Engineer

You are reviewing the security properties of the verification flow.

**Start here:**
1. [07 Holder Binding](../07-holder-binding/) — Proof-of-possession mechanisms
2. [11 Cryptography](../11-cryptography/) — Algorithms and attack prevention
3. [09 Verifier Validation](../09-verifier-validation/) — Signature and revocation checks
4. [04 Authorization Request](../04-authorization-request/) — Nonce and replay protection

---

## By Implementation

### EUDI Wallet Focus

If your target is the European Digital Identity Wallet:

- [EUDI Architecture](../01-architecture/eudi-architecture.md)
- [EUDI Presentation Request](../02-presentation-request/eudi-implementation.md)
- [EUDI Credential Selection](../06-credential-selection/eudi-implementation.md)

Key characteristics:
- Native Android (Kotlin) and iOS (Swift) implementations
- OID4VP v1.0 compliance
- ISO 18013-5 proximity support (BLE/NFC)
- Hardware-backed key attestation
- MSO-MDOC and SD-JWT-VC formats

### Procivis ONE Focus

If your target is the Procivis ONE wallet:

- [Procivis Architecture](../01-architecture/procivis-architecture.md)
- [Procivis Presentation Request](../02-presentation-request/procivis-implementation.md)

Key characteristics:
- React Native cross-platform
- Presentation Definition V1 and V2 support
- Remote Secure Element (RSE) signing
- Flexible protocol configuration
- ISO mDL protocol support

### Affinidi Focus

If your target is the Affinidi ecosystem:

- [Affinidi Architecture](../01-architecture/affinidi-architecture.md)
- [Affinidi Iota Framework](../02-presentation-request/affinidi-implementation.md)

Key characteristics:
- Cloud-based Affinidi Vault
- Iota Framework for verification
- PEX-based queries
- WebSocket and Redirect modes
- TDK (Trust Development Kit) integration

---

## By Credential Format

### SD-JWT-VC

If you are working primarily with SD-JWT Verifiable Credentials:

- [Presentation Definition — SD-JWT constraints](../05-presentation-definition/sd-jwt-queries.md)
- [Holder Binding — SD-JWT key binding](../07-holder-binding/sd-jwt-binding.md)
- [Cryptography — SD-JWT selective disclosure](../11-cryptography/sd-jwt-presentation.md)

### MSO-MDOC / ISO 18013-5

If you are working with mobile documents (mDL):

- [Transport Modes — Proximity (BLE/NFC)](../10-transport-modes/proximity-mode.md)
- [Presentation Definition — mDoc constraints](../05-presentation-definition/mdoc-queries.md)
- [Holder Binding — Device authentication](../07-holder-binding/mdoc-device-auth.md)

### W3C Verifiable Credentials (JSON-LD)

If you are working with W3C JSON-LD credentials:

- [Presentation Definition — JSON-LD paths](../05-presentation-definition/jsonld-queries.md)
- [Holder Binding — Linked data proofs](../07-holder-binding/jsonld-proofs.md)

---

## Quick Reference Paths

### "I just want to understand the flow"

Read these in order:
1. [What Is SSI Verification?](./what-is-ssi-verification.md)
2. [Presentation Request — Conceptual Overview](../02-presentation-request/conceptual-overview.md)
3. [Presentation Response — Conceptual Overview](../08-presentation-response/conceptual-overview.md)

### "I need to implement a Verifier"

Read these in order:
1. [Authorization Request — Protocol](../04-authorization-request/protocol-and-standards.md)
2. [Presentation Definition — Protocol](../05-presentation-definition/protocol-and-standards.md)
3. [Verifier Validation — Protocol](../09-verifier-validation/protocol-and-standards.md)

### "I need to implement Holder presentation"

Read these in order:
1. [Credential Selection — Implementation](../06-credential-selection/conceptual-overview.md)
2. [Holder Binding — Implementation](../07-holder-binding/conceptual-overview.md)
3. [Presentation Response — Implementation](../08-presentation-response/implementation-details.md)

---

## Cross-References with Issuance Documentation

Many verification concepts have direct parallels in the [Issuance Documentation](../../docs/):

| Verification Concept | Issuance Parallel |
|----------------------|-------------------|
| Presentation Request | Credential Offer |
| Verifier Metadata | Issuer Metadata |
| Authorization Request (VP) | Authorization Request (VC) |
| VP Token | Credential Response |
| Holder Binding Proof | Proof of Possession |
| Nonce handling | c_nonce handling |

If you are implementing both issuance and verification, read the issuance documentation first as it establishes foundational concepts that verification builds upon.
