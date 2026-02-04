# Proof Construction -- Conceptual Overview

A **proof** in the context of OID4VCI credential issuance is a cryptographic demonstration that the wallet holder controls the private key to which the credential will be bound. The proof is sent alongside the credential request, and the issuer verifies it before issuing the credential. Without a valid proof, the issuer has no assurance that the credential will be usable only by the intended holder.

---

## Why Proofs Exist

Credential issuance involves a fundamental trust problem: the issuer is about to create a cryptographically signed credential that attests to claims about the holder. If that credential is not bound to a specific key pair, anyone who obtains a copy of the credential can present it as their own. Proofs solve this problem by establishing a verifiable link between the credential and the holder's key material.

### The Threat Without Proofs

Without proof of possession, several attacks become possible:

- **Credential theft** -- An attacker who intercepts the credential response can use the credential without restriction, since nothing ties the credential to a specific holder.
- **Credential cloning** -- A credential without key binding can be copied and used by multiple parties simultaneously.
- **Man-in-the-middle issuance** -- An attacker who intercepts the credential request could substitute their own key material, causing the issuer to bind the credential to the attacker's key instead of the legitimate holder's key.

### What the Proof Achieves

A valid proof establishes three properties:

1. **Possession** -- The requester controls the private key corresponding to the public key that will be embedded in the credential.
2. **Freshness** -- The proof was created recently (enforced via the `c_nonce` from the token response and the `iat` timestamp), preventing replay of captured proofs.
3. **Context binding** -- The proof is bound to a specific issuer (via the `aud` claim) and a specific issuance session (via the `nonce` claim), preventing its use in a different context.

---

## Proof of Possession

Proof of possession (PoP) is the core concept. The holder proves they control the key that will be bound to the credential by signing a structured message with that key.

### How It Works

```
Holder                                  Issuer
  |                                       |
  |  1. Receive c_nonce (token response)  |
  |<--------------------------------------|
  |                                       |
  |  2. Construct proof JWT               |
  |     - Include c_nonce                 |
  |     - Include holder's public key     |
  |     - Sign with holder's private key  |
  |                                       |
  |  3. Send proof with credential request|
  |-------------------------------------->|
  |                                       |
  |  4. Issuer verifies:                  |
  |     - Signature valid                 |
  |     - c_nonce matches                 |
  |     - Public key acceptable           |
  |                                       |
  |  5. Issue credential bound to         |
  |     holder's public key               |
  |<--------------------------------------|
  |                                       |
```

The holder constructs a JWT that contains the `c_nonce` received from the issuer's token response and signs it with the private key corresponding to the public key they want bound to the credential. The issuer verifies the JWT signature against the included public key, confirms the nonce matches the one it issued, and then binds the credential to that public key.

### Key Binding

Once the issuer verifies the proof, it creates the credential with the holder's public key embedded or referenced within the credential structure. This is **key binding** -- the credential is cryptographically tied to the holder's key pair.

- In **mDoc (ISO 18013-5)** credentials, the holder's public key is included in the Mobile Security Object (MSO) as the `deviceKey`.
- In **SD-JWT VC** credentials, the holder's public key is included in the `cnf` (confirmation) claim.
- In **JSON-LD VC** credentials, the holder's public key is referenced in the `credentialSubject.id` as a DID.

After issuance, the holder must use the corresponding private key to prove they are the legitimate holder of the credential during presentation. If the private key is lost or compromised, the credential becomes unusable or must be revoked.

---

## JWT Proof Type

The most common proof type in OID4VCI is the **JWT proof**. The wallet constructs a signed JWT that serves as the proof of possession. This JWT has a specific structure defined by the OID4VCI specification.

### Why JWT

JWTs are widely supported, compact, and self-contained. They carry the public key (or a reference to it), the claims needed for freshness and context binding, and the cryptographic signature -- all in a single, transmissible token. The JWT format also aligns with existing OAuth 2.0 and OpenID Connect tooling, making implementation straightforward for ecosystems already built on these standards.

### Proof in the Credential Request

The proof is included in the credential request body as a `proof` object:

```json
{
  "format": "vc+sd-jwt",
  "vct": "VerifiablePortableDocumentA1",
  "proof": {
    "proof_type": "jwt",
    "jwt": "eyJhbGciOiJFUzI1NiIsInR5cCI6Im9wZW5pZDR2Y2ktcHJvb2Yrand0Ii..."
  }
}
```

For batch issuance (multiple credentials in a single request), the `proofs` (plural) field is used instead:

```json
{
  "format": "mso_mdoc",
  "doctype": "org.iso.18013.5.1.mDL",
  "proofs": {
    "jwt": [
      "eyJhbGciOiJFUzI1NiIs...",
      "eyJhbGciOiJFUzI1NiIs..."
    ]
  }
}
```

Each JWT in the array corresponds to a distinct key pair, allowing the issuer to bind each credential instance to a different key.

---

## DPoP and Key Binding JWT

Beyond the primary credential request proof, two related proof mechanisms appear in the issuance flow.

### DPoP Proof

Demonstrating Proof of Possession (DPoP, RFC 9449) is a separate mechanism that binds the access token to the client's key pair. A DPoP proof is a JWT sent in the `DPoP` HTTP header with each request to the token endpoint and credential endpoint. It proves that the entity presenting the access token is the same entity that originally obtained it.

DPoP proofs are distinct from credential request proofs. The DPoP proof binds the access token to a transport key, while the credential request proof binds the credential to a holder key. These may be the same key or different keys, depending on the implementation.

### Key Binding JWT in SD-JWT

When an SD-JWT credential is presented, the holder may include a **Key Binding JWT (KB-JWT)** that proves possession of the key bound to the credential. The SD-JWT is structured as:

```
<issuer-signed-jwt>~<disclosure1>~<disclosure2>~<kb-jwt>
```

The KB-JWT is signed with the holder's private key (the same key proven during issuance) and includes the audience (verifier) and nonce for the presentation context. This is not part of the issuance flow itself, but it is the downstream consequence of the key binding established during proof construction at issuance time.

---

## User Authentication and Proof Construction

A critical implementation concern is how the holder's private key is protected. The private key used for proof construction is typically stored in a hardware-backed secure element (Android Keystore, iOS Secure Enclave) and requires user authentication before it can be used for signing.

This means proof construction is not purely a software operation -- it involves the user:

- **Biometric authentication** -- The user presents a fingerprint or face to unlock the signing key (EUDI Android, EUDI iOS).
- **PIN entry** -- The user enters a PIN to authorize a signing operation (Procivis ONE with Remote Signing Element).
- **Device credential** -- The user provides their device lock screen credential (pattern, PIN, or password) as a fallback when biometrics are unavailable.

The user authentication step is what makes the proof meaningful beyond cryptography. It establishes that a human, not just a device, authorized the credential request.

---

## Summary

Proofs are the mechanism by which the OID4VCI protocol ensures that credentials are bound to the entity that requested them. The JWT proof type is the dominant approach: the holder signs a JWT containing the issuer's nonce and their own public key, and the issuer verifies this before issuing the credential. The security of this mechanism depends on the protection of the holder's private key, which is typically gated behind user authentication (biometrics or PIN) and stored in hardware-backed secure storage.

After the proof is constructed, it is included in the credential request sent to the issuer. The construction of this request is covered in [08 -- Credential Request](../08-credential-request/).
