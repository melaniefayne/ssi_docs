# SSI Credential Verification — Documentation Hub

> Production-grade technical documentation for designing and implementing SSI wallet credential verification and presentation flows. Covers EUDI (Android & iOS), Procivis ONE, and Affinidi implementations.

---

## Audience

This documentation is intended for experienced engineering teams (backend, mobile, platform) that are comfortable with cryptography and distributed systems but new to Self-Sovereign Identity verification flows.

After reading this documentation, your team will be able to:

- Understand SSI credential verification and presentation end-to-end
- Choose between architectural approaches for verifier and holder implementations
- Rebuild a verification/presentation flow independently
- Make informed trade-offs between EUDI, Procivis, and Affinidi design patterns

## Scope

**In scope:** The complete credential verification and presentation flow — from presentation request to response validation.

**Out of scope:** Credential issuance flows are documented separately in the [Issuance Documentation](../docs/).

---

## Relationship to Issuance

Verification is the counterpart to issuance in the SSI trust triangle. While issuance creates and delivers credentials from an Issuer to a Holder, verification enables a Holder to present those credentials to a Verifier.

```
                  Issues credential
Issuer  ─────────────────────────────────►  Holder
                                              │
                                              │ Presents credential
                                              ▼
                                           Verifier
```

The protocols mirror each other:
- **OID4VCI** (OpenID for Verifiable Credential Issuance) — Issuer → Holder
- **OID4VP** (OpenID for Verifiable Presentations) — Holder → Verifier

---

## Implementations Analyzed

| Implementation | Platform | Language | Core SDK | OID4VP Support |
|---|---|---|---|---|
| **EUDI Wallet** (Android) | Android | Kotlin | `eudi-lib-android-wallet-core` | OID4VP v1.0, ISO 18013-5 (BLE) |
| **EUDI Wallet** (iOS) | iOS | Swift | EudiWalletKit | OID4VP v1.0, ISO 18013-5 (BLE) |
| **Procivis ONE** | Cross-platform | TypeScript (React Native) | `@procivis/react-native-one-core` | OID4VP, ISO mDL, PEX v1/v2 |
| **Affinidi** | Cloud + Vault | Multi-language TDK | Affinidi Iota Framework | OID4VP with PEX |

---

## Documentation Map

### Foundations

| # | Section | Description |
|---|---------|-------------|
| 00 | [Introduction](./00-introduction/) | What SSI verification is, trust models, how to navigate these docs |
| 01 | [Architecture Overview](./01-architecture/) | High-level architecture of each wallet's verification pipeline |

### Verification Flow (Step by Step)

| # | Section | Description |
|---|---------|-------------|
| 02 | [Presentation Request](./02-presentation-request/) | QR codes, deep links, request resolution, transport mechanisms |
| 03 | [Verifier Metadata](./03-verifier-metadata/) | Client metadata, trust establishment, verifier identification |
| 04 | [Authorization Request](./04-authorization-request/) | OID4VP request structure, client_id schemes, response modes |
| 05 | [Presentation Definition](./05-presentation-definition/) | PEX queries, DCQL, input descriptors, credential constraints |
| 06 | [Credential Selection](./06-credential-selection/) | Holder-side matching, user consent, selective disclosure UI |
| 07 | [Holder Binding](./07-holder-binding/) | Proof of possession, key binding, cryptographic proofs |
| 08 | [Presentation Response](./08-presentation-response/) | VP Token construction, response transmission, redirect handling |
| 09 | [Verifier Validation](./09-verifier-validation/) | Response verification, signature checks, revocation status |
| 10 | [Transport Modes](./10-transport-modes/) | Remote (HTTPS), proximity (BLE/NFC), cross-device vs same-device |

### Deep Dives & Reference

| # | Section | Description |
|---|---------|-------------|
| 11 | [Cryptography Deep Dive](./11-cryptography/) | Verification-specific cryptographic operations |
| 12 | [Comparative Summary](./12-comparative-summary/) | Executive comparison, decision framework, feature matrices |
| — | [Appendix](./appendix/) | Glossary, references, standards index |

---

## Reading Order

**For a complete understanding:** Read sections 00 through 12 in order. Each section builds on the previous.

**For quick architectural decisions:** Start with [01 Architecture](./01-architecture/) and [12 Comparative Summary](./12-comparative-summary/).

**For a specific verification step:** Jump directly to the relevant section (02–10). Each is self-contained with its own conceptual overview.

**For cryptographic details:** Section [11 Cryptography](./11-cryptography/) covers every mechanism referenced throughout the docs.

---

## Conventions

- **Code references** cite actual source files from the analyzed repositories. No APIs are invented.
- **Mermaid diagrams** are used for sequence and flow diagrams. Render with any Mermaid-compatible viewer.
- **Comparison tables** appear in every section where implementations diverge.
- When a project does not implement a feature or abstracts it away, this is explicitly stated with implications discussed.
- Where information is unavailable from source code (especially for Affinidi, which is analyzed from public documentation rather than source), this is clearly noted.

---

## Standards Referenced

| Standard | Description |
|---|---|
| OID4VP | OpenID for Verifiable Presentations (Draft 20+, v1.0) |
| OID4VCI | OpenID for Verifiable Credential Issuance (for issuance context) |
| ISO 18013-5 | Mobile Driving License (mDL / mDoc) — proximity presentation |
| W3C VC Data Model | Verifiable Credentials Data Model v1.1 / v2.0 |
| DID Core | Decentralized Identifiers v1.0 |
| DIF PEX | Presentation Exchange v1.0 / v2.0 |
| DCQL | Digital Credentials Query Language |
| SD-JWT | Selective Disclosure for JWTs |
| RFC 7519 | JSON Web Token (JWT) |
| RFC 7517 | JSON Web Key (JWK) |
| SIOPv2 | Self-Issued OpenID Provider v2 |
