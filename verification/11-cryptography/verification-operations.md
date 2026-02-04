# Verification Cryptographic Operations

This document details the cryptographic operations performed during credential verification.

---

## Overview of Operations

```
┌─────────────────────────────────────────────────────────────────┐
│              VERIFICATION CRYPTOGRAPHIC OPERATIONS               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  WALLET SIDE (Creating Presentation):                            │
│  1. Retrieve credential from secure storage                      │
│  2. Generate holder proof (sign with holder key)                 │
│  3. Apply selective disclosure (if applicable)                   │
│  4. Construct VP with embedded proof                             │
│                                                                  │
│  VERIFIER SIDE (Validating Presentation):                        │
│  1. Verify VP structure                                          │
│  2. Verify holder proof signature                                │
│  3. Verify VC issuer signature                                   │
│  4. Verify disclosure integrity (SD-JWT/mDoc)                    │
│  5. Verify nonce binding                                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Holder Proof Generation

### JWT VP Proof

The wallet signs a VP JWT:

```
Header:
{
  "alg": "ES256",
  "typ": "JWT",
  "kid": "did:key:z6Mk...#key-1"
}

Payload:
{
  "iss": "did:key:z6Mk...",
  "aud": "https://verifier.example.com",
  "nonce": "n-0S6_WzA2Mj",
  "iat": 1683000000,
  "exp": 1683000300,
  "vp": {
    "@context": ["https://www.w3.org/2018/credentials/v1"],
    "type": ["VerifiablePresentation"],
    "verifiableCredential": ["eyJ..."]
  }
}

Signature: ES256(header.payload, holderPrivateKey)
```

### SD-JWT Key Binding JWT

For SD-JWT credentials, holder signs a Key Binding JWT:

```
Header:
{
  "alg": "ES256",
  "typ": "kb+jwt"
}

Payload:
{
  "iat": 1683000000,
  "aud": "https://verifier.example.com",
  "nonce": "n-0S6_WzA2Mj",
  "sd_hash": "fUMPJzLqf..."
}

Signature: ES256(header.payload, holderPrivateKey)
```

Where `sd_hash` = SHA-256(SD-JWT before key binding JWT)

### mDoc Device Authentication

For mDoc, device signs the session transcript:

```cbor
DeviceAuthentication = [
  "DeviceAuthentication",
  SessionTranscript,
  DocType,
  DeviceNameSpacesBytes
]

DeviceSignature = COSE_Sign1(
  protected: { alg: ES256 },
  payload: DeviceAuthentication,
  key: devicePrivateKey
)
```

---

## Signature Verification

### ES256 Verification

```
Input:
  - message (header.payload for JWT)
  - signature (r || s, 64 bytes)
  - publicKey (P-256 point)

Algorithm:
  1. Decode signature as (r, s) integers
  2. Compute message hash: h = SHA-256(message)
  3. Compute s_inv = s^(-1) mod n
  4. Compute u1 = h * s_inv mod n
  5. Compute u2 = r * s_inv mod n
  6. Compute point R = u1*G + u2*publicKey
  7. Verify R.x mod n == r
```

### EdDSA (Ed25519) Verification

```
Input:
  - message
  - signature (R || s, 64 bytes)
  - publicKey (32 bytes)

Algorithm:
  1. Decode R as point, s as scalar
  2. Compute h = SHA-512(R || publicKey || message)
  3. Reduce h mod L to get k
  4. Verify 8*s*B == 8*R + 8*k*publicKey
```

---

## Key Resolution

### From DID

```
DID: did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK

Resolution:
  1. Decode multibase: z6Mk... → bytes
  2. Decode multicodec: 0xed01 → Ed25519 public key
  3. Extract raw public key: 32 bytes
```

### From JWK

```json
{
  "kty": "EC",
  "crv": "P-256",
  "x": "WbbKn2...",
  "y": "GUph8B..."
}
```

Convert to public key point on P-256 curve.

### From X.509 Certificate

```
1. Parse certificate ASN.1 structure
2. Extract SubjectPublicKeyInfo
3. Parse algorithm identifier (EC P-256)
4. Extract public key bytes
```

---

## Disclosure Verification

### SD-JWT Disclosure Verification

```
SD-JWT: issuer-jwt~disclosure1~disclosure2~kb-jwt

For each disclosure:
  1. Decode: base64url(disclosure) → ["salt", "name", "value"]
  2. Compute hash: SHA-256(disclosure)
  3. Find hash in _sd array of issuer-jwt payload
  4. If not found, disclosure is invalid
```

### mDoc Digest Verification

```
MSO contains:
  valueDigests: {
    "org.iso.18013.5.1": {
      0: h'a1b2c3...',  // digest of element 0
      1: h'd4e5f6...',  // digest of element 1
      ...
    }
  }

For each disclosed element:
  1. Compute: digest = SHA-256(CBOR(element))
  2. Compare with MSO valueDigests[namespace][elementId]
  3. Must match exactly
```

---

## Nonce Verification

### Purpose

Nonce prevents replay attacks:

```
Without nonce:
  Attacker captures VP → Attacker replays VP → Verifier accepts

With nonce:
  Verifier: Generate random nonce, store in session
  Holder: Include nonce in VP proof
  Verifier: Check VP.nonce == session.nonce
  Verifier: Delete nonce from session
  Attacker: Cannot replay (nonce already consumed)
```

### Requirements

| Property | Requirement |
|----------|-------------|
| Length | ≥ 128 bits entropy |
| Randomness | CSPRNG |
| Storage | Server-side session |
| Lifetime | Short (minutes) |
| Usage | Single-use only |

---

## Session Key Derivation (Proximity)

### ISO 18013-5 Session Keys

```
ECDH:
  EDeviceKey = device ephemeral key pair
  EReaderKey = reader ephemeral key pair

  sharedSecret = ECDH(EDeviceKey.private, EReaderKey.public)
                = ECDH(EReaderKey.private, EDeviceKey.public)

HKDF:
  SKDevice = HKDF-SHA256(
    salt: SHA-256(SessionTranscript),
    IKM: sharedSecret,
    info: "SKDevice",
    L: 32
  )

  SKReader = HKDF-SHA256(
    salt: SHA-256(SessionTranscript),
    IKM: sharedSecret,
    info: "SKReader",
    L: 32
  )
```

### Message Encryption

```
AES-256-GCM encryption:
  Device → Reader: Encrypt with SKDevice
  Reader → Device: Encrypt with SKReader

  nonce: Counter-based (incremented per message)
  AAD: Additional authenticated data (none in basic mode)
```

---

## Trust Chain Verification

### X.509 Chain

```
1. Build chain: [leaf, intermediate, ..., root]
2. For each pair (cert, issuer):
   - Verify cert.signature with issuer.publicKey
   - Check cert.notBefore ≤ now ≤ cert.notAfter
   - Check issuer.basicConstraints.cA == true
   - Check issuer.keyUsage includes keyCertSign
3. Verify root is in trust store
4. Check revocation (CRL or OCSP)
```

### DID Chain (if attestation)

```
1. Resolve issuer DID to DID Document
2. Extract verification method
3. Verify VC signature with verification method key
4. Check issuer against trust registry
```

---

## Algorithm Support Summary

| Algorithm | VP Signing | VC Verification | Session | Status |
|-----------|------------|-----------------|---------|--------|
| ES256 | ✓ | ✓ | ✗ | Required |
| ES384 | ✓ | ✓ | ✗ | Optional |
| ES512 | ✗ | ✓ | ✗ | Optional |
| EdDSA | ✓ | ✓ | ✗ | Recommended |
| ECDH-ES | ✗ | ✗ | ✓ | For proximity |
| AES-256-GCM | ✗ | ✗ | ✓ | For proximity |

---

## Security Considerations

### Timing Attacks

```
Constant-time comparison for:
- Signature verification
- Hash comparison
- Nonce comparison

Use crypto library functions, not direct ==
```

### Key Storage

```
Holder keys must be:
- Hardware-backed when possible
- Protected by user authentication
- Never exported in plaintext
```

### Randomness

```
All random values (nonces, ephemeral keys) must use:
- /dev/urandom or equivalent
- Platform CSPRNG
- Never predictable or low-entropy sources
```
