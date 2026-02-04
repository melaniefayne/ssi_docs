# Holder Binding — Conceptual Overview

Holder binding is the security property that ensures only the legitimate credential holder can present it. This document explains the mechanisms, their security properties, and how they integrate into the verification flow.

---

## The Binding Problem

Consider a credential as having two parts:
1. **Claims** — The assertions about the subject (name, age, qualification)
2. **Binding** — The link between the credential and the holder

```
┌─────────────────────────────────────────────────────┐
│  Credential                                          │
│  ┌─────────────────────────────────────────────────┐│
│  │  CLAIMS                                          ││
│  │  ┌─────────────────────────────────────────────┐││
│  │  │ given_name: "Alice"                         │││
│  │  │ family_name: "Smith"                        │││
│  │  │ birthdate: "1990-01-15"                     │││
│  │  │ nationality: "DE"                           │││
│  │  └─────────────────────────────────────────────┘││
│  └─────────────────────────────────────────────────┘│
│  ┌─────────────────────────────────────────────────┐│
│  │  BINDING                                         ││
│  │  ┌─────────────────────────────────────────────┐││
│  │  │ cnf: {                                      │││
│  │  │   jwk: {                                    │││
│  │  │     kty: "EC",                              │││
│  │  │     crv: "P-256",                           │││
│  │  │     x: "...",                               │││
│  │  │     y: "..."                                │││◄── Holder's
│  │  │   }                                         │││    public key
│  │  │ }                                           │││
│  │  └─────────────────────────────────────────────┘││
│  └─────────────────────────────────────────────────┘│
│  ┌─────────────────────────────────────────────────┐│
│  │  ISSUER SIGNATURE                                ││
│  │  Signs both claims AND binding                   ││
│  └─────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────┘
```

The issuer's signature covers both the claims and the binding, making them inseparable.

---

## Cryptographic Holder Binding

### At Issuance Time

1. Holder generates a key pair (or uses existing)
2. Holder sends public key to Issuer (in credential request proof)
3. Issuer embeds public key in credential
4. Issuer signs the complete credential

```
Holder                                Issuer
  │                                      │
  │  1. Generate key pair                │
  │     private_key, public_key          │
  │                                      │
  │  2. Credential Request               │
  │  ────────────────────────────────►   │
  │     proof signed with private_key    │
  │     contains public_key              │
  │                                      │
  │                                      │  3. Verify proof
  │                                      │  4. Create credential
  │                                      │     embed public_key
  │                                      │  5. Sign credential
  │                                      │
  │  6. Credential Response              │
  │  ◄────────────────────────────────   │
  │     credential with embedded key     │
  │                                      │
```

### At Presentation Time

1. Verifier sends request with nonce
2. Holder creates proof signed with private key
3. Holder sends credential + proof
4. Verifier validates signature matches embedded key

```
Holder                                Verifier
  │                                      │
  │  1. Presentation Request             │
  │  ◄────────────────────────────────   │
  │     nonce: "abc123"                  │
  │                                      │
  │  2. Create holder binding proof      │
  │     Sign(nonce + audience)           │
  │     with private_key                 │
  │                                      │
  │  3. Presentation Response            │
  │  ────────────────────────────────►   │
  │     credential + proof               │
  │                                      │
  │                                      │  4. Extract public_key
  │                                      │     from credential
  │                                      │  5. Verify proof signature
  │                                      │     with public_key
  │                                      │  6. Verify nonce matches
  │                                      │
```

---

## Format-Specific Mechanisms

### SD-JWT-VC: Key Binding JWT

SD-JWT uses a separate Key Binding JWT (kb-jwt) appended to the credential:

```
SD-JWT-VC Structure:
┌─────────────────────────────────────────────────────┐
│  Issuer JWT (signed by issuer)                       │
│  {                                                   │
│    "iss": "https://issuer.example.com",             │
│    "sub": "did:example:holder",                     │
│    "cnf": {                                         │
│      "jwk": { /* holder public key */ }             │
│    },                                                │
│    "_sd": [ /* selective disclosure hashes */ ]     │
│  }                                                   │
├─────────────────────────────────────────────────────┤
│  ~ (separator)                                       │
├─────────────────────────────────────────────────────┤
│  Disclosures (revealed claims)                       │
│  base64(salt + claim_name + claim_value)            │
├─────────────────────────────────────────────────────┤
│  ~ (separator)                                       │
├─────────────────────────────────────────────────────┤
│  Key Binding JWT (signed by holder)                  │
│  {                                                   │
│    "aud": "https://verifier.example.com",           │
│    "nonce": "abc123",                               │
│    "iat": 1683000000,                               │
│    "sd_hash": "hash of issuer JWT + disclosures"    │
│  }                                                   │
└─────────────────────────────────────────────────────┘
```

**Verification:**
1. Parse issuer JWT, extract `cnf.jwk`
2. Parse key binding JWT
3. Verify kb-jwt signature with `cnf.jwk`
4. Verify `nonce` matches request
5. Verify `aud` matches verifier
6. Verify `sd_hash` matches credential hash

### MSO-MDOC: Device Authentication

mDoc uses device authentication within the CBOR structure:

```
DeviceResponse:
┌─────────────────────────────────────────────────────┐
│  documents: [                                        │
│    {                                                 │
│      docType: "org.iso.18013.5.1.mDL",              │
│      issuerSigned: {                                 │
│        nameSpaces: { /* signed claims */ },         │
│        issuerAuth: { /* MSO + signature */ }        │
│      },                                              │
│      deviceSigned: {                                 │
│        nameSpaces: { /* device-added data */ },     │
│        deviceAuth: {                                 │
│          deviceSignature: COSE_Sign1(               │
│            SessionTranscript,                       │◄── Holder binding
│            deviceKey                                 │
│          )                                           │
│        }                                             │
│      }                                               │
│    }                                                 │
│  ]                                                   │
└─────────────────────────────────────────────────────┘
```

**SessionTranscript includes:**
- Device engagement
- Reader engagement
- Handover data (NFC/BLE session specifics)

### W3C VC: Verifiable Presentation Proof

W3C VCs use a proof on the Verifiable Presentation:

```json
{
  "@context": ["https://www.w3.org/2018/credentials/v1"],
  "type": ["VerifiablePresentation"],
  "holder": "did:key:z6Mk...",
  "verifiableCredential": [
    { /* VC with credentialSubject.id = holder DID */ }
  ],
  "proof": {
    "type": "Ed25519Signature2020",
    "created": "2024-01-15T12:00:00Z",
    "verificationMethod": "did:key:z6Mk...#key-1",
    "proofPurpose": "authentication",
    "challenge": "abc123",
    "domain": "https://verifier.example.com",
    "proofValue": "z..."
  }
}
```

**Verification:**
1. Resolve holder DID to get public key
2. Verify VP proof signature
3. Verify `challenge` matches request nonce
4. Verify `domain` matches verifier
5. Verify `holder` matches credential subject

---

## Nonce Security Properties

The nonce provides critical security guarantees:

### Replay Prevention

```
Without nonce:
┌────────────────┐         ┌────────────────┐
│ Attacker       │ Capture │ Legitimate     │
│ intercepts     │◄────────│ presentation   │
│ presentation   │         │                │
└───────┬────────┘         └────────────────┘
        │
        │ Replay
        ▼
┌────────────────┐
│ Verifier       │
│ accepts!       │ ← Attacker succeeds
└────────────────┘

With nonce:
┌────────────────┐         ┌────────────────┐
│ Attacker       │ Capture │ Presentation   │
│ intercepts     │◄────────│ nonce="abc"    │
└───────┬────────┘         └────────────────┘
        │
        │ Replay
        ▼
┌────────────────┐
│ Verifier       │
│ nonce="xyz"    │
│ Rejects!       │ ← Nonce mismatch
└────────────────┘
```

### Nonce Requirements

| Property | Requirement |
|----------|-------------|
| **Randomness** | Cryptographically random |
| **Length** | Minimum 128 bits entropy |
| **Single-use** | Each request gets fresh nonce |
| **Time-bound** | Short validity window |
| **Stored** | Verifier stores until validated |

---

## Hardware-Backed Keys

For strongest security, holder keys should be hardware-backed:

### Android Keystore

```kotlin
// Key generation in Keystore
val keyPairGenerator = KeyPairGenerator.getInstance(
    KeyProperties.KEY_ALGORITHM_EC,
    "AndroidKeyStore"
)

keyPairGenerator.initialize(
    KeyGenParameterSpec.Builder(
        "holder_key",
        KeyProperties.PURPOSE_SIGN
    )
    .setAlgorithmParameterSpec(ECGenParameterSpec("secp256r1"))
    .setDigests(KeyProperties.DIGEST_SHA256)
    .setUserAuthenticationRequired(true)  // Biometric required
    .setIsStrongBoxBacked(true)           // Hardware SE when available
    .build()
)
```

**Properties:**
- Private key never leaves secure hardware
- Biometric required for each use
- Hardware attestation proves key provenance

### iOS Secure Enclave

```swift
// Key generation in Secure Enclave
let access = SecAccessControlCreateWithFlags(
    kCFAllocatorDefault,
    kSecAttrAccessibleWhenUnlockedThisDeviceOnly,
    [.privateKeyUsage, .biometryCurrentSet],
    nil
)!

let attributes: [String: Any] = [
    kSecAttrKeyType as String: kSecAttrKeyTypeECSECPrimeRandom,
    kSecAttrKeySizeInBits as String: 256,
    kSecAttrTokenID as String: kSecAttrTokenIDSecureEnclave,
    kSecPrivateKeyAttrs as String: [
        kSecAttrIsPermanent as String: true,
        kSecAttrAccessControl as String: access
    ]
]

let privateKey = SecKeyCreateRandomKey(attributes as CFDictionary, nil)
```

**Properties:**
- Key material in Secure Enclave
- Face ID/Touch ID required
- Non-extractable

---

## Binding Without Keys

Some scenarios don't use cryptographic binding:

### Portrait Matching (mDL)

```
Credential contains:
- Portrait image (photo)

Verification:
- Display portrait
- Human or automated face comparison
- Match against presenter
```

### Knowledge-Based

```
Credential contains:
- Name, birthdate, address

Verification:
- Compare against other identity document
- Ask knowledge questions
```

### Limitations

Without cryptographic binding:
- Requires in-person verification
- Susceptible to skilled impersonation
- Cannot be done remotely securely

---

## Implementation Considerations

### When to Skip Holder Binding

OID4VP allows requesting without binding:

```json
{
  "dcql_query": {
    "credentials": [{
      "require_cryptographic_holder_binding": false
    }]
  }
}
```

**Use cases:**
- Low-value transactions
- Public information
- Bearer credentials (intentionally transferable)

### Authentication Before Presentation

Implementations typically require user authentication:

```
1. User triggers presentation
2. Biometric/PIN prompt
3. Key unlocked for signing
4. Proof created
5. Key locked again
```

This ensures the device owner authorized the presentation.
