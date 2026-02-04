# SSI Documentation Hub

> Production-grade technical documentation for designing and implementing Self-Sovereign Identity (SSI) wallet flows. Covers credential issuance and verification/presentation across EUDI (Android & iOS), Procivis ONE, and Affinidi implementations.

---

## Documentation Structure

This repository contains two comprehensive documentation sets covering the complete SSI credential lifecycle:

| Documentation | Description | Entry Point |
|---------------|-------------|-------------|
| **[Issuance](./issuance/)** | Credential issuance flow — from offer to secure storage | [Issuance README](./issuance/README.md) |
| **[Verification](./verification/)** | Presentation flow — from request to validation | [Verification README](./verification/README.md) |

```
                    Issues credential (OID4VCI)
    Issuer  ───────────────────────────────────────►  Holder
                                                        │
                                                        │ Presents credential (OID4VP)
                                                        ▼
                                                     Verifier
```

---

## Audience

This documentation is intended for experienced engineering teams (backend, mobile, platform) that are comfortable with cryptography and distributed systems but new to Self-Sovereign Identity.

After reading this documentation, your team will be able to:

- Understand SSI credential issuance and verification end-to-end
- Choose between architectural approaches
- Rebuild issuance and presentation flows independently
- Make informed trade-offs between EUDI, Procivis, and Affinidi design patterns

---

## Implementations Analyzed

| Implementation | Platform | Language | Core SDK | Protocols |
|---|---|---|---|---|
| **EUDI Wallet** (Android) | Android | Kotlin | `eudi-lib-android-wallet-core` | OID4VCI v1.0, OID4VP v1.0, ISO 18013-5 |
| **EUDI Wallet** (iOS) | iOS | Swift | EudiWalletKit | OID4VCI v1.0, OID4VP v1.0, ISO 18013-5 |
| **Procivis ONE** | Cross-platform | TypeScript (React Native) | `@procivis/react-native-one-core` | OID4VCI (Draft 13, Final), OID4VP, PEX v1/v2 |
| **Affinidi** | Cloud + Vault | Multi-language TDK | Affinidi TDK | OID4VCI (Pre-Auth), OID4VP with PEX |

---

## Issuance Documentation

The complete credential issuance flow — from credential offer to secure storage.

### Sections

| # | Section | Description |
|---|---------|-------------|
| 00 | [Introduction](./issuance/00-introduction/README.md) | What SSI issuance is, who this is for, how to navigate these docs |
| 01 | [Architecture Overview](./issuance/01-architecture/README.md) | High-level architecture of each wallet's issuance pipeline |
| 02 | [Credential Offer](./issuance/02-credential-offer/README.md) | QR codes, deep links, offer resolution, transport mechanisms |
| 03 | [Issuer Metadata Discovery](./issuance/03-issuer-metadata/README.md) | `/.well-known` endpoints, credential configurations, display metadata |
| 04 | [Authorization & Token](./issuance/04-authorization-and-token/README.md) | OAuth 2.0, PAR, DPoP, pre-authorized code, PKCE |
| 05 | [Nonce & Replay Protection](./issuance/05-nonce-and-replay-protection/README.md) | c_nonce lifecycle, transaction codes, replay prevention |
| 06 | [Key Attestation](./issuance/06-key-attestation/README.md) | Hardware-backed keys, wallet attestation, key provenance |
| 07 | [Proof Construction](./issuance/07-proof-construction/README.md) | JWT proofs, key binding, proof-of-possession, signing |
| 08 | [Credential Request](./issuance/08-credential-request/README.md) | POST /credential, Draft 13 vs Final, payload structure |
| 09 | [Issuer Response](./issuance/09-issuer-response/README.md) | Credential delivery, deferred issuance, notifications |
| 10 | [Secure Storage](./issuance/10-secure-storage/README.md) | Android Keystore, iOS Secure Enclave, encrypted databases |
| 11 | [Cryptography Deep Dive](./issuance/11-cryptography/README.md) | Every algorithm, protocol, and mechanism explained in depth |
| 12 | [Comparative Summary](./issuance/12-comparative-summary/README.md) | Executive comparison, decision framework, feature matrices |

---

## Verification Documentation

The complete credential verification and presentation flow — from presentation request to response validation.

### Sections

| # | Section | Description |
|---|---------|-------------|
| 00 | [Introduction](./verification/00-introduction/README.md) | What SSI verification is, trust models, how to navigate these docs |
| 01 | [Architecture Overview](./verification/01-architecture/README.md) | High-level architecture of each wallet's verification pipeline |
| 02 | [Presentation Request](./verification/02-presentation-request/README.md) | QR codes, deep links, request resolution, transport mechanisms |
| 03 | [Verifier Metadata](./verification/03-verifier-metadata/README.md) | Client metadata, trust establishment, verifier identification |
| 04 | [Authorization Request](./verification/04-authorization-request/README.md) | OID4VP request structure, client_id schemes, response modes |
| 05 | [Presentation Definition](./verification/05-presentation-definition/README.md) | PEX queries, DCQL, input descriptors, credential constraints |
| 06 | [Credential Selection](./verification/06-credential-selection/README.md) | Holder-side matching, user consent, selective disclosure UI |
| 07 | [Holder Binding](./verification/07-holder-binding/README.md) | Proof of possession, key binding, cryptographic proofs |
| 08 | [Presentation Response](./verification/08-presentation-response/README.md) | VP Token construction, response transmission, redirect handling |
| 09 | [Verifier Validation](./verification/09-verifier-validation/README.md) | Response verification, signature checks, revocation status |
| 10 | [Transport Modes](./verification/10-transport-modes/README.md) | Remote (HTTPS), proximity (BLE/NFC), cross-device vs same-device |
| 11 | [Cryptography Deep Dive](./verification/11-cryptography/README.md) | Verification-specific cryptographic operations |
| 12 | [Comparative Summary](./verification/12-comparative-summary/README.md) | Executive comparison, decision framework, feature matrices |

---

## Reading Paths

### Complete Understanding
Read issuance (00-12) then verification (00-12) in order. Each section builds on the previous.

### Quick Architectural Decisions
- Issuance: [Architecture](./issuance/01-architecture/README.md) and [Comparative Summary](./issuance/12-comparative-summary/README.md)
- Verification: [Architecture](./verification/01-architecture/README.md) and [Comparative Summary](./verification/12-comparative-summary/README.md)

### Specific Flow Step
Jump directly to the relevant section. Each is self-contained with its own conceptual overview.

### Cryptographic Details
- Issuance: [Cryptography](./issuance/11-cryptography/README.md)
- Verification: [Cryptography](./verification/11-cryptography/README.md)

---

## Appendices

| Resource | Location | Description |
|----------|----------|-------------|
| Issuance Appendix | [appendix/README.md](./appendix/README.md) | Glossary and standards index for issuance |
| Verification Appendix | [verification/appendix/README.md](./verification/appendix/README.md) | Glossary and standards index for verification |

---

## Conventions

- **Code references** cite actual source files from the analyzed repositories. No APIs are invented.
- **Mermaid diagrams** are used for sequence and flow diagrams. Render with any Mermaid-compatible viewer.
- **Comparison tables** appear in every section where implementations diverge.
- When a project does not implement a feature or abstracts it away, this is explicitly stated with implications discussed.
- Where information is unavailable from source code (especially for Affinidi, which is analyzed from public documentation rather than source), this is clearly noted.

---

## Standards Referenced

### Issuance Standards

| Standard | Description |
|---|---|
| OID4VCI | OpenID for Verifiable Credential Issuance (Draft 13, v1.0 Final) |
| RFC 9126 | Pushed Authorization Requests (PAR) |
| RFC 9449 | DPoP (Demonstration of Proof-of-Possession) |

### Verification Standards

| Standard | Description |
|---|---|
| OID4VP | OpenID for Verifiable Presentations (Draft 20+, v1.0) |
| DIF PEX | Presentation Exchange v1.0 / v2.0 |
| DCQL | Digital Credentials Query Language |
| SIOPv2 | Self-Issued OpenID Provider v2 |

### Shared Standards

| Standard | Description |
|---|---|
| ISO 18013-5 | Mobile Driving License (mDL / mDoc) |
| W3C VC Data Model | Verifiable Credentials Data Model v1.1 / v2.0 |
| DID Core | Decentralized Identifiers v1.0 |
| SD-JWT | Selective Disclosure for JWTs |
| RFC 7519 | JSON Web Token (JWT) |
| RFC 7517 | JSON Web Key (JWK) |
