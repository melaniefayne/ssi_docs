# Selective Disclosure Cryptography

This document details the cryptographic mechanisms enabling selective disclosure in credential presentations.

---

## The Selective Disclosure Problem

Traditional credentials reveal everything:

```
Full Credential:
{
  "name": "John Doe",
  "dateOfBirth": "1990-05-15",
  "address": "123 Main St",
  "socialSecurityNumber": "123-45-6789",
  "over18": true
}

Verifier only needs: over18

But with basic JWT:
- Verifier receives ALL claims
- Cannot prove partial disclosure
- Privacy violation
```

Selective disclosure solves this:

```
With SD-JWT:
- Holder chooses which claims to reveal
- Cryptographic proof that revealed claims are authentic
- Unrevealed claims remain hidden (not even hashes exposed in presentation)
```

---

## SD-JWT Selective Disclosure

### Issuance Structure

```
Issuer creates:
{
  "_sd": [
    "hash_of_disclosure_1",
    "hash_of_disclosure_2",
    "hash_of_disclosure_3"
  ],
  "_sd_alg": "sha-256",
  "iss": "https://issuer.example.com",
  "cnf": { "jwk": { ... holder key ... } }
}

With disclosures:
~WyJzYWx0MSIsICJuYW1lIiwgIkpvaG4gRG9lIl0
~WyJzYWx0MiIsICJkYXRlT2ZCaXJ0aCIsICIxOTkwLTA1LTE1Il0
~WyJzYWx0MyIsICJvdmVyMTgiLCB0cnVlXQ
```

### Disclosure Structure

```
Disclosure = base64url(["salt", "claim_name", "claim_value"])

Example:
["abc123salt", "over18", true]
→ base64url → "WyJhYmMxMjNzYWx0IiwgIm92ZXIxOCIsIHRydWVd"
```

### Hash Computation

```
disclosure = "WyJhYmMxMjNzYWx0IiwgIm92ZXIxOCIsIHRydWVd"
hash = base64url(SHA-256(ASCII(disclosure)))
     = "fUMPJzLqf..."
```

### Presentation with Selective Disclosure

```
Full SD-JWT (at issuance):
eyJ...issuer-jwt...~disclosure1~disclosure2~disclosure3

Selective Presentation (only over18):
eyJ...issuer-jwt...~disclosure3~eyJ...kb-jwt...

Holder omits disclosure1 and disclosure2
```

### Verification Algorithm

```python
def verify_sd_jwt_disclosure(sd_jwt_with_kb):
    # Split components
    parts = sd_jwt_with_kb.split('~')
    issuer_jwt = parts[0]
    disclosures = parts[1:-1]
    kb_jwt = parts[-1]

    # Verify issuer JWT
    issuer_payload = verify_jwt(issuer_jwt)

    # Verify each disclosure
    disclosed_claims = {}
    for disclosure in disclosures:
        # Compute hash
        hash = base64url(sha256(disclosure))

        # Check hash in _sd array
        if hash not in issuer_payload['_sd']:
            raise Error("Invalid disclosure")

        # Decode disclosure
        salt, name, value = json.loads(base64url_decode(disclosure))
        disclosed_claims[name] = value

    # Verify key binding JWT
    verify_kb_jwt(kb_jwt, issuer_payload['cnf'], sd_jwt_without_kb)

    return disclosed_claims
```

---

## mDoc Selective Disclosure

### IssuerAuth Structure

```cbor
IssuerAuth = COSE_Sign1 {
  protected: { alg: ES256 },
  payload: MobileSecurityObject
}

MobileSecurityObject = {
  "version": "1.0",
  "digestAlgorithm": "SHA-256",
  "valueDigests": {
    "org.iso.18013.5.1": {
      0: h'a1b2c3...',  // SHA-256 of element 0
      1: h'd4e5f6...',  // SHA-256 of element 1
      2: h'g7h8i9...'   // SHA-256 of element 2
    }
  },
  "deviceKeyInfo": { ... },
  "validityInfo": { ... }
}
```

### Element Encoding

```cbor
IssuerSignedItem = {
  "digestID": 0,
  "random": h'random_bytes',
  "elementIdentifier": "family_name",
  "elementValue": "Doe"
}

Digest = SHA-256(CBOR(IssuerSignedItem))
```

### Selective Disclosure in Presentation

```
DeviceResponse = {
  documents: [{
    docType: "org.iso.18013.5.1.mDL",
    issuerSigned: {
      nameSpaces: {
        "org.iso.18013.5.1": [
          // Only include selected elements
          IssuerSignedItem_for_element_2  // e.g., over18
          // element 0 and 1 NOT included
        ]
      },
      issuerAuth: COSE_Sign1  // Full MSO with all digests
    },
    deviceSigned: {
      deviceAuth: ...
    }
  }]
}
```

### Verification

```python
def verify_mdoc_disclosure(device_response):
    for doc in device_response.documents:
        # Verify issuer signature on MSO
        mso = verify_cose(doc.issuerSigned.issuerAuth)

        # For each disclosed element
        for item in doc.issuerSigned.nameSpaces[namespace]:
            # Compute expected digest
            computed = sha256(cbor.encode(item))

            # Compare with MSO
            expected = mso.valueDigests[namespace][item.digestID]

            if computed != expected:
                raise Error("Digest mismatch")

        # Verify device authentication
        verify_device_auth(doc.deviceSigned, mso.deviceKeyInfo)
```

---

## BBS+ Signatures (Advanced)

BBS+ enables zero-knowledge selective disclosure:

### Properties

| Property | Value |
|----------|-------|
| Unlinkability | Presentations cannot be correlated |
| Zero-knowledge | Prove statements without revealing values |
| Multi-message | Sign multiple messages, disclose subset |

### Signature Generation (Issuer)

```
Input: messages m1, m2, ..., mn
       secret key sk
       public key pk

BBS_Sign(sk, messages) → signature σ
```

### Proof Generation (Holder)

```
Input: signature σ
       messages m1, m2, ..., mn
       disclosed indices D ⊂ {1..n}

BBS_ProofGen(σ, messages, D) → proof π

Proof reveals only messages at indices in D
Proof is unlinkable to signature
```

### Verification (Verifier)

```
Input: proof π
       disclosed messages
       public key pk

BBS_ProofVerify(pk, proof, disclosed) → bool
```

### Current Support

| Implementation | BBS+ Support |
|----------------|--------------|
| EUDI | Not yet |
| Procivis | Partial |
| Affinidi | Not yet |

---

## Comparison of Mechanisms

| Aspect | SD-JWT | mDoc | BBS+ |
|--------|--------|------|------|
| **Correlation** | Linkable | Linkable | Unlinkable |
| **Proof size** | Medium | Large | Constant |
| **Computation** | Fast | Fast | Slower |
| **Standards maturity** | High | High | Medium |
| **Crypto complexity** | Low | Medium | High |
| **Zero-knowledge** | No | No | Yes |

---

## Privacy Trade-offs

### SD-JWT

```
Privacy: Good (only selected claims revealed)
Risk: Same presentation can be correlated by issuer signature
Mitigation: Multiple credentials for different contexts
```

### mDoc

```
Privacy: Good (only selected elements revealed)
Risk: MSO contains all digests (verifier learns cardinality)
Risk: Device key enables correlation
Mitigation: Per-session device keys (not yet standard)
```

### BBS+

```
Privacy: Excellent (unlinkable presentations)
Risk: Implementation complexity may lead to bugs
Risk: Newer cryptography, less battle-tested
Mitigation: Use audited implementations
```

---

## Implementation Guidance

### Minimizing Disclosure

```
DO:
- Only include disclosures for required claims
- Remove optional claim disclosures
- Consider using derived credentials

DON'T:
- Include all disclosures "just in case"
- Share unique identifiers when not needed
- Use same credential across unrelated verifiers
```

### Credential Design for Disclosure

```
Good:
{
  "over18": true,           // Derived claim
  "ageRange": "18-25",      // Range instead of exact
  "countryOfResidence": "DE" // Needed claim
}

Bad:
{
  "dateOfBirth": "1990-05-15",  // Exact date when not needed
  "address": "123 Main St",     // Full address when not needed
  "nationalId": "123456789"     // Identifier when not needed
}
```
