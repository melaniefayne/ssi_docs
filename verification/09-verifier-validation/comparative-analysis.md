# Verifier Validation — Comparative Analysis

This document compares validation approaches across different ecosystems, focusing on trust models, validation depth, and implementation patterns.

---

## Validation Scope

| Aspect | EUDI Ecosystem | Procivis | Affinidi |
|--------|----------------|----------|----------|
| **Validation location** | Backend + SDK | Core library | Cloud service |
| **Trust framework** | EU PKI (eIDAS) | Configurable | Platform registry |
| **Revocation method** | StatusList2021 | Multiple | StatusList |
| **DID resolution** | Limited | Multiple methods | did:web, did:key |

---

## Trust Model Comparison

### EUDI

```
┌─────────────────────────────────────────────────────────────┐
│  EU Trust Framework                                          │
│                                                              │
│  EUDI Wallet Root CA                                         │
│       │                                                      │
│       ├── Member State CA (DE)                               │
│       │       └── Issuer Certificate                         │
│       │                                                      │
│       ├── Member State CA (FR)                               │
│       │       └── Issuer Certificate                         │
│       │                                                      │
│       └── ... (each member state)                            │
│                                                              │
│  Verifier validates:                                         │
│  ✓ Certificate chain to EUDI root                            │
│  ✓ Certificate not revoked (CRL/OCSP)                        │
│  ✓ Certificate has correct key usage                         │
└─────────────────────────────────────────────────────────────┘
```

### Procivis

```
┌─────────────────────────────────────────────────────────────┐
│  Configurable Trust                                          │
│                                                              │
│  Option 1: DID-based                                         │
│    did:web:issuer.example → DID Document → Public Key        │
│    Trust: Explicit list or federation                        │
│                                                              │
│  Option 2: Certificate-based                                 │
│    X.509 certificate → Chain validation                      │
│    Trust: Configured root CAs                                │
│                                                              │
│  Option 3: Hybrid                                            │
│    Combine DID + certificate attestation                     │
└─────────────────────────────────────────────────────────────┘
```

### Affinidi

```
┌─────────────────────────────────────────────────────────────┐
│  Platform-Managed Trust                                      │
│                                                              │
│  Affinidi Trust Registry                                     │
│       │                                                      │
│       ├── Registered Issuers                                 │
│       │       └── Issuer A (verified)                        │
│       │       └── Issuer B (verified)                        │
│       │                                                      │
│       └── Verification Rules                                 │
│               └── Which issuers for which credential types   │
│                                                              │
│  Verifier receives pre-validated result                      │
└─────────────────────────────────────────────────────────────┘
```

---

## Signature Verification

| Format | EUDI | Procivis | Affinidi |
|--------|------|----------|----------|
| **JWT** | ES256 (required) | ES256, EdDSA | ES256, EdDSA |
| **SD-JWT** | ES256 | ES256, EdDSA | ES256 |
| **mDoc** | ES256, ES384, ES512 | ES256 | N/A |
| **JSON-LD** | Limited | Multiple suites | ecdsa-secp256k1 |

---

## Revocation Checking

### Methods Supported

| Method | EUDI | Procivis | Affinidi |
|--------|------|----------|----------|
| **StatusList2021** | ✓ | ✓ | ✓ |
| **BitstringStatusList** | Planned | ✓ | ✓ |
| **OCSP** | ✓ (certs) | ✗ | ✗ |
| **CRL** | ✓ (certs) | ✗ | ✗ |

### Caching Behavior

| Aspect | EUDI | Procivis | Affinidi |
|--------|------|----------|----------|
| **Cache duration** | Per CRL/OCSP | Configurable | Platform-managed |
| **Offline mode** | Cached CRLs | No revocation | No revocation |
| **Real-time check** | OCSP optional | Always fetch | Platform handles |

---

## Validation Depth

### EUDI: Deep Validation

```
✓ VP structure
✓ VP signature (holder)
✓ Nonce binding
✓ Audience binding
✓ VC structure
✓ VC signature (issuer)
✓ Issuer certificate chain
✓ Certificate revocation
✓ Credential expiration
✓ Credential revocation (StatusList)
✓ Trust framework membership
✓ Presentation definition match
```

### Procivis: Configurable Validation

```
✓ VP structure
✓ VP proof (format-dependent)
✓ Nonce binding
✓ VC signature
? Issuer trust (configurable)
? Revocation (if enabled)
✓ Expiration
✓ Presentation definition match
```

### Affinidi: Platform Validation

```
✓ VP structure
✓ Signatures (all)
✓ Nonce binding
✓ Issuer trust (platform registry)
✓ Revocation status
✓ Expiration
✓ Schema validation
✓ Business rules (configurable)
```

---

## Error Handling

### Error Granularity

| Aspect | EUDI | Procivis | Affinidi |
|--------|------|----------|----------|
| **Error detail** | High | Medium | Low |
| **Debugging info** | Certificate details | Error codes | Generic messages |
| **Recovery hints** | Yes | Some | Platform-specific |

### Sample Errors

**EUDI:**
```json
{
  "error": "certificate_validation_failed",
  "details": {
    "certificate_subject": "CN=Issuer Example",
    "reason": "certificate_revoked",
    "revocation_date": "2024-03-15T00:00:00Z",
    "crl_url": "https://ca.example/crl"
  }
}
```

**Procivis:**
```json
{
  "error": "SIGNATURE_VERIFICATION_FAILED",
  "credential_index": 0
}
```

**Affinidi:**
```json
{
  "success": false,
  "error": "Verification failed. Please try again."
}
```

---

## Performance Characteristics

| Aspect | EUDI | Procivis | Affinidi |
|--------|------|----------|----------|
| **Local validation** | Fast (after cert cache) | Fast | N/A |
| **Network calls** | DID resolution, revocation | DID resolution | API call |
| **Caching** | Extensive | Configurable | Platform |
| **Offline capable** | Partial | Partial | No |

---

## Security Trade-offs

### EUDI

**Strengths:**
- Strong PKI foundation
- Legal non-repudiation
- Established revocation
- Deep validation

**Weaknesses:**
- Certificate management burden
- PKI dependencies
- Limited DID support

### Procivis

**Strengths:**
- Flexible trust models
- DID-native
- Multiple formats
- Configurable depth

**Weaknesses:**
- Trust framework setup required
- Variable security levels
- Complexity in configuration

### Affinidi

**Strengths:**
- Simple integration
- Consistent validation
- Managed security

**Weaknesses:**
- Platform dependency
- Less transparency
- Limited customization
- No offline mode

---

## Compliance Considerations

| Requirement | EUDI | Procivis | Affinidi |
|-------------|------|----------|----------|
| **eIDAS 2.0** | Full compliance | Partial | Not targeted |
| **GDPR logging** | Built-in | Configurable | Platform handles |
| **Audit trail** | Comprehensive | Configurable | Platform provides |
| **Data minimization** | Enforced | Configurable | Platform policy |

---

## Recommendations

### Choose EUDI validation if:
- Operating in EU regulatory context
- eIDAS compliance required
- X.509 PKI available
- Strong legal requirements
- Government/enterprise verifiers

### Choose Procivis validation if:
- Building flexible ecosystems
- DID-based trust preferred
- Multiple credential formats
- Custom trust frameworks
- Self-hosted verification

### Choose Affinidi validation if:
- Rapid deployment needed
- Managed service acceptable
- Standard use cases
- Limited security expertise
- Platform integration OK

---

## Future Trends

| Trend | EUDI | Procivis | Affinidi |
|-------|------|----------|----------|
| **Zero-knowledge proofs** | Research | Possible | Possible |
| **Selective disclosure** | SD-JWT, mDoc | Multiple | SD-JWT |
| **Cross-border trust** | EU focus | Global | Global |
| **Decentralized trust** | Limited | Yes | Platform |
