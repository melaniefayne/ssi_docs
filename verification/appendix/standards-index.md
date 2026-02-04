# Standards Index

References to all standards, specifications, and RFCs cited in this documentation.

---

## Core Verification Standards

### OpenID for Verifiable Presentations (OID4VP)

**Specification:** https://openid.net/specs/openid-4-verifiable-presentations-1_0.html

| Version | Status | Notes |
|---------|--------|-------|
| v1.0 | Published | Current production version |
| Draft 20+ | Draft | Latest working draft |

**Key sections referenced:**
- Section 5: Authorization Request
- Section 5.1: Client Identifier
- Section 5.4: Presentation Definition
- Section 5.5: DCQL
- Section 6: Response Modes

### DIF Presentation Exchange (PEX)

**Specification:** https://identity.foundation/presentation-exchange/

| Version | Status | Notes |
|---------|--------|-------|
| v1.0.0 | Final | Widely implemented |
| v2.0.0 | Final | Adds credential sets |

**Key sections referenced:**
- Section 4: Presentation Definition
- Section 5: Input Descriptor
- Section 6: Submission Requirements

---

## Credential Format Standards

### W3C Verifiable Credentials Data Model

**Specification:** https://www.w3.org/TR/vc-data-model/

| Version | Status | Notes |
|---------|--------|-------|
| v1.1 | Recommendation | Current production |
| v2.0 | Candidate Rec | Emerging |

### SD-JWT (Selective Disclosure for JWTs)

**Specification:** https://datatracker.ietf.org/doc/draft-ietf-oauth-selective-disclosure-jwt/

| Version | Status | Notes |
|---------|--------|-------|
| Draft 07+ | IETF Draft | Nearing completion |

### ISO 18013-5 (Mobile Driving License)

**Specification:** ISO/IEC 18013-5:2021

| Part | Title |
|------|-------|
| Part 5 | Mobile Driving Licence (mDL) application |
| Part 7 | mDL add-on functions |

---

## Identity Standards

### DID Core

**Specification:** https://www.w3.org/TR/did-core/

W3C Recommendation for Decentralized Identifiers.

### SIOPv2 (Self-Issued OpenID Provider)

**Specification:** https://openid.net/specs/openid-connect-self-issued-v2-1_0.html

Self-issued identity provider for OID4VP integration.

---

## OAuth 2.0 Foundation

### RFC 6749 - OAuth 2.0 Authorization Framework

**URL:** https://datatracker.ietf.org/doc/html/rfc6749

Foundation for OID4VP authorization.

### RFC 7636 - PKCE

**URL:** https://datatracker.ietf.org/doc/html/rfc7636

Proof Key for Code Exchange.

### RFC 9101 - JAR (JWT-Secured Authorization Request)

**URL:** https://datatracker.ietf.org/doc/html/rfc9101

Signed authorization requests.

### RFC 9126 - PAR (Pushed Authorization Requests)

**URL:** https://datatracker.ietf.org/doc/html/rfc9126

Server-side authorization request storage.

### RFC 9449 - DPoP (Demonstration of Proof-of-Possession)

**URL:** https://datatracker.ietf.org/doc/html/rfc9449

Sender-constrained access tokens.

---

## Cryptographic Standards

### RFC 7519 - JSON Web Token (JWT)

**URL:** https://datatracker.ietf.org/doc/html/rfc7519

JWT format specification.

### RFC 7515 - JSON Web Signature (JWS)

**URL:** https://datatracker.ietf.org/doc/html/rfc7515

Digital signature format.

### RFC 7516 - JSON Web Encryption (JWE)

**URL:** https://datatracker.ietf.org/doc/html/rfc7516

Content encryption format.

### RFC 7517 - JSON Web Key (JWK)

**URL:** https://datatracker.ietf.org/doc/html/rfc7517

Key representation format.

### RFC 8152 - CBOR Object Signing and Encryption (COSE)

**URL:** https://datatracker.ietf.org/doc/html/rfc8152

CBOR-based signatures (used in mDoc).

---

## Query Languages

### RFC 9535 - JSONPath

**URL:** https://datatracker.ietf.org/doc/html/rfc9535

JSON query language (used in PEX).

### JSON Schema

**Specification:** https://json-schema.org/

Filter vocabulary for PEX field constraints.

---

## High Assurance Profiles

### HAIP (High Assurance Interoperability Profile)

**Specification:** https://openid.net/high-assurance-interoperability-profile/

Stricter requirements for high-security deployments.

### ARF (Architecture Reference Framework)

**Specification:** EUDI Wallet Architecture Reference Framework

European Digital Identity Wallet architecture.

---

## Implementation References

### EUDI Wallet

| Repository | URL |
|------------|-----|
| Android UI | https://github.com/eu-digital-identity-wallet/eudi-app-android-wallet-ui |
| iOS UI | https://github.com/eu-digital-identity-wallet/eudi-app-ios-wallet-ui |
| Android Core | https://github.com/eu-digital-identity-wallet/eudi-lib-android-wallet-core |
| iOS Core | https://github.com/eu-digital-identity-wallet/eudi-lib-ios-wallet-core |

### Procivis ONE

| Resource | URL |
|----------|-----|
| Documentation | https://docs.procivis.ch/ |
| React Native SDK | npm: @procivis/react-native-one-core |

### Affinidi

| Resource | URL |
|----------|-----|
| Documentation | https://docs.affinidi.com/ |
| Iota Framework | https://docs.affinidi.com/frameworks/iota-framework/ |
| TDK | https://docs.affinidi.com/dev-tools/affinidi-tdk/ |

---

## Citation Format

When referencing standards in implementations:

```
[OID4VP] OpenID for Verifiable Presentations 1.0
[PEX] DIF Presentation Exchange v2.0.0
[SD-JWT] Selective Disclosure for JWTs (draft-ietf-oauth-selective-disclosure-jwt)
[ISO18013-5] ISO/IEC 18013-5:2021 Mobile Driving Licence
[VC-DM] W3C Verifiable Credentials Data Model v1.1
```
