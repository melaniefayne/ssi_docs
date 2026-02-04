# Glossary

Definitions of terms used throughout the SSI Verification documentation.

---

## A

### Authorization Request
The initial message from a Verifier to a Holder's wallet requesting credential presentation. In OID4VP, this extends the OAuth 2.0 Authorization Request with verifiable credential-specific parameters.

### Attestation
A signed statement vouching for something. In SSI, can refer to:
- **Key attestation**: Proof that a key is stored in secure hardware
- **Verifier attestation**: Proof that a verifier is authorized to request credentials

---

## B

### BLE (Bluetooth Low Energy)
A wireless protocol used for proximity-based credential presentation. Enables offline verification when the Holder and Verifier devices are physically close.

### Binding (Holder Binding)
The cryptographic link between a credential and its holder. Prevents credential transfer or impersonation.

---

## C

### Claim
A single piece of information asserted about a subject. Example: "given_name": "Alice".

### Client ID
The identifier for a Verifier in OID4VP. Format depends on the client_id_scheme (redirect_uri, x509_san_dns, DID, etc.).

### Credential
See Verifiable Credential.

### Cryptographic Holder Binding
Proof of credential possession through cryptographic signature. The holder proves control of a private key embedded in the credential.

---

## D

### DCQL (Digital Credentials Query Language)
A query language for specifying credential requirements in OID4VP v1.0. Simpler alternative to Presentation Exchange.

### DID (Decentralized Identifier)
A globally unique identifier that does not require a central registration authority. Resolves to a DID Document containing public keys and service endpoints.

### direct_post
An OID4VP response mode where the wallet sends the presentation directly to the verifier's endpoint via HTTP POST.

---

## E

### EUDI
European Digital Identity. The EU's framework for digital identity wallets under eIDAS 2.0 regulation.

---

## F

### Format
The serialization format of a credential. Common formats:
- SD-JWT-VC: Selective Disclosure JWT Verifiable Credential
- MSO-MDOC: Mobile Security Object Mobile Document (ISO 18013-5)
- JWT-VC: JWT-encoded Verifiable Credential
- JSON-LD VC: Linked Data Verifiable Credential

---

## H

### Holder
The entity that possesses and presents credentials. Typically the subject of the credential claims.

### Holder Binding
See Binding.

### HSM (Hardware Security Module)
Dedicated hardware for secure key storage and cryptographic operations.

---

## I

### Input Descriptor
A component of a Presentation Definition that describes a single credential requirement.

### Issuer
The entity that creates and signs credentials. Asserts that claims are true at issuance time.

---

## J

### JSONPath
A query language for extracting data from JSON documents. Used in Presentation Exchange to locate claim values.

---

## K

### Key Binding JWT (kb-jwt)
In SD-JWT, the JWT signed by the holder that proves possession. Contains the nonce and audience.

### Keystore
Secure storage for cryptographic keys. On Android, the Android Keystore; on iOS, the Secure Enclave.

---

## M

### mDL (Mobile Driving License)
A digital driver's license following ISO 18013-5. Uses MSO-MDOC format.

### MSO (Mobile Security Object)
The signed data structure in an mDoc that contains claim hashes and issuer signature.

---

## N

### NFC (Near Field Communication)
A short-range wireless protocol for proximity communication. Used for tap-to-present scenarios.

### Nonce
A single-use random value. In OID4VP, prevents replay attacks by binding the presentation to a specific request.

---

## O

### OID4VP (OpenID for Verifiable Presentations)
The OpenID Foundation specification for requesting and receiving verifiable credentials. Extends OAuth 2.0.

### OID4VCI (OpenID for Verifiable Credential Issuance)
The counterpart to OID4VP for credential issuance.

---

## P

### PEX (Presentation Exchange)
DIF specification for expressing credential requirements. Defines Presentation Definition and Presentation Submission formats.

### Presentation
See Verifiable Presentation.

### Presentation Definition
A JSON structure specifying what credentials and claims a Verifier requires. Contains input descriptors and optional submission requirements.

### Presentation Submission
A JSON structure mapping between the Presentation Definition and the actual credentials being presented.

### Proof (Holder Proof)
The cryptographic signature proving the holder authorized the presentation.

### Proximity
Verification scenarios where Holder and Verifier are physically co-located. Uses BLE or NFC transport.

---

## R

### Relying Party
See Verifier.

### Replay Attack
Attempting to reuse a previously captured presentation. Prevented by nonce verification.

### Response Mode
How the wallet returns the presentation. Options: direct_post, fragment, direct_post.jwt.

### RSE (Remote Secure Element)
A cloud-based secure element for key storage and signing. Used by Procivis ONE.

---

## S

### SD-JWT (Selective Disclosure JWT)
A JWT format that enables revealing only selected claims while hiding others.

### Secure Enclave
Apple's hardware security module in iOS devices. Provides secure key storage.

### Selective Disclosure
The ability to reveal only specific claims from a credential. Depends on credential format support.

### SIOPv2 (Self-Issued OpenID Provider v2)
A protocol allowing users to act as their own OpenID Provider. Often combined with OID4VP.

### State
An opaque value echoed between request and response for session correlation.

---

## T

### Trust Framework
A governance structure defining which issuers and verifiers are trusted, and under what conditions.

### Trust Store
A collection of trusted certificates or keys used to validate verifier identity.

---

## V

### Vault (Affinidi Vault)
Affinidi's cloud-based credential wallet.

### VC
See Verifiable Credential.

### Verifiable Credential (VC)
A tamper-evident digital assertion made by an issuer about a subject. Contains claims and issuer signature.

### Verifiable Presentation (VP)
A container for one or more verifiable credentials, signed by the holder for a specific verifier.

### Verifier
The entity that requests and validates credential presentations. Also called Relying Party.

### VP Token
The response parameter in OID4VP containing the Verifiable Presentation.

---

## W

### Wallet
The software application that stores credentials and manages presentations. May be mobile app, web app, or cloud service.
