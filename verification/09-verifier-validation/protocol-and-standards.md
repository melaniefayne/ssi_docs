# Verifier Validation — Protocol and Standards

This document details the validation requirements and algorithms as specified in OID4VP, W3C VC Data Model, and related standards.

---

## VP Token Validation

### JWT VP Validation

```
1. Decode JWT without verifying signature
2. Extract header and payload
3. Validate JWT structure:
   - header.alg is supported
   - header.typ is "JWT" or absent
   - payload.iss is present (holder)
   - payload.aud matches verifier
   - payload.nonce matches request nonce
   - payload.vp is present
4. Resolve holder key (from iss or embedded)
5. Verify JWT signature
6. Extract embedded VCs from payload.vp.verifiableCredential
```

### SD-JWT VP Validation

```
1. Split SD-JWT+KB: issuer-jwt~disclosures~kb-jwt
2. Validate issuer-jwt:
   - Decode and verify signature
   - Issuer key from iss claim
3. Process disclosures:
   - Decode each disclosure
   - Verify hash matches _sd array entry
   - Reconstruct disclosed claims
4. Validate key binding JWT (kb-jwt):
   - header.typ == "kb+jwt"
   - payload.nonce == request nonce
   - payload.aud == verifier identifier
   - payload.iat is recent
   - Signature verifies with credential's cnf key
```

### mDoc (ISO 18013-5) Validation

```
1. Decode CBOR DeviceResponse
2. For each document:
   a. Validate IssuerAuth:
      - Decode COSE_Sign1 structure
      - Verify MSO signature with issuer certificate
      - Validate certificate chain
   b. Validate DeviceAuth:
      - Decode COSE_Sign1 or COSE_Mac0
      - Verify using deviceKey from MSO
      - Verify SessionTranscript binding
   c. Validate disclosed elements:
      - Verify element digests match MSO
```

---

## VC Signature Validation

### JWT VC

```
JWT VC:
  eyJhbGciOiJFUzI1NiIsInR5cCI6InZjK3NkLWp3dCJ9.
  eyJpc3MiOiJkaWQ6d2ViOmlzc3Vlci5leGFtcGxlIiwuLi59.
  signature

Validation:
1. Decode JWT
2. Extract issuer from iss claim
3. Resolve issuer to public key:
   - DID → DID Document → verification method
   - URL → JWKS → matching kid
   - X.509 → certificate chain validation
4. Verify signature with issuer's public key
5. Check iat (issued at) is in past
6. Check exp (expiration) is in future or absent
```

### SD-JWT VC

```
SD-JWT VC:
  issuer-jwt~disclosure1~disclosure2~...

Validation:
1. Decode issuer-jwt as JWT
2. Verify signature with issuer's key
3. For each disclosure:
   - base64url decode: ["salt", "claim_name", "claim_value"]
   - compute hash: SHA-256(disclosure)
   - verify hash in _sd array
4. Reconstruct selective disclosed claims
5. Verify cnf (confirmation) contains holder's public key
```

### Data Integrity Proofs

```json
{
  "@context": ["https://www.w3.org/ns/credentials/v2"],
  "type": ["VerifiableCredential"],
  "issuer": "did:web:issuer.example",
  "credentialSubject": { ... },
  "proof": {
    "type": "DataIntegrityProof",
    "cryptosuite": "ecdsa-rdfc-2019",
    "verificationMethod": "did:web:issuer.example#key-1",
    "proofPurpose": "assertionMethod",
    "proofValue": "z58DAdFfa9..."
  }
}
```

Validation:
1. JSON-LD canonicalization
2. Hash canonical form
3. Verify signature in proofValue
4. Resolve verificationMethod to key

---

## Nonce Validation

### Requirements

| Aspect | Requirement |
|--------|-------------|
| Uniqueness | Nonce must be unique per request |
| Entropy | Minimum 128 bits of randomness |
| Storage | Verifier must track issued nonces |
| Expiration | Nonce should have time limit |
| Single-use | Must be consumed after validation |

### Validation Algorithm

```python
def validate_nonce(vp_nonce, expected_nonce, nonce_store):
    # Check match
    if vp_nonce != expected_nonce:
        return Error("nonce_mismatch")

    # Check not already used
    if nonce_store.is_consumed(expected_nonce):
        return Error("nonce_replay")

    # Check not expired
    if nonce_store.is_expired(expected_nonce):
        return Error("nonce_expired")

    # Mark as consumed
    nonce_store.consume(expected_nonce)

    return Success()
```

---

## DID Resolution

### Resolution Process

```
DID: did:web:issuer.example.com:department

Step 1: Transform to URL
  → https://issuer.example.com/department/did.json

Step 2: Fetch DID Document
  GET https://issuer.example.com/department/did.json

Step 3: Parse DID Document
  {
    "id": "did:web:issuer.example.com:department",
    "verificationMethod": [{
      "id": "#key-1",
      "type": "JsonWebKey2020",
      "publicKeyJwk": { ... }
    }],
    "assertionMethod": ["#key-1"]
  }

Step 4: Extract key for verification
  Match kid from JWT header to verificationMethod id
```

### Supported DID Methods

| Method | Resolution | Trust Model |
|--------|------------|-------------|
| `did:web` | HTTPS fetch | DNS + TLS |
| `did:key` | Decode from identifier | Self-certifying |
| `did:jwk` | Decode JWK | Self-certifying |
| `did:ion` | Bitcoin anchored | Blockchain |
| `did:ebsi` | EBSI registry | EU trust framework |

---

## Revocation Validation

### StatusList2021

```
Credential:
{
  "credentialStatus": {
    "id": "https://issuer.example/status/3#94567",
    "type": "StatusList2021Entry",
    "statusPurpose": "revocation",
    "statusListIndex": "94567",
    "statusListCredential": "https://issuer.example/status/3"
  }
}

Validation:
1. Fetch status list credential
2. Verify status list VC signature
3. Decode status list (GZIP + base64)
4. Extract bit at index 94567
5. If bit == 1, credential is revoked
```

### Bitstring Status List

Updated mechanism in VC Data Model 2.0:

```
{
  "credentialStatus": {
    "id": "https://issuer.example/status/3#94567",
    "type": "BitstringStatusListEntry",
    "statusPurpose": "revocation",
    "statusListIndex": "94567",
    "statusListCredential": "https://issuer.example/status/3"
  }
}
```

### Status Purposes

| Purpose | Meaning |
|---------|---------|
| `revocation` | Permanent invalidation |
| `suspension` | Temporary invalidation |
| `message` | Issuer communication |

---

## Trust Framework Validation

### EUDI Trust Framework

```
1. Extract issuer identifier
2. Resolve to certificate (X.509)
3. Validate certificate chain:
   - Chain to EUDI Wallet Root CA
   - Extended Key Usage includes Credential Issuer
   - Certificate not expired
   - Certificate not revoked (CRL/OCSP)
4. Check issuer in EU Trust List
```

### Generic Trust List

```python
def validate_issuer_trust(issuer_id, trust_list):
    if issuer_id in trust_list.trusted_issuers:
        entry = trust_list.get(issuer_id)
        if entry.valid_until > now():
            return TrustResult(
                trusted=True,
                trust_level=entry.trust_level,
                capabilities=entry.capabilities
            )
    return TrustResult(trusted=False)
```

---

## Presentation Submission Validation

### Matching Algorithm

```python
def validate_submission(
    presentation_definition,
    presentation_submission,
    vp_token
):
    # Check definition ID matches
    if submission.definition_id != definition.id:
        return Error("definition_mismatch")

    # Validate each descriptor mapping
    for mapping in submission.descriptor_map:
        # Find corresponding input descriptor
        descriptor = definition.get_descriptor(mapping.id)
        if not descriptor:
            return Error("unknown_descriptor")

        # Extract credential using path
        credential = jsonpath(vp_token, mapping.path)

        # Validate credential satisfies descriptor
        if not satisfies(credential, descriptor):
            return Error("constraint_not_satisfied")

    return Success()
```

### Constraint Satisfaction

```python
def satisfies(credential, descriptor):
    for field in descriptor.constraints.fields:
        # Evaluate each path
        matched = False
        for path in field.path:
            values = jsonpath(credential, path)
            if values and filter_matches(values, field.filter):
                matched = True
                break

        if not matched and not field.optional:
            return False

    return True
```

---

## Error Responses

### OID4VP Error Codes

| Error | Description |
|-------|-------------|
| `invalid_request` | Malformed request |
| `invalid_token` | VP Token validation failed |
| `insufficient_scope` | Missing required claims |
| `access_denied` | Trust or policy check failed |

### Detailed Error Response

```json
{
  "error": "invalid_token",
  "error_description": "Credential signature verification failed",
  "error_details": {
    "credential_index": 0,
    "check_failed": "signature_verification",
    "issuer": "did:web:issuer.example"
  }
}
```

---

## Standards Reference

| Standard | Section | Topic |
|----------|---------|-------|
| OID4VP | 7 | VP Token Validation |
| W3C VC DM 1.1 | 4.8 | Proofs |
| W3C VC DM 2.0 | 5.5 | Verification |
| SD-JWT | 7 | Verification |
| ISO 18013-5 | 9 | Verification |
| DIF PEX | 8 | Submission Processing |
| RFC 8725 | - | JWT Best Practices |
