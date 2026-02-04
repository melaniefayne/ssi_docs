# Verifier Validation — Conceptual Overview

After receiving a Verifiable Presentation, the verifier must validate it to establish trust. This document explains what verifiers check and why each validation matters.

---

## The Verification Pipeline

```
┌─────────────────────────────────────────────────────────────────┐
│                    VP Token Received                             │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  1. STRUCTURAL VALIDATION                                        │
│     Parse VP, extract VCs, verify format                         │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  2. CRYPTOGRAPHIC VALIDATION                                     │
│     Verify VP proof (holder binding)                             │
│     Verify VC signatures (issuer authenticity)                   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  3. TRUST VALIDATION                                             │
│     Resolve issuer DID/certificate                               │
│     Check issuer against trust framework                         │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  4. STATUS VALIDATION                                            │
│     Check revocation status                                      │
│     Check expiration                                             │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  5. CONTENT VALIDATION                                           │
│     Verify claims match presentation definition                  │
│     Apply business rules                                         │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ✓ Valid / ✗ Invalid
```

---

## 1. Structural Validation

### What It Checks

- VP Token format is valid (JWT, JSON-LD, CBOR)
- Required fields are present
- Presentation Submission matches definition
- VCs are properly embedded or referenced

### Why It Matters

Malformed presentations cannot be processed. Structural validation is the first gate.

### Example Failures

| Failure | Reason |
|---------|--------|
| Invalid JSON | Cannot parse VP |
| Missing `vp_token` | Response incomplete |
| Wrong format | Expected SD-JWT, got JSON-LD |
| Missing descriptor mapping | Can't correlate VC to request |

---

## 2. Cryptographic Validation

### VP Proof (Holder Binding)

Verifies the presenter controls the credential:

```
VP Proof contains:
  - Signature over VP content + nonce
  - Signed with holder's private key

Verifier checks:
  1. Extract holder's public key (from VC or VP)
  2. Verify signature is valid
  3. Verify nonce matches request nonce
```

### VC Signature (Issuer Authenticity)

Verifies the issuer created the credential:

```
VC Signature:
  - Signed by issuer at issuance time
  - Covers all credential claims

Verifier checks:
  1. Resolve issuer identifier (DID or certificate)
  2. Extract issuer's public key
  3. Verify signature over credential
```

### Format-Specific Validation

| Format | VP Proof | VC Signature |
|--------|----------|--------------|
| JWT VP | JWT signature | Embedded JWT VC signature |
| SD-JWT | Key Binding JWT | SD-JWT issuer signature |
| mDoc | Device signature | MSO issuer signature |
| JSON-LD | Data Integrity proof | VC Data Integrity proof |

---

## 3. Trust Validation

### Issuer Trust

The credential is only as trustworthy as its issuer. Verifiers must determine if they trust the issuer.

```
Issuer Identifier:
  did:web:issuer.government.example

Trust Framework:
  ┌─────────────────────────────────────┐
  │  Trusted Issuers Registry           │
  │                                     │
  │  ✓ did:web:issuer.government.example│
  │  ✓ did:key:z6Mkf...                │
  │  ✗ did:web:unknown-issuer.example  │
  └─────────────────────────────────────┘
```

### Trust Models

| Model | How Trust Is Established |
|-------|-------------------------|
| **Trust List** | Issuer on approved list |
| **X.509 PKI** | Certificate chains to trusted CA |
| **DID Resolution** | DID resolves with known key |
| **Federation** | Issuer attested by trusted party |

### Trust Hierarchy

```
Root of Trust
     │
     ├── Trust Framework Operator
     │        │
     │        ├── Issuer A (trusted)
     │        ├── Issuer B (trusted)
     │        └── Issuer C (trusted)
     │
     └── Unknown issuers (not trusted by default)
```

---

## 4. Status Validation

### Revocation Checking

Credentials may be revoked after issuance:

```
Credential contains:
  credentialStatus: {
    type: "StatusList2021Entry",
    statusPurpose: "revocation",
    statusListIndex: "94567",
    statusListCredential: "https://issuer.example/status/3"
  }

Verifier:
  1. Fetch status list credential
  2. Extract bit at index 94567
  3. If bit = 1, credential is revoked
```

### Status Types

| Status | Meaning | Result |
|--------|---------|--------|
| **Valid** | Credential in good standing | Accept |
| **Revoked** | Permanently invalidated | Reject |
| **Suspended** | Temporarily invalidated | Reject or warn |
| **Unknown** | Status check failed | Policy decision |

### Expiration Checking

```
Credential:
  expirationDate: "2025-01-15T00:00:00Z"

Verifier:
  currentTime = 2024-06-15T12:00:00Z

  if currentTime > expirationDate:
      reject("Credential expired")
```

---

## 5. Content Validation

### Presentation Definition Matching

The VP must satisfy the original request:

```
Request (Presentation Definition):
  input_descriptors: [{
    id: "age_verification",
    constraints: {
      fields: [{
        path: ["$.credentialSubject.over18"],
        filter: { const: true }
      }]
    }
  }]

Response (Presentation Submission):
  descriptor_map: [{
    id: "age_verification",
    path: "$",
    format: "vc+sd-jwt"
  }]

Verifier validates:
  ✓ Descriptor "age_verification" is satisfied
  ✓ Claim "over18" is present and equals true
```

### Business Rules

Beyond protocol validation, verifiers apply business logic:

| Rule | Example |
|------|---------|
| Age requirement | `over18 == true` |
| Geographic restriction | `country in ["DE", "FR", "IT"]` |
| Credential type | `type == "DriversLicense"` |
| Recency | `issuanceDate > (now - 1 year)` |

---

## Validation Result

### Success

All checks pass:

```json
{
  "valid": true,
  "verified_claims": {
    "given_name": "John",
    "family_name": "Doe",
    "over18": true
  },
  "issuer": "did:web:issuer.government.example",
  "issuer_trusted": true
}
```

### Failure

Any check fails:

```json
{
  "valid": false,
  "error": "revocation_check_failed",
  "error_description": "Credential has been revoked",
  "details": {
    "credential_id": "urn:uuid:123...",
    "revocation_date": "2024-03-15T00:00:00Z"
  }
}
```

---

## Validation Order

The order of validation affects efficiency and security:

```
1. Structural (fast, cheap)
   └── Fail fast on malformed input

2. Nonce (fast, important)
   └── Detect replay attacks early

3. Cryptographic (slower, essential)
   └── Signatures require computation

4. Trust (may require network)
   └── DID resolution, trust list fetch

5. Status (requires network)
   └── Revocation list fetch

6. Content (application-specific)
   └── Business rules last
```

---

## Security Considerations

### Replay Attacks

Without nonce validation, attackers could reuse captured VPs:

```
Attack:
  Attacker captures VP from legitimate presentation
  Attacker replays VP to same or different verifier

Defense:
  Nonce is unique per request
  Nonce is bound to VP proof
  Verifier rejects non-matching nonce
```

### Substitution Attacks

Without proper binding, attackers could swap credentials:

```
Attack:
  Attacker receives legitimate credential
  Attacker tries to use it without holder's key

Defense:
  Key binding in credential
  VP proof requires holder's private key
```

### Issuer Impersonation

Without trust validation, fake issuers could be accepted:

```
Attack:
  Attacker creates fake issuer
  Attacker issues credentials
  Attacker presents to verifier

Defense:
  Trust framework verification
  Issuer must be in trusted list
```
