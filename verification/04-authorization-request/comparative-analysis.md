# Authorization Request — Comparative Analysis

This document compares how EUDI, Procivis, and Affinidi handle OID4VP authorization requests.

---

## Request Parsing Approach

| Aspect | EUDI | Procivis | Affinidi |
|--------|------|----------|----------|
| **Primary mechanism** | Native SDK | Core library | Cloud service |
| **URI scheme handling** | Multiple schemes | Universal links + custom | Standard OID4VP |
| **Request URI fetching** | SDK automatic | Manual + core | Service handles |
| **JWT validation** | SDK with X.509 | Core library | Service validates |

---

## Supported Features

### URI Schemes

| Scheme | EUDI Android | EUDI iOS | Procivis | Affinidi |
|--------|--------------|----------|----------|----------|
| `openid4vp://` | ✓ | ✓ | ✓ | ✓ |
| `eudi-openid4vp://` | ✓ | ✗ | ✗ | ✗ |
| `mdoc-openid4vp://` | ✓ | ✗ | ✗ | ✗ |
| `haip-openid4vp://` | ✓ | ✓ | ✗ | ✗ |
| Universal links | ✗ | ✓ | ✓ | ✓ |

### Response Modes

| Mode | EUDI | Procivis | Affinidi |
|------|------|----------|----------|
| `direct_post` | ✓ | ✓ | ✓ |
| `direct_post.jwt` | ✓ | ✗ | ✓ |
| `fragment` | ✗ | ✗ | ✓ |

### Client ID Schemes

| Scheme | EUDI | Procivis | Affinidi |
|--------|------|----------|----------|
| `redirect_uri` | ✗ | ✓ | ✓ |
| `x509_san_dns` | ✓ | ✗ | ✗ |
| `x509_san_uri` | ✓ | ✗ | ✗ |
| `did` | ✗ | ✓ | ✓ |
| `verifier_attestation` | ✗ | ✗ | ✗ |
| `pre-registered` | ✗ | ✓ | ✓ |

---

## Presentation Definition Support

| Feature | EUDI | Procivis | Affinidi |
|---------|------|----------|----------|
| **PEX v1.0** | ✓ | ✓ | ✓ |
| **PEX v2.0** | Partial | ✓ | ✓ |
| **DCQL** | ✗ | ✗ | ✗ |
| **By reference** | ✓ | ✓ | ✓ |
| **Inline** | ✓ | ✓ | ✓ |

---

## Request Validation Depth

### EUDI

```
┌─────────────────────────────────────────────────────────────┐
│  Validation Steps                                            │
├─────────────────────────────────────────────────────────────┤
│  1. ✓ URI scheme validation                                  │
│  2. ✓ JWT structure validation                               │
│  3. ✓ JWT signature validation                               │
│  4. ✓ X.509 certificate chain validation                     │
│  5. ✓ Certificate SAN matching                               │
│  6. ✓ Timing validation (iat/exp)                            │
│  7. ✓ Required parameter presence                            │
│  8. ✓ Presentation definition structure                      │
└─────────────────────────────────────────────────────────────┘
```

### Procivis

```
┌─────────────────────────────────────────────────────────────┐
│  Validation Steps                                            │
├─────────────────────────────────────────────────────────────┤
│  1. ✓ URL parsing                                            │
│  2. ✓ HTTP redirect following                                │
│  3. ✓ JWT validation (if present)                            │
│  4. ✓ DID resolution (if did scheme)                         │
│  5. ✓ Presentation definition parsing                        │
│  6. ✓ Credential format matching                             │
└─────────────────────────────────────────────────────────────┘
```

### Affinidi

```
┌─────────────────────────────────────────────────────────────┐
│  Validation Steps                                            │
├─────────────────────────────────────────────────────────────┤
│  1. ✓ Iota framework request validation                      │
│  2. ✓ Platform-level client validation                       │
│  3. ✓ Presentation definition parsing                        │
│  4. ✓ Format compatibility check                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Security Model Comparison

| Security Aspect | EUDI | Procivis | Affinidi |
|-----------------|------|----------|----------|
| **Request signing required** | Yes (for trust) | Optional | Optional |
| **Certificate validation** | Deep PKI | N/A | N/A |
| **DID resolution** | N/A | Yes | Yes |
| **Nonce validation** | SDK enforced | Core enforced | Platform |
| **Expiration enforcement** | Yes | Yes | Yes |
| **Replay protection** | Nonce-based | Nonce-based | Platform |

---

## Error Handling Approaches

### EUDI

- **Pattern**: State machine events
- **User feedback**: Error displayed in UI
- **Recovery**: Return to previous screen

```kotlin
TransferEventPartialState.Error(error: String)
```

### Procivis

- **Pattern**: Try-catch with exception reporting
- **User feedback**: Error screen or toast
- **Recovery**: Navigate back, allow retry

```typescript
try {
    await invitationHandler(url);
} catch (e) {
    reportException(e, 'context');
}
```

### Affinidi

- **Pattern**: Service response codes
- **User feedback**: Platform-provided messages
- **Recovery**: Retry or contact support

---

## Trade-off Analysis

### EUDI Approach

**Strengths:**
- Strong PKI-based trust
- EU regulatory compliance
- Comprehensive validation
- Offline-capable trust (bundled CAs)

**Weaknesses:**
- Limited to X.509 schemes
- Less flexible for non-EU ecosystems
- Certificate management overhead

### Procivis Approach

**Strengths:**
- Flexible client identification
- Multi-protocol support (V1, V2)
- DID-native ecosystem
- Configurable trust models

**Weaknesses:**
- Less prescriptive trust model
- Requires trust framework setup
- DID resolution dependencies

### Affinidi Approach

**Strengths:**
- Simplified integration
- Managed validation
- Consistent experience

**Weaknesses:**
- Platform dependency
- Less transparency
- Limited customization

---

## Implementation Complexity

| Aspect | EUDI | Procivis | Affinidi |
|--------|------|----------|----------|
| **Request parsing** | SDK abstracted | Manual + Core | Platform |
| **Trust validation** | Complex (PKI) | Medium (DID) | Simple |
| **Error handling** | State events | Try-catch | Callbacks |
| **Custom schemes** | Multiple registered | Universal links | Standard |
| **Testing** | Requires certs | Configurable | Platform tools |

---

## Recommendations

### Choose EUDI-style if:
- Operating in EU/eIDAS context
- X.509 infrastructure available
- High-assurance requirements
- Government/enterprise verifiers

### Choose Procivis-style if:
- Building decentralized systems
- DID-based identity preferred
- Multi-ecosystem operation
- Flexibility is priority

### Choose Affinidi-style if:
- Rapid development needed
- Managed service acceptable
- Standard OID4VP sufficient
- Limited security expertise

---

## Future Evolution

| Evolution | EUDI | Procivis | Affinidi |
|-----------|------|----------|----------|
| **DCQL support** | Planned | Possible | Possible |
| **Verifier attestation** | Expected | Possible | Possible |
| **Additional schemes** | Likely | Likely | Likely |
| **Response encryption** | Yes | TBD | Yes |
