# SSI Wallets — Overview

## 1. What is an SSI Wallet?
A **Self‑Sovereign Identity (SSI) wallet** is a user‑controlled application that stores and manages digital credentials (Verifiable Credentials) and enables users to prove claims about themselves without relying on a centralized identity provider. The wallet holds credentials, keys, and proofs, and allows selective disclosure of information.

## 2. What SSI Wallets Are Used For
SSI wallets are used to:
- **Store credentials** (IDs, certificates, permits, licenses).
- **Present proofs** of claims (age, membership, education, residency).
- **Control data sharing** by disclosing only what is required.
- **Support offline and online verification** using cryptographic checks.
- **Reduce dependency on centralized identity providers**.

Common use cases:
- Government IDs and licenses
- Education and professional credentials
- Healthcare and vaccination records
- Travel documents and permits
- KYC/AML verification in finance

## 3. Core Concepts
### Verifiable Credentials (VCs)
A **Verifiable Credential** is a cryptographically signed data object issued by a trusted issuer to a subject. It can be verified without contacting the issuer.

### Verifiable Presentation (VP)
A **Verifiable Presentation** is a package of one or more VCs presented by a holder to a verifier. It may include **selective disclosure** or **zero‑knowledge proofs**.

### DIDs (Decentralized Identifiers)
A **DID** is a globally unique identifier controlled by the wallet. It resolves to a DID document containing public keys and service endpoints.

### Issuer, Holder, Verifier
- **Issuer**: entity that creates and signs credentials.
- **Holder**: user who stores and presents credentials.
- **Verifier**: relying party that verifies presented credentials.

## 4. Credential Issuance Flow (High‑Level)
1. **Credential Offer**: Holder receives a credential offer (QR, link, deep link).
2. **Issuer Metadata**: Wallet resolves issuer capabilities (formats, proof requirements).
3. **Authorization / Token**: Holder obtains an access token if required.
4. **Proof Generation**: Holder proves possession of key or satisfies issuer challenge.
5. **Credential Request**: Wallet requests the credential from issuer.
6. **Credential Response**: Issuer returns the signed VC.
7. **Storage**: Wallet stores credential securely.

## 5. Proofs and Binding
SSI wallets rely on cryptographic proofs:
- **Proof of possession (PoP)**: Shows the holder controls a key bound to the credential.
- **Nonce challenges**: Prevent replay attacks in issuance or presentation.
- **Key attestation**: Demonstrates the key is hardware‑backed (optional).
- **Selective disclosure proofs**: Reveal only specific claims.

Common proof types:
- JWT‑based proofs
- Data Integrity proofs (JSON‑LD)
- SD‑JWT selective disclosure
- Zero‑Knowledge proofs (ZK)

## 6. Verification Flow (High‑Level)
1. **Verifier Request**: Verifier sends a presentation request.
2. **Credential Selection**: Holder chooses matching credential(s).
3. **Proof Creation**: Wallet produces a verifiable presentation.
4. **Verification**: Verifier checks signature, issuer trust, schema, revocation, and validity.

Verification checks typically include:
- Signature validation
- Issuer trust (DID resolution, trust registry)
- Schema conformance
- Expiration and issuance date
- Revocation status

## 7. Revocation and Status
SSI systems often support revocation via:
- **Status lists** (e.g., StatusList2021/Bitstring Status List)
- **Issuer‑hosted revocation registries**

Wallets and verifiers should check revocation status when presenting or verifying.

## 8. Standards and Protocols
Common standards used by SSI wallets:
- **W3C Verifiable Credentials**
- **Decentralized Identifiers (DIDs)**
- **OID4VCI** (OpenID for Credential Issuance)
- **OID4VP** (OpenID for Verifiable Presentations)
- **SD‑JWT** (Selective Disclosure JWT)

## 9. Security Considerations
- **Secure key storage** (hardware‑backed when possible)
- **Protection of credentials at rest** (encrypted storage)
- **Replay prevention** (nonce usage)
- **Selective disclosure** to minimize data exposure
- **Issuer trust framework** integration

## 10. Benefits of SSI Wallets
- User control and privacy
- Reduced data sharing
- Tamper‑evident credentials
- Offline verification
- Interoperability across issuers and verifiers

## 11. Limitations and Challenges
- Trust frameworks and issuer discovery
- Revocation scalability
- UX complexity for non‑technical users
- Cross‑platform interoperability still evolving

---

### Summary
SSI wallets are a foundational component of decentralized identity, providing secure storage, controlled sharing, and cryptographic verification of credentials. They enable users to manage identity data independently while supporting trusted verification by relying parties.


### Resources
- [EU Digital Identitiy Wallet](https://github.com/eu-digital-identity-wallet/.github/blob/main/profile/README.md)
- [EUDI Android wallet](https://github.com/eu-digital-identity-wallet/eudi-app-android-wallet-ui/blob/main/README.md)
- [EUDI iOS wallet](https://github.com/eu-digital-identity-wallet/eudi-app-ios-wallet-ui/blob/main/README.md)
- [Affindi](https://docs.affinidi.com/docs/overview/)
- [Procivis](https://github.com/procivis/one-wallet/blob/main/README.md)



### The Issuance Flow
1. Offer - QR Code
2. Issuer URL
3. Authorisation
4. Token -> nonce
5. Attestation hardware
6. Proof -> header, jwt
7. Credential request -> POST (draft 13, or draft 20)
8. Notification
9. Storage -> IEE
