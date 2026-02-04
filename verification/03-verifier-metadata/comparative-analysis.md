# Verifier Metadata — Comparative Analysis

This document compares how EUDI, Procivis, and Affinidi approach verifier identification and trust establishment.

---

## Trust Model Comparison

| Aspect | EUDI | Procivis | Affinidi |
|--------|------|----------|----------|
| **Primary trust anchor** | X.509 PKI | Protocol/DID-based | Platform-mediated |
| **Certificate validation** | Full chain validation | Configurable | N/A (cloud service) |
| **DID support** | Limited | Primary | Supported |
| **Trust registry** | EU member state CAs | Configurable | Affinidi registry |
| **Offline trust check** | Yes (bundled certs) | Partial | No (requires API) |

---

## Client ID Scheme Support

| Scheme | EUDI Android | EUDI iOS | Procivis | Affinidi |
|--------|--------------|----------|----------|----------|
| `redirect_uri` | ✗ | ✗ | ✓ | ✓ |
| `x509_san_dns` | ✓ | ✓ | ✗ | ✗ |
| `x509_san_uri` | ✓ | ✓ | ✗ | ✗ |
| `did` | ✗ | ✗ | ✓ | ✓ |
| `verifier_attestation` | ✗ | ✗ | ✗ | ✗ |
| `pre-registered` | ✗ | ✗ | ✓ | ✓ |

### Analysis

**EUDI**: Focuses on X.509 PKI, aligned with EU regulatory framework and eIDAS. This provides strong legal standing and integrates with existing government PKI.

**Procivis**: Favors DID-based identification, providing flexibility for decentralized ecosystems. Supports both traditional and SSI-native trust models.

**Affinidi**: Uses platform registration model where verifiers are pre-registered with Affinidi services.

---

## Verifier Metadata Handling

### Discovery Mechanism

| Implementation | Inline | By Reference | Pre-registered |
|----------------|--------|--------------|----------------|
| EUDI | ✓ | ✓ | ✗ |
| Procivis | ✓ | ✓ | ✓ |
| Affinidi | ✓ | ✓ | ✓ |

### Metadata Fields Used

| Field | EUDI | Procivis | Affinidi |
|-------|------|----------|----------|
| `client_name` | ✓ (from cert CN) | ✓ | ✓ |
| `logo_uri` | ✗ | ✓ | ✓ |
| `client_uri` | ✗ | ✓ | ✓ |
| `policy_uri` | ✗ | ✗ | ✓ |
| `vp_formats` | ✓ | ✓ | ✓ |

---

## Trust Indicator UI

### EUDI

```
┌────────────────────────────────────┐
│  🏢  [Certificate Common Name]     │
│  ✓ Verified (if chain valid)      │
│  ⚠ Not verified (if chain invalid) │
└────────────────────────────────────┘
```

- Binary trust state (verified/not verified)
- Name extracted from X.509 certificate
- Green checkmark for trusted verifiers

### Procivis

```
┌────────────────────────────────────┐
│  🏢  [Verifier DID or Name]        │
│  Protocol: [OID4VP/ISO_MDL]        │
└────────────────────────────────────┘
```

- Shows protocol used
- DID-based identification
- Less emphasis on trust badges

### Affinidi

```
┌────────────────────────────────────┐
│  🏢  [Registered Verifier Name]    │
│  Powered by Affinidi               │
└────────────────────────────────────┘
```

- Platform-provided metadata
- Affinidi branding

---

## Security Trade-offs

### EUDI: X.509 PKI Model

**Strengths:**
- Strong legal framework (eIDAS qualified certificates)
- Established revocation mechanisms (CRL, OCSP)
- Offline validation possible with bundled roots
- Familiar to enterprise deployments

**Weaknesses:**
- Certificate management overhead
- Centralized trust (dependent on CAs)
- Cost of qualified certificates
- Less flexible for rapid onboarding

### Procivis: DID-Based Model

**Strengths:**
- Decentralized trust
- Self-sovereign verifier identity
- Flexible trust frameworks
- Lower barrier to entry

**Weaknesses:**
- DID resolution dependencies
- Trust framework must be established externally
- Less regulatory clarity
- Revocation mechanisms vary by DID method

### Affinidi: Platform Model

**Strengths:**
- Simple integration
- Managed trust
- Consistent UX

**Weaknesses:**
- Platform dependency
- Centralized trust decisions
- Less transparency
- Vendor lock-in

---

## Regulatory Alignment

| Requirement | EUDI | Procivis | Affinidi |
|-------------|------|----------|----------|
| eIDAS 2.0 compliant | ✓ | Partial | ✗ |
| GDPR considerations | ✓ | ✓ | ✓ |
| Qualified certificates | ✓ | ✗ | ✗ |
| Trust framework agnostic | ✗ | ✓ | ✗ |

---

## Implementation Complexity

| Aspect | EUDI | Procivis | Affinidi |
|--------|------|----------|----------|
| **Trust configuration** | Medium (bundle certs) | Low (config-based) | Low (platform-managed) |
| **Certificate handling** | High (X.509 parsing) | Low | N/A |
| **DID resolution** | N/A | Medium | Low |
| **Custom trust frameworks** | Hard | Easy | N/A |
| **Multi-ecosystem support** | Limited | Good | Limited |

---

## Recommendations

### Choose EUDI-style if:
- Operating in EU regulatory context
- Requiring eIDAS compliance
- Government or enterprise verifiers
- Need offline trust validation
- Legal non-repudiation required

### Choose Procivis-style if:
- Building decentralized ecosystems
- Multiple trust frameworks needed
- Flexibility is priority
- DID infrastructure already exists
- Cross-border interoperability without EU PKI

### Choose Affinidi-style if:
- Rapid development priority
- Limited security expertise
- Platform integration acceptable
- Managed service preferred
- Single ecosystem operation

---

## Future Considerations

1. **EUDI**: Expected expansion to support verifier attestation as EU wallet ecosystem matures
2. **Procivis**: Likely addition of X.509 support for EU compliance
3. **Affinidi**: May add trust framework flexibility as standards evolve
4. **All**: OID4VP verifier attestation scheme adoption expected

---

## Summary Table

| Dimension | EUDI | Procivis | Affinidi |
|-----------|------|----------|----------|
| Trust philosophy | Regulated PKI | Decentralized | Platform |
| Primary identifier | X.509 cert | DID | Platform ID |
| Offline capability | Strong | Partial | Weak |
| Flexibility | Low | High | Low |
| Regulatory fit | EU | Global | Global |
| Setup complexity | Medium | Low | Very Low |
| Trust transparency | High | Medium | Low |
