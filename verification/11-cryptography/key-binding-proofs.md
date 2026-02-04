# Key Binding Proofs

This document details the cryptographic mechanisms for proving holder control of credential-bound keys.

---

## Why Key Binding Matters

Key binding prevents credential theft:

```
WITHOUT Key Binding:
┌───────────┐         ┌───────────┐         ┌───────────┐
│  Issuer   │──VC────►│  Holder   │──VC────►│ Attacker  │
│           │         │           │         │   steals  │
└───────────┘         └───────────┘         └───────────┘
                                                  │
                                                  ▼
                                            ┌───────────┐
                                            │ Verifier  │ ← Accepts stolen VC
                                            │ deceived  │
                                            └───────────┘

WITH Key Binding:
┌───────────┐         ┌───────────┐         ┌───────────┐
│  Issuer   │──VC────►│  Holder   │──VC────►│ Attacker  │
│ binds key │         │ has key   │         │ no key!   │
└───────────┘         └───────────┘         └───────────┘
                                                  │
                                                  ▼
                                            ┌───────────┐
                                            │ Verifier  │ ← Rejects (no proof)
                                            │ protected │
                                            └───────────┘
```

---

## Key Binding at Issuance

### Confirmation Claim (`cnf`)

The credential includes the holder's public key:

```json
{
  "iss": "https://issuer.example.com",
  "sub": "user123",
  "credentialSubject": { ... },
  "cnf": {
    "jwk": {
      "kty": "EC",
      "crv": "P-256",
      "x": "WbbKn2...",
      "y": "GUph8B..."
    }
  }
}
```

Alternatives to `jwk`:

| Method | Format | Example |
|--------|--------|---------|
| `jwk` | Embedded JWK | `{"jwk": {...}}` |
| `kid` | Key ID reference | `{"kid": "key123"}` |
| `jkt` | JWK thumbprint | `{"jkt": "NzbLsXh8..."}` |

### mDoc Device Key

```cbor
MobileSecurityObject = {
  "deviceKeyInfo": {
    "deviceKey": {
      1: 2,        ; kty: EC
      -1: 1,       ; crv: P-256
      -2: h'...',  ; x coordinate
      -3: h'...'   ; y coordinate
    }
  }
}
```

---

## Key Binding Proof Types

### 1. JWT VP Signature

Holder signs the entire VP JWT:

```
VP JWT = {
  header: { alg: "ES256", typ: "JWT" },
  payload: {
    iss: "did:key:z6Mk...",  // Holder DID
    aud: "https://verifier.example.com",
    nonce: "n-0S6_WzA2Mj",
    vp: {
      verifiableCredential: [...]
    }
  }
}

Signature with holder's private key proves control.
```

**Verification:**
1. Extract `iss` from VP
2. Resolve holder's public key
3. Verify VP signature
4. Compare holder key with credential's `cnf`

### 2. SD-JWT Key Binding JWT

Separate JWT proving key possession:

```
KB-JWT = {
  header: { alg: "ES256", typ: "kb+jwt" },
  payload: {
    iat: 1683000000,
    aud: "https://verifier.example.com",
    nonce: "n-0S6_WzA2Mj",
    sd_hash: "fUMPJzLqf..."  // Hash of SD-JWT before KB-JWT
  }
}
```

**`sd_hash` computation:**
```
sd_jwt_without_kb = "issuer-jwt~disclosure1~disclosure2"
sd_hash = base64url(SHA-256(ASCII(sd_jwt_without_kb)))
```

**Verification:**
1. Decode KB-JWT
2. Verify signature with key from credential's `cnf.jwk`
3. Check `nonce` matches request
4. Check `aud` matches verifier
5. Verify `sd_hash` matches presented SD-JWT

### 3. mDoc Device Authentication

COSE signature over session data:

```cbor
DeviceAuthentication = [
  "DeviceAuthentication",
  SessionTranscript,
  DocType,
  DeviceNameSpacesBytes
]

DeviceSignature = COSE_Sign1(
  protected: { 1: -7 },  // alg: ES256
  payload: nil,          // detached
  signature: ...
)

// External AAD contains DeviceAuthentication
```

**Verification:**
1. Reconstruct `DeviceAuthentication` from session
2. Verify COSE_Sign1 with `deviceKey` from MSO
3. Session transcript binds to this specific session

### 4. Data Integrity Proof

For JSON-LD presentations:

```json
{
  "@context": [...],
  "type": ["VerifiablePresentation"],
  "verifiableCredential": [...],
  "proof": {
    "type": "DataIntegrityProof",
    "cryptosuite": "ecdsa-rdfc-2019",
    "verificationMethod": "did:key:z6Mk...#key-1",
    "proofPurpose": "authentication",
    "challenge": "n-0S6_WzA2Mj",
    "domain": "https://verifier.example.com",
    "proofValue": "z58DAdF..."
  }
}
```

---

## Challenge-Response Flow

```
┌────────────┐                           ┌────────────┐
│  Verifier  │                           │   Wallet   │
└─────┬──────┘                           └─────┬──────┘
      │                                        │
      │  1. Generate nonce                     │
      │     nonce = random(256 bits)           │
      │     store(session_id → nonce)          │
      │                                        │
      │  2. Send request with nonce            │
      │ ──────────────────────────────────────►│
      │     { nonce: "n-0S6_WzA2Mj", ... }     │
      │                                        │
      │                                        │  3. Sign with holder key
      │                                        │     including nonce
      │                                        │
      │  4. VP with proof                      │
      │ ◄──────────────────────────────────────│
      │     { proof.challenge: "n-0S6_WzA2Mj" }│
      │                                        │
      │  5. Verify:                            │
      │     - Signature valid                  │
      │     - nonce == stored nonce            │
      │     - Key matches credential cnf       │
      │                                        │
```

---

## Multi-Credential Binding

When presenting multiple credentials:

### Same Holder Key

```
VC1.cnf.jwk = holder_key
VC2.cnf.jwk = holder_key  // Same key

VP signed with holder_key proves control of both
```

### Different Holder Keys

```
VC1.cnf.jwk = holder_key_1
VC2.cnf.jwk = holder_key_2  // Different keys

Options:
1. Separate proofs for each key
2. Multi-signature scheme
3. Linked credentials (advanced)
```

---

## Security Properties

### Proof of Possession (PoP)

The holder proves they possess the private key:

```
Property: Only key holder can generate valid signature
Attack: Attacker without key cannot create proof
Result: Credential theft is useless without key
```

### Freshness

The proof is bound to current session:

```
Property: Nonce prevents replay
Attack: Old proof cannot be reused
Result: Each verification requires new proof
```

### Audience Binding

The proof is bound to specific verifier:

```
Property: Audience claim prevents misdirection
Attack: Cannot redirect proof to different verifier
Result: Proof valid only for intended recipient
```

---

## Hardware-Backed Key Binding

### Secure Enclave / TEE

```
Key Generation:
  Private key generated in hardware
  Private key never leaves secure element
  Only public key exported to credential

Signing:
  Signature computed in hardware
  User authentication required (biometric/PIN)
  Protected from malware on device
```

### Key Attestation

Proves key is hardware-backed:

```json
{
  "cnf": {
    "jwk": { ... },
    "key_attestation": "eyJ..."  // Platform attestation JWT
  }
}
```

Verifier can validate:
- Key is in secure hardware
- Device meets security requirements
- Key cannot be exported

---

## Implementation Patterns

### EUDI Key Binding Flow

```kotlin
// Generate key-bound proof
val disclosedDocuments = listOf(
    DisclosedDocument(
        documentId = credential.id,
        disclosedItems = selectedItems,
        keyUnlockData = biometricUnlock  // Unlocks key for signing
    )
)

// SDK generates VP with proof
val response = processedRequest.generateResponse(disclosedDocuments)
```

### Procivis Key Binding

```typescript
// Key binding via core library
await core.holderSubmitProof(interactionId, {
    credentialId: credential.id,
    submitClaims: selectedClaims
});
// Core handles proof generation with holder key
```

---

## Common Pitfalls

| Pitfall | Problem | Solution |
|---------|---------|----------|
| No key binding | Credentials can be stolen | Always use `cnf` |
| Weak nonce | Replay attacks possible | Use CSPRNG, 256 bits |
| No audience | Proof misdirection | Always include `aud` |
| Exported keys | Keys can be copied | Use hardware-backed |
| Missing expiry | Old proofs accepted | Check `iat`, short validity |
