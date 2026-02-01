# Standards Index

All standards, specifications, and RFCs referenced in this documentation, with links and descriptions of their relevance to SSI.

---

## OpenID Specifications

### OID4VCI -- OpenID for Verifiable Credential Issuance

| Property | Detail |
|----------|--------|
| Versions referenced | Draft 13, Draft 14, v1.0 |
| Specification | https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html |
| Working Group | OpenID Foundation |
| Relevance | The core protocol for credential issuance in all three implementations. Defines credential offers, token exchange, credential request/response, batch issuance, deferred issuance, and notification endpoints. |

**Key sections referenced in this documentation:**
- Section 3: Credential Offer (pre-authorized and authorization code flows)
- Section 5: Token Endpoint (DPoP binding, c_nonce)
- Section 7: Credential Endpoint (proof types, credential formats)
- Section 8: Batch Credential Endpoint
- Section 9: Deferred Credential Endpoint
- Section 10: Notification Endpoint
- Appendix A: Credential Format Profiles (jwt_vc_json, vc+sd-jwt, mso_mdoc)

### OID4VP -- OpenID for Verifiable Presentations

| Property | Detail |
|----------|--------|
| Specification | https://openid.net/specs/openid-4-verifiable-presentations-1_0.html |
| Working Group | OpenID Foundation |
| Relevance | The protocol for credential presentation from wallet to verifier. Defines presentation requests, response formats, and VP token structure. |

---

## ISO Standards

### ISO 18013-5 -- Mobile Driving Licence (mDL) Application

| Property | Detail |
|----------|--------|
| Title | Personal identification -- ISO-compliant driving licence -- Part 5: Mobile driving licence (mDL) application |
| Published | 2021 |
| Publisher | International Organization for Standardization |
| Relevance | Defines the mDoc credential format, Mobile Security Object (MSO), CBOR encoding, COSE signing, namespace-based claims, device authentication, and offline presentation protocols (BLE, NFC, Wi-Fi Aware). |

**Key concepts from ISO 18013-5 referenced in this documentation:**
- mDoc document structure (IssuerSigned, DeviceSigned)
- MSO (Mobile Security Object) with per-claim digest hashes
- DeviceAuth for holder proof of possession
- BLE and NFC transport for proximity presentation
- Selective disclosure via individual claim hashing

### ISO 23220 -- Building Blocks for Identity Management via Mobile Devices

| Property | Detail |
|----------|--------|
| Publisher | International Organization for Standardization |
| Relevance | Extends the mDoc format from ISO 18013-5 to general-purpose identity documents beyond driving licenses. Provides the framework for PID (Person Identification Data) and other government-issued credentials in mDoc format. |

---

## W3C Specifications

### W3C Verifiable Credentials Data Model

| Property | Detail |
|----------|--------|
| Version 1.1 | https://www.w3.org/TR/vc-data-model/ |
| Version 2.0 | https://www.w3.org/TR/vc-data-model-2.0/ |
| Working Group | W3C Verifiable Credentials Working Group |
| Relevance | Defines the standard data model for verifiable credentials and verifiable presentations using JSON-LD. Specifies @context, type, issuer, credentialSubject, and proof structures. Used by Affinidi (JSON-LD VC) and Procivis (JSON-LD VC and JWT VC). |

### W3C DID Core

| Property | Detail |
|----------|--------|
| Specification | https://www.w3.org/TR/did-core/ |
| Working Group | W3C Decentralized Identifier Working Group |
| Relevance | Defines the Decentralized Identifier (DID) format and resolution process. DIDs identify issuers, holders, and verifiers. DID documents contain public keys and service endpoints. Referenced throughout the documentation for issuer identification and key resolution. |

### W3C Data Integrity

| Property | Detail |
|----------|--------|
| Specification | https://www.w3.org/TR/vc-data-integrity/ |
| Working Group | W3C Verifiable Credentials Working Group |
| Relevance | Defines the mechanism for adding cryptographic proofs to JSON-LD documents, including canonicalization (URDNA2015), hashing, and proof embedding. Used by Affinidi's EcdsaSecp256k1Signature2019 proof suite and Procivis's JSON-LD VC support. |

---

## SD-JWT

### SD-JWT -- Selective Disclosure for JWTs

| Property | Detail |
|----------|--------|
| Specification | https://www.ietf.org/archive/id/draft-ietf-oauth-selective-disclosure-jwt-13.html |
| Working Group | IETF OAuth Working Group |
| Relevance | Defines the selective disclosure mechanism for JWTs: _sd arrays with hashed disclosures, disclosure format ([salt, claim_name, claim_value]), key binding JWT (KB-JWT), and the wire format (issuer-jwt~disclosure1~disclosure2~kb-jwt). Used by EUDI and Procivis. |

### SD-JWT VC -- SD-JWT-based Verifiable Credentials

| Property | Detail |
|----------|--------|
| Specification | https://www.ietf.org/archive/id/draft-ietf-oauth-sd-jwt-vc-05.html |
| Working Group | IETF OAuth Working Group |
| Relevance | Profiles SD-JWT for use as a verifiable credential format. Defines the vct (verifiable credential type) claim, metadata discovery, and status mechanisms. Used as the vc+sd-jwt format in OID4VCI. |

---

## IETF RFCs

### RFC 7515 -- JSON Web Signature (JWS)

| Property | Detail |
|----------|--------|
| Link | https://www.rfc-editor.org/rfc/rfc7515 |
| Published | May 2015 |
| Relevance | Defines the JWS structure (header.payload.signature) used throughout SSI for signing JWTs, DPoP proofs, credential proofs, and key binding tokens. JWS Compact Serialization is the primary serialization format used. |

### RFC 7517 -- JSON Web Key (JWK)

| Property | Detail |
|----------|--------|
| Link | https://www.rfc-editor.org/rfc/rfc7517 |
| Published | May 2015 |
| Relevance | Defines the JSON format for representing cryptographic keys. JWKs are used in DPoP proof headers, issuer metadata (JWKS), SD-JWT cnf claims, DID documents, and credential proofs. JWK Sets (JWKS) are used by issuers to publish their signing keys. |

### RFC 7518 -- JSON Web Algorithms (JWA)

| Property | Detail |
|----------|--------|
| Link | https://www.rfc-editor.org/rfc/rfc7518 |
| Published | May 2015 |
| Relevance | Registers cryptographic algorithms for use with JWS, JWE, and JWK. Defines ES256 (ECDSA with P-256 and SHA-256), which is the primary signing algorithm used in EUDI and Procivis OID4VCI implementations. |

### RFC 7519 -- JSON Web Token (JWT)

| Property | Detail |
|----------|--------|
| Link | https://www.rfc-editor.org/rfc/rfc7519 |
| Published | May 2015 |
| Relevance | Defines the JWT format: a compact, URL-safe token with JSON claims. JWTs are used for access tokens, DPoP proofs, credential proofs, wallet attestation, key binding proofs, and as a credential format (JWT VC). Standard claims (iss, sub, aud, exp, iat, jti) are referenced throughout the documentation. |

### RFC 7636 -- Proof Key for Code Exchange (PKCE)

| Property | Detail |
|----------|--------|
| Link | https://www.rfc-editor.org/rfc/rfc7636 |
| Published | September 2015 |
| Relevance | Defines the code_verifier/code_challenge mechanism for preventing authorization code interception in OAuth 2.0. Used in the OID4VCI authorization code flow. The S256 challenge method (SHA-256 hash of the verifier) is required by HAIP. |

### RFC 7638 -- JSON Web Key (JWK) Thumbprint

| Property | Detail |
|----------|--------|
| Link | https://www.rfc-editor.org/rfc/rfc7638 |
| Published | September 2015 |
| Relevance | Defines a method for computing a canonical thumbprint (SHA-256 hash) of a JWK. JWK thumbprints are used in DPoP to bind access tokens to keys (the jkt confirmation method) without embedding the full key in the token. |

### RFC 8949 -- Concise Binary Object Representation (CBOR)

| Property | Detail |
|----------|--------|
| Link | https://www.rfc-editor.org/rfc/rfc8949 |
| Published | December 2020 |
| Relevance | Defines the CBOR binary serialization format used as the encoding layer for mDoc credentials (ISO 18013-5). CBOR provides compact, efficient encoding for constrained environments (BLE, NFC transport). |

### RFC 9052 -- CBOR Object Signing and Encryption (COSE)

| Property | Detail |
|----------|--------|
| Link | https://www.rfc-editor.org/rfc/rfc9052 |
| Published | August 2022 |
| Relevance | Defines the COSE signing and encryption structures used in mDoc credentials. COSE_Sign1 is used for issuer authentication (signing the MSO) and device authentication (holder proof of possession). COSE uses integer algorithm identifiers (-7 for ES256). |

### RFC 9126 -- OAuth 2.0 Pushed Authorization Requests (PAR)

| Property | Detail |
|----------|--------|
| Link | https://www.rfc-editor.org/rfc/rfc9126 |
| Published | September 2022 |
| Relevance | Defines the PAR mechanism for pre-registering authorization requests with the authorization server via a back-channel POST. Used in OID4VCI to prevent request tampering, avoid URL length limits, and keep authorization parameters confidential. Returns a request_uri that replaces the full parameter set in the browser redirect. |

### RFC 9449 -- OAuth 2.0 Demonstrating Proof of Possession (DPoP)

| Property | Detail |
|----------|--------|
| Link | https://www.rfc-editor.org/rfc/rfc9449 |
| Published | September 2023 |
| Relevance | Defines the DPoP mechanism for binding OAuth 2.0 access tokens to specific key pairs, preventing token theft. The DPoP proof JWT contains jti, htm, htu, iat, and ath fields. The access token's cnf.jkt claim references the DPoP key. Used in both the token endpoint and credential endpoint of OID4VCI. |

---

## NIST Standards

### NIST FIPS 186-4 -- Digital Signature Standard (DSS)

| Property | Detail |
|----------|--------|
| Link | https://csrc.nist.gov/publications/detail/fips/186/4/final |
| Relevance | Specifies ECDSA and defines the P-256 (secp256r1) curve used by ES256. FIPS 186-4 certification is required by many government security standards, which is one reason P-256 is mandated by HAIP and used by EUDI. |

### NIST FIPS 203 -- Module-Lattice-Based Key-Encapsulation Mechanism Standard

| Property | Detail |
|----------|--------|
| Link | https://csrc.nist.gov/publications/detail/fips/203/final |
| Relevance | Part of the NIST Post-Quantum Cryptography standardization. Related to the CRYSTALS family. FIPS 204 (ML-DSA, based on CRYSTALS-DILITHIUM) defines the post-quantum signature scheme used by Procivis. |

### NIST FIPS 204 -- Module-Lattice-Based Digital Signature Standard (ML-DSA)

| Property | Detail |
|----------|--------|
| Link | https://csrc.nist.gov/publications/detail/fips/204/final |
| Relevance | Standardizes the CRYSTALS-DILITHIUM signature scheme as ML-DSA. DILITHIUM Level 3 (ML-DSA-65) is the variant used by Procivis for post-quantum credential signing. |

---

## EU Regulations

### eIDAS 2.0 -- European Digital Identity Framework

| Property | Detail |
|----------|--------|
| Title | Regulation (EU) 2024/1183 amending Regulation (EU) No 910/2014 |
| Relevance | The legal framework driving the EUDI Reference Wallet. Mandates that EU member states offer digital identity wallets to citizens. Specifies requirements for Wallet Secure Cryptographic Devices (WSCD), qualified electronic signatures, cross-border interoperability, and trust services. |

### ARF -- Architecture and Reference Framework

| Property | Detail |
|----------|--------|
| Title | European Digital Identity Wallet Architecture and Reference Framework |
| Relevance | The technical architecture document for the EUDI Wallet ecosystem. Specifies the roles (issuer, wallet, verifier), protocols (OID4VCI, OID4VP), credential formats (mDoc, SD-JWT), and trust infrastructure (trust registries, wallet attestation) for the European Digital Identity framework. |

---

## Quick Reference Table

| Standard | Short Name | Primary Use in SSI |
|----------|-----------|-------------------|
| OID4VCI | Credential Issuance | Protocol for issuing credentials to wallets |
| OID4VP | Credential Presentation | Protocol for presenting credentials to verifiers |
| ISO 18013-5 | mDL / mDoc | mDoc credential format and offline presentation |
| W3C VC Data Model | VC | JSON-LD credential data model |
| W3C DID Core | DID | Decentralized identifiers |
| SD-JWT | Selective Disclosure JWT | Selective disclosure credential format |
| RFC 7515 | JWS | JSON signature structure |
| RFC 7517 | JWK | JSON key representation |
| RFC 7519 | JWT | JSON token format |
| RFC 7636 | PKCE | Authorization code interception prevention |
| RFC 8949 | CBOR | Binary data encoding for mDoc |
| RFC 9052 | COSE | Binary signing/encryption for mDoc |
| RFC 9126 | PAR | Authorization request pre-registration |
| RFC 9449 | DPoP | Proof-of-possession for access tokens |
| FIPS 186-4 | DSS | ECDSA and P-256 curve specification |
| FIPS 204 | ML-DSA | Post-quantum signature standard (DILITHIUM) |
| eIDAS 2.0 | EU Digital Identity | Regulatory framework for EU digital wallets |
