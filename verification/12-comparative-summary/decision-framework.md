# Decision Framework

A structured approach to choosing the right verification implementation for your use case.

---

## Step 1: Identify Your Primary Use Case

### Government / Regulated Identity

**Requirements:**
- Hardware-backed keys mandatory
- X.509 certificate-based verifier trust
- eIDAS 2.0 compliance path
- Proximity verification (ID checks)

**→ Choose: EUDI**

Rationale: EUDI is designed specifically for European Digital Identity Wallet regulation. Hardware key storage, certificate-based trust, and ISO 18013-5 proximity support are built-in.

---

### Enterprise / B2B Verification

**Requirements:**
- Cross-platform mobile deployment
- Flexible protocol support
- Integration with existing enterprise systems
- Remote signing acceptable

**→ Choose: Procivis ONE**

Rationale: React Native enables efficient cross-platform development. PEX v1/v2 support provides flexibility. RSE model fits enterprise key management practices.

---

### Consumer Web Application

**Requirements:**
- No mobile app development
- Quick integration timeline
- Consent-focused data sharing
- Web-first user experience

**→ Choose: Affinidi**

Rationale: Web SDK and portal-based configuration enable rapid integration. Vault handles credential management and consent. No native development required.

---

## Step 2: Evaluate Technical Constraints

### Do you need offline verification?

```
YES → EUDI or Procivis
       (Both support BLE/NFC for proximity)

NO  → Any implementation works
```

### Do you need hardware-backed keys?

```
YES → EUDI only
       (Device Keystore/Secure Enclave)

NO  → Procivis (RSE) or Affinidi (Cloud)
```

### What credential formats do you need?

```
MSO-MDOC (mDL)     → EUDI or Procivis
SD-JWT-VC          → Any implementation
W3C VC (JSON-LD)   → Procivis or Affinidi
```

### What's your platform strategy?

```
Native Android + iOS    → EUDI
React Native            → Procivis
Web only               → Affinidi
```

---

## Step 3: Consider Team Capabilities

### Native Mobile Expertise (Kotlin/Swift)

**Best fit:** EUDI

- Leverage existing Android/iOS skills
- Full control over native features
- Two codebases to maintain

### React Native Expertise

**Best fit:** Procivis

- Single codebase efficiency
- Familiar tooling
- Bridge overhead acceptable

### Web/Backend Expertise

**Best fit:** Affinidi

- API-based integration
- No mobile development needed
- Cloud service management

---

## Step 4: Assess Security Requirements

### High Security (Government, Financial)

| Requirement | Solution |
|-------------|----------|
| Non-extractable keys | EUDI (hardware keystore) |
| Hardware attestation | EUDI |
| PKI-based trust | EUDI (X.509) |
| Offline capability | EUDI or Procivis |

### Standard Security (Enterprise)

| Requirement | Solution |
|-------------|----------|
| Professional key management | Procivis (RSE) or Affinidi |
| Multi-device access | Procivis or Affinidi |
| Audit logging | All implementations |

### Basic Security (Consumer)

| Requirement | Solution |
|-------------|----------|
| User consent | All implementations |
| TLS transport | All implementations |
| Simple integration | Affinidi |

---

## Step 5: Timeline and Resource Assessment

### Fast Time-to-Market (Weeks)

**→ Affinidi**

- Portal configuration
- Pre-built consent flows
- Minimal custom code

### Moderate Timeline (Months)

**→ Procivis**

- Cross-platform development
- Protocol customization
- RSE integration

### Extended Timeline (6+ Months)

**→ EUDI**

- Native development for both platforms
- Hardware integration
- Full compliance implementation

---

## Step 6: Cost Considerations

### EUDI

| Cost Factor | Impact |
|-------------|--------|
| Development | High (two platforms) |
| Maintenance | High (native updates) |
| Infrastructure | Low (device-based) |
| Licensing | Open source |

### Procivis

| Cost Factor | Impact |
|-------------|--------|
| Development | Medium (cross-platform) |
| Maintenance | Medium |
| Infrastructure | Medium (RSE service) |
| Licensing | Commercial license |

### Affinidi

| Cost Factor | Impact |
|-------------|--------|
| Development | Low |
| Maintenance | Low |
| Infrastructure | Usage-based (SaaS) |
| Licensing | Usage-based |

---

## Decision Tree Summary

```
Start
  │
  ├─ Need EU regulatory compliance?
  │   └─ YES → EUDI
  │
  ├─ Need offline/proximity verification?
  │   └─ YES → EUDI or Procivis
  │
  ├─ Have React Native expertise?
  │   └─ YES → Procivis
  │
  ├─ Web-only application?
  │   └─ YES → Affinidi
  │
  ├─ Need hardware-backed keys?
  │   └─ YES → EUDI
  │
  ├─ Need fastest integration?
  │   └─ YES → Affinidi
  │
  └─ Default for mobile apps
      └─ Procivis (cross-platform efficiency)
```

---

## Hybrid Approaches

Some organizations may benefit from combining implementations:

### EUDI + Affinidi

- EUDI wallet for mobile users
- Affinidi for web-based verification requests
- Shared backend for claim processing

### Procivis + Custom Verifier

- Procivis wallet for holders
- Custom verifier service for specific requirements
- OID4VP interoperability

---

## Migration Considerations

### Starting with Affinidi, moving to native

1. Begin with Affinidi for quick MVP
2. Build native wallet (EUDI-based) in parallel
3. Migrate users with credential portability
4. Phase out cloud dependency

### Starting with Procivis, adding hardware security

1. Deploy with RSE signing
2. Add optional device-key support
3. Let users choose security level
4. Hardware for high-assurance scenarios
