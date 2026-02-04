# What Is SSI Credential Issuance?

Credential issuance is the process by which a trusted authority creates a digitally signed credential and delivers it to the individual (or entity) it describes. In Self-Sovereign Identity (SSI), this process is designed so that the recipient — not the issuer and not the verifier — retains control over the credential after it has been issued.

This document explains the conceptual model. No implementation details, no code, no protocol-level specifics. Those come in sections 01 through 12.

---

## The Trust Triangle

SSI credential issuance operates within a three-party trust model:

```
           +---------+
           | Issuer  |
           +----+----+
                |
        issues credential to
                |
                v
           +---------+        presents credential to        +-----------+
           | Holder  | --------------------------------------> | Verifier  |
           +---------+                                       +-----------+
```

### Issuer

The entity that creates and cryptographically signs the credential. The issuer asserts that the claims within the credential are true at the time of issuance. Examples include government agencies (issuing national IDs), universities (issuing diplomas), employers (issuing proof of employment), and healthcare providers (issuing vaccination records).

The issuer's authority is not inherent in the protocol. A verifier must independently decide whether to trust a given issuer, typically by recognizing the issuer's Decentralized Identifier (DID) or public key through a trust framework or registry.

### Holder

The entity that receives, stores, and later presents the credential. In most SSI systems, the holder is the subject of the credential — the person the claims are about. The holder stores the credential in a digital wallet (a mobile app, a cloud vault, or both) and decides when, to whom, and which parts of the credential to disclose.

This is the "self-sovereign" element: the holder is not dependent on the issuer to prove something about themselves after issuance. The credential is a self-contained, portable, cryptographically verifiable artifact.

### Verifier

The entity that receives a credential presentation from the holder and checks its validity. Verification involves confirming that the credential was signed by a trusted issuer, that it has not been tampered with, that it has not been revoked, and that the presenter is the legitimate holder (holder binding).

Verification is out of scope for this documentation, but it matters here because the design of the issuance flow directly determines what the verifier can later check. For example, if issuance does not bind the credential to a holder-controlled key, the verifier cannot confirm holder binding at presentation time.

---

## What Is a Verifiable Credential?

A Verifiable Credential (VC) is a tamper-evident digital assertion made by an issuer about a subject. The W3C Verifiable Credentials Data Model (v1.1 and v2.0) defines the standard data structure.

At its core, a VC contains:

- **Metadata**: Who issued it, when, what type it is, its unique identifier, and its expiration or validity period.
- **Claims**: The actual assertions about the subject. For example, `"givenName": "Alice"` or `"dateOfBirth": "1990-01-15"`. Claims are key-value pairs grouped under a `credentialSubject`.
- **Proof**: A cryptographic signature (or set of signatures) that binds the metadata and claims together. The proof allows any party to verify that the credential was issued by the stated issuer and has not been modified.

### Credential Formats

The W3C VC Data Model is a logical model. It can be serialized in multiple formats, each with different properties:

| Format | Description | Key Property |
|--------|-------------|--------------|
| **JSON-LD** | VC expressed as a JSON-LD document with `@context` for semantic interoperability. Proofs are typically linked data signatures (e.g., `EcdsaSecp256k1Signature2019`). | Semantic precision, interoperability across contexts. |
| **SD-JWT-VC** | VC expressed as a JWT with selective disclosure. Individual claims can be hidden or revealed at presentation time using salted hashes. | Selective disclosure without complex ZKP math. |
| **mDoc (ISO 18013-5)** | VC expressed as a CBOR-encoded mobile document, originally designed for mobile driving licenses. Uses COSE signatures. | Optimized for offline, proximity-based presentation. Hardware-friendly. |
| **JSON-LD with BBS+** | VC expressed as JSON-LD with BBS+ signatures, enabling selective disclosure and zero-knowledge proof of credential possession. | Unlinkable presentations, selective disclosure. |

Different implementations support different subsets of these formats. This is one of the key comparison axes throughout this documentation.

---

## The Issuance Flow at a High Level

Credential issuance is not a single request-response exchange. It is a multi-step protocol that establishes trust between the issuer and the wallet before a credential is created and delivered.

The following sequence represents the generalized flow used by OID4VCI (OpenID for Verifiable Credential Issuance), which all three implementations in this documentation follow in some form:

```
  Issuer                                Wallet (Holder)
    |                                        |
    |  1. Credential Offer                   |
    |  ------------------------------------> |
    |     (QR code, deep link, or push)      |
    |                                        |
    |  2. Issuer Metadata Discovery          |
    |  <------------------------------------ |
    |     (GET /.well-known/...)             |
    |  ------------------------------------> |
    |     (credential configs, display info) |
    |                                        |
    |  3. Authorization                      |
    |  <------------------------------------ |
    |     (OAuth 2.0 / pre-auth code + PIN)  |
    |  ------------------------------------> |
    |     (access token, c_nonce)            |
    |                                        |
    |  4. Proof Construction                 |
    |        (wallet-side, no network)       |
    |     [sign c_nonce with holder key]     |
    |                                        |
    |  5. Credential Request                 |
    |  <------------------------------------ |
    |     (POST /credential with proof)      |
    |  ------------------------------------> |
    |     (signed credential or deferred ID) |
    |                                        |
    |  6. Secure Storage                     |
    |        (wallet-side, no network)       |
    |     [store credential + keys locally]  |
    |                                        |
```

### Step 1 — Credential Offer

The issuer initiates the flow by creating a credential offer. This is typically delivered as a QR code scanned by the wallet, a deep link opened by the wallet, or a pushed notification. The offer contains enough information for the wallet to know which issuer to contact and which credential type is being offered.

### Step 2 — Issuer Metadata Discovery

The wallet retrieves the issuer's metadata from well-known endpoints. This metadata describes the issuer's capabilities: which credential types it can issue, which formats it supports, which authorization mechanisms it requires, and how to display the credential to the user.

### Step 3 — Authorization and Token Exchange

The wallet authenticates with the issuer. This may involve a full OAuth 2.0 authorization code flow (where the user logs in via a browser) or a pre-authorized code flow (where the issuer has already authenticated the user and provides a code directly in the offer). The result is an access token and, critically, a `c_nonce` — a cryptographic nonce that the wallet must include in its proof to prevent replay attacks.

### Step 4 — Proof Construction

The wallet constructs a proof-of-possession: a signed object (typically a JWT) that proves the wallet controls a specific cryptographic key. This proof includes the `c_nonce` from the issuer, binding it to this specific issuance session. The issuer will embed the corresponding public key into the credential, establishing holder binding.

### Step 5 — Credential Request and Response

The wallet sends a credential request to the issuer, including the proof. The issuer validates the proof, constructs the credential with the holder's public key bound into it, signs it with the issuer's private key, and returns it. In some cases, the issuer returns a deferred issuance identifier instead, indicating that the credential will be available later.

### Step 6 — Secure Storage

The wallet stores the received credential and its associated private key in secure local storage. On mobile platforms, this typically means the credential is encrypted in an application database and the private key is stored in hardware-backed secure storage (Android Keystore, iOS Secure Enclave). The credential is now ready for presentation.

---

## Why Issuance Matters

Issuance is the origin event for every credential in the SSI ecosystem. Decisions made during issuance have permanent downstream consequences:

- **Holder binding**: If the credential is not bound to a holder-controlled key at issuance, it can be freely transferred or replayed by anyone who obtains a copy. Verifiers cannot confirm that the presenter is the legitimate holder.

- **Format choice**: The credential format chosen at issuance determines what disclosure capabilities are available at presentation. An mDoc credential supports proximity-based presentation. An SD-JWT-VC supports selective disclosure. A JSON-LD credential with BBS+ signatures supports unlinkable presentations. These properties cannot be added retroactively.

- **Key security**: The security of the holder's private key — whether it is stored in hardware, in software, or in a remote HSM — is established during issuance when the key pair is generated and the public key is embedded in the credential. A key that is extractable at issuance time remains extractable forever.

- **Trust chain**: The cryptographic link between the issuer's signature and the credential content is forged at issuance. If the issuer's signing infrastructure is compromised or misconfigured during issuance, every credential produced is tainted.

- **Revocation capability**: Whether and how a credential can be revoked is determined by the issuer at issuance time, based on the revocation mechanism embedded in the credential (status list, accumulator, or none).

Everything that follows in this documentation — architecture, protocol details, cryptographic mechanisms, storage strategies — is in service of getting issuance right.

---

**Next**: [Implementations Overview](./implementations-overview.md) — The three systems analyzed in this documentation.
