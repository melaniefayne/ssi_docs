# SSI Credential Issuance — Documentation Hub

> Production-grade technical documentation for designing and implementing SSI wallet credential issuance flows. Covers EUDI (Android & iOS), Procivis ONE, and Affinidi implementations.

---

## Audience

This documentation is intended for experienced engineering teams (backend, mobile, platform) that are comfortable with cryptography and distributed systems but new to Self-Sovereign Identity.

After reading this documentation, your team will be able to:

- Understand SSI credential issuance end-to-end
- Choose between architectural approaches
- Rebuild an issuance flow independently
- Make informed trade-offs between EUDI, Procivis, and Affinidi design patterns

## Scope

**In scope:** The complete credential issuance flow — from credential offer to secure storage.

**Out of scope:** Verification and presentation flows are referenced only where they intersect with issuance (e.g., dynamic issuance requiring a presentation step).

---

## Implementations Analyzed

| Implementation | Platform | Language | Core SDK | OID4VCI Version |
|---|---|---|---|---|
| **EUDI Wallet** (Android) | Android | Kotlin | `eudi-lib-android-wallet-core` v0.23.0 | v1.0 (Final) |
| **EUDI Wallet** (iOS) | iOS | Swift | EudiWalletKit v0.19.4 | v1.0 (Final) |
| **Procivis ONE** | Cross-platform | TypeScript (React Native) | `@procivis/react-native-one-core` v1.81885.0 | Draft 13, Final 1, HAIP |
| **Affinidi** | Cloud + Vault | Multi-language TDK | Affinidi Credential Issuance Service | OID4VCI (Pre-Auth Code) |

---

## Documentation Map

### Foundations

| # | Section | Description |
|---|---------|-------------|
| 00 | [Introduction](./00-introduction/) | What SSI issuance is, who this is for, how to navigate these docs |
| 01 | [Architecture Overview](./01-architecture/) | High-level architecture of each wallet's issuance pipeline |

### Issuance Flow (Step by Step)

| # | Section | Description |
|---|---------|-------------|
| 02 | [Credential Offer](./02-credential-offer/) | QR codes, deep links, offer resolution, transport mechanisms |
| 03 | [Issuer Metadata Discovery](./03-issuer-metadata/) | `/.well-known` endpoints, credential configurations, display metadata |
| 04 | [Authorization & Token](./04-authorization-and-token/) | OAuth 2.0, PAR, DPoP, pre-authorized code, PKCE |
| 05 | [Nonce & Replay Protection](./05-nonce-and-replay-protection/) | c_nonce lifecycle, transaction codes, replay prevention |
| 06 | [Key Attestation](./06-key-attestation/) | Hardware-backed keys, wallet attestation, key provenance |
| 07 | [Proof Construction](./07-proof-construction/) | JWT proofs, key binding, proof-of-possession, signing |
| 08 | [Credential Request](./08-credential-request/) | POST /credential, Draft 13 vs Final, payload structure |
| 09 | [Issuer Response](./09-issuer-response/) | Credential delivery, deferred issuance, notifications |
| 10 | [Secure Storage](./10-secure-storage/) | Android Keystore, iOS Secure Enclave, encrypted databases |

### Deep Dives & Reference

| # | Section | Description |
|---|---------|-------------|
| 11 | [Cryptography Deep Dive](./11-cryptography/) | Every algorithm, protocol, and mechanism explained in depth |
| 12 | [Comparative Summary](./12-comparative-summary/) | Executive comparison, decision framework, feature matrices |
| — | [Appendix](./appendix/) | Glossary, references, standards index |

---

## Reading Order

**For a complete understanding:** Read sections 00 through 12 in order. Each section builds on the previous.

**For quick architectural decisions:** Start with [01 Architecture](./01-architecture/) and [12 Comparative Summary](./12-comparative-summary/).

**For a specific issuance step:** Jump directly to the relevant section (02–10). Each is self-contained with its own conceptual overview.

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
| OID4VCI | OpenID for Verifiable Credential Issuance (Draft 13, v1.0 Final) |
| OID4VP | OpenID for Verifiable Presentations |
| ISO 18013-5 | Mobile Driving License (mDL / mDoc) |
| W3C VC Data Model | Verifiable Credentials Data Model v1.1 / v2.0 |
| DID Core | Decentralized Identifiers v1.0 |
| SD-JWT | Selective Disclosure for JWTs |
| RFC 7519 | JSON Web Token (JWT) |
| RFC 7517 | JSON Web Key (JWK) |
| RFC 9126 | Pushed Authorization Requests (PAR) |
| RFC 9449 | DPoP (Demonstration of Proof-of-Possession) |
