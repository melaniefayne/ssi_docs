# Holder Binding — Protocol and Standards

This document details the protocol-level specifications for holder binding proofs in different credential formats.

---

## JWT VP Holder Binding

### VP JWT Structure

```
Header:
{
  "alg": "ES256",
  "typ": "JWT",
  "kid": "did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK#key-1"
}

Payload:
{
  "iss": "did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK",
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

Signature: ES256(base64url(header).base64url(payload), holderPrivateKey)
```

### Verification Steps

1. Decode JWT header and payload
2. Resolve `iss` to holder's public key
3. Verify JWT signature with holder's key
4. Check `nonce` matches request
5. Check `aud` matches verifier
6. Check `iat` and `exp` timing

---

## SD-JWT Key Binding JWT

### Structure

```
SD-JWT Presentation:
  issuer-jwt~disclosure1~disclosure2~kb-jwt

KB-JWT:
{
  "alg": "ES256",
  "typ": "kb+jwt"
}.{
  "iat": 1683000000,
  "aud": "https://verifier.example.com",
  "nonce": "n-0S6_WzA2Mj",
  "sd_hash": "fUMPJzLqf..."
}.[signature]
```

### sd_hash Computation

```
sd_jwt_without_kb = "eyJ...issuer-jwt...~disclosure1~disclosure2"
sd_hash = base64url(SHA-256(ASCII(sd_jwt_without_kb)))
```

### Verification Steps

1. Split SD-JWT into parts
2. Extract KB-JWT (last part after final `~`)
3. Decode KB-JWT header and payload
4. Get holder key from credential's `cnf` claim
5. Verify KB-JWT signature
6. Verify `nonce` matches request
7. Verify `aud` matches verifier
8. Compute `sd_hash` and verify match

---

## mDoc Device Authentication

### Device Authentication Structure (CBOR)

```cbor
DeviceAuthentication = [
  "DeviceAuthentication",
  SessionTranscript,
  DocType,
  DeviceNameSpacesBytes
]

SessionTranscript = [
  DeviceEngagementBytes,  ; CBOR-encoded DeviceEngagement
  EReaderKeyBytes,        ; CBOR-encoded EReaderKey
  Handover               ; Protocol-specific handover
]
```

### COSE_Sign1 for Device Signature

```cbor
COSE_Sign1 = [
  protected: bstr .cbor {
    1: -7  ; alg: ES256
  },
  unprotected: {},
  payload: nil,  ; Detached payload
  signature: bstr
]

; External AAD = DeviceAuthentication
```

### Verification Steps

1. Reconstruct `DeviceAuthentication` from session
2. Get `deviceKey` from MSO in `IssuerAuth`
3. Verify COSE_Sign1 signature
4. External AAD is `DeviceAuthentication`
5. Verify session transcript matches engagement

---

## Data Integrity Proofs

### VP with Data Integrity Proof

```json
{
  "@context": [
    "https://www.w3.org/ns/credentials/v2",
    "https://w3id.org/security/data-integrity/v2"
  ],
  "type": ["VerifiablePresentation"],
  "verifiableCredential": [...],
  "proof": {
    "type": "DataIntegrityProof",
    "cryptosuite": "ecdsa-rdfc-2019",
    "created": "2024-01-15T09:00:00Z",
    "verificationMethod": "did:key:z6Mk...#key-1",
    "proofPurpose": "authentication",
    "challenge": "n-0S6_WzA2Mj",
    "domain": "https://verifier.example.com",
    "proofValue": "z58DAdFfa9..."
  }
}
```

### Verification Steps

1. Extract proof from VP
2. Canonicalize VP without proof (RDFC-1.0)
3. Hash canonicalized VP
4. Resolve `verificationMethod` to key
5. Verify `proofValue` signature
6. Check `challenge` matches nonce
7. Check `domain` matches verifier

---

## Confirmation Claim (`cnf`)

### In JWT-based Credentials

```json
{
  "iss": "https://issuer.example.com",
  "sub": "user123",
  "cnf": {
    "jwk": {
      "kty": "EC",
      "crv": "P-256",
      "x": "WbbKn2xn0D8N_U9JhK7c3kQJPHqD_gQ7ofWFHpMbQPE",
      "y": "GUph8BgXPFLbOj7N_Y6zq5fzR3dYGkvLJpVVVCKN_Zs"
    }
  }
}
```

### Alternatives to Embedded JWK

| Method | Example | Use Case |
|--------|---------|----------|
| `jwk` | `{"jwk": {...}}` | Self-contained |
| `kid` | `{"kid": "key-1"}` | Reference to known key |
| `jkt` | `{"jkt": "NzbL..."}` | JWK thumbprint |

### In mDoc Credentials

```cbor
MobileSecurityObject = {
  "deviceKeyInfo": {
    "deviceKey": {
      1: 2,       ; kty: EC
      -1: 1,      ; crv: P-256
      -2: h'...',  ; x
      -3: h'...'   ; y
    }
  }
}
```

---

## Nonce Requirements

### OID4VP Specification

| Property | Requirement |
|----------|-------------|
| Format | String |
| Entropy | ≥128 bits |
| Uniqueness | Per request |
| Lifetime | Short (minutes) |

### Location in Response

| Format | Nonce Location |
|--------|----------------|
| JWT VP | `payload.nonce` |
| SD-JWT KB | `kb_jwt.payload.nonce` |
| mDoc | Session transcript |
| Data Integrity | `proof.challenge` |

---

## Standards Reference

| Standard | Section | Topic |
|----------|---------|-------|
| OID4VP | 6.1 | VP Token format |
| SD-JWT | 5 | Key Binding JWT |
| ISO 18013-5 | 9.1.3 | Device Authentication |
| W3C VC DI | 3 | Data Integrity Proofs |
| RFC 7800 | 3 | Confirmation Claim |
