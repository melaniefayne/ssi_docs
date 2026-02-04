# Comparative Architecture Analysis

This document provides a side-by-side comparison of the verification architectures across EUDI, Procivis ONE, and Affinidi implementations, highlighting key design decisions and their implications.

---

## 1. High-Level Comparison

| Aspect | EUDI | Procivis ONE | Affinidi |
|--------|------|--------------|----------|
| **Platform** | Native (Android/iOS) | React Native | Cloud + Web SDK |
| **Wallet Model** | Device-local | Device-local | Cloud-based Vault |
| **SDK Language** | Kotlin/Swift | Rust (via RN bridge) | TypeScript |
| **Protocol Engine** | EudiWalletKit | Procivis Core | Iota Framework |
| **Key Storage** | Hardware (TEE/SE) | RSE (Remote) | Cloud HSM |
| **Transport Modes** | HTTP, BLE, NFC | HTTP, BLE, NFC | HTTP only |

---

## 2. Module Structure Comparison

### EUDI (Native)

```
┌─────────────────────────────────┐
│ presentation-feature (UI)       │  Platform-specific
├─────────────────────────────────┤
│ business-logic (Interactors)    │  Platform-specific
├─────────────────────────────────┤
│ core-logic (Controllers)        │  Platform-specific
├─────────────────────────────────┤
│ wallet-core SDK                 │  Shared library
└─────────────────────────────────┘
```

**Characteristics:**
- Strict layer separation
- Platform-idiomatic code
- SDK abstracts protocol complexity
- Parallel implementations for Android/iOS

### Procivis ONE (React Native)

```
┌─────────────────────────────────┐
│ React Screens & Components      │  Shared JS/TS
├─────────────────────────────────┤
│ Utils & Business Logic          │  Shared JS/TS
├─────────────────────────────────┤
│ React Native Bridge             │  Platform bridges
├─────────────────────────────────┤
│ Rust Core Library               │  Shared native
└─────────────────────────────────┘
```

**Characteristics:**
- Shared codebase across platforms
- Native performance for crypto
- Bridge overhead for JS ↔ Native
- Single implementation to maintain

### Affinidi (Cloud)

```
┌─────────────────────────────────┐
│ Application Code                │  Customer-owned
├─────────────────────────────────┤
│ Affinidi TDK                    │  SDK (npm)
├─────────────────────────────────┤
│ Affinidi Cloud Services         │  SaaS
├─────────────────────────────────┤
│ Affinidi Vault                  │  SaaS
└─────────────────────────────────┘
```

**Characteristics:**
- Minimal client-side code
- SaaS dependency
- No native integration required
- Serverless-friendly

---

## 3. Key Storage & Signing

| Implementation | Key Location | Signing Mechanism | Hardware Protection |
|----------------|--------------|-------------------|---------------------|
| **EUDI Android** | Android Keystore | Local (TEE) | StrongBox when available |
| **EUDI iOS** | Secure Enclave | Local (SE) | Always hardware-backed |
| **Procivis** | Remote Secure Element | Remote (RSE) | Cloud HSM |
| **Affinidi** | Affinidi Vault | Remote (Cloud) | Cloud HSM |

### Implications

**Device-Local (EUDI):**
- Keys never leave device
- Offline signing possible
- Hardware attestation available
- User controls key lifecycle

**Remote (Procivis RSE, Affinidi):**
- Professional key management
- Multi-device access
- Network dependency for signing
- Key backup/recovery managed

---

## 4. Protocol Support

| Feature | EUDI | Procivis | Affinidi |
|---------|------|----------|----------|
| **OID4VP** | v1.0 Final | Supported | Supported |
| **ISO 18013-5** | Full support | Supported | Not supported |
| **PEX v1** | Via DCQL | Supported | Supported |
| **PEX v2** | Via DCQL | Supported | Not documented |
| **DCQL** | Supported | Not documented | Not documented |
| **SIOPv2** | Supported | Not documented | Not documented |

### Query Language Comparison

| Implementation | Primary Query Language | Format |
|----------------|------------------------|--------|
| EUDI | DCQL (Digital Credentials Query Language) | JSON object |
| Procivis | DIF Presentation Exchange v1/v2 | JSON object |
| Affinidi | DIF Presentation Exchange | JSON object |

---

## 5. Transport Mode Support

### EUDI

| Mode | Android | iOS | Use Case |
|------|---------|-----|----------|
| **HTTP (OID4VP)** | ✓ | ✓ | Remote verification, cross-device |
| **BLE** | ✓ | ✓ | Proximity, offline-capable |
| **NFC** | ✓ | ✓ | Tap-to-share scenarios |
| **Deep Link** | ✓ | ✓ | Same-device flows |

### Procivis

| Mode | Support | Use Case |
|------|---------|----------|
| **HTTP** | ✓ | Remote verification |
| **BLE** | ✓ | Proximity verification |
| **NFC** | ✓ | Tap-to-share |

### Affinidi

| Mode | Support | Use Case |
|------|---------|----------|
| **Redirect** | ✓ | Standard web flow |
| **WebSocket** | ✓ | Real-time communication |

---

## 6. Credential Format Support

| Format | EUDI | Procivis | Affinidi |
|--------|------|----------|----------|
| **MSO-MDOC** | ✓ | ✓ | Not documented |
| **SD-JWT-VC** | ✓ | ✓ | ✓ |
| **W3C VC (JSON-LD)** | Limited | ✓ | ✓ |

---

## 7. Trust Establishment

### EUDI: PKI-Based

```
Verifier Certificate
       │
       ▼
Certificate Chain Validation
       │
       ▼
Trust Store (IACA certificates)
       │
       ▼
Display: Trusted/Untrusted indicator
```

**Implementation:**
- X.509 certificate validation
- Client ID schemes: `x509_san_dns`, `x509_san_hash`
- Trust store configured at build time
- User sees verified/unverified indicator

### Procivis: Protocol-Based

```
Invitation URL
       │
       ▼
Protocol Detection
       │
       ▼
Configuration-based trust
       │
       ▼
Display: Verifier information
```

**Implementation:**
- Trust based on invitation source
- Protocol capabilities determine behavior
- Flexible configuration

### Affinidi: OAuth-Based

```
Application initiates request
       │
       ▼
Iota Framework validates app
       │
       ▼
Vault displays app identity
       │
       ▼
User grants consent
```

**Implementation:**
- OAuth 2.0 authorization model
- Affinidi validates applications
- User sees application identity

---

## 8. User Consent Flow

### EUDI

1. Display verifier identity (from certificate)
2. Show trust status (verified/unverified)
3. List requested credentials and claims
4. User selects/deselects optional claims
5. Biometric authentication to sign
6. Presentation sent

### Procivis

1. Display proof request
2. Show available credentials
3. User selects credentials (if multiple)
4. User selects claims (selective disclosure)
5. RSE PIN entry for signing
6. Presentation sent

### Affinidi

1. User redirected to Vault
2. Vault displays requesting application
3. Vault shows requested data
4. User consents
5. Vault signs and returns VP
6. User redirected back to application

---

## 9. Error Handling Patterns

| Implementation | Error Type | Handling |
|----------------|------------|----------|
| **EUDI** | Partial state classes | Type-safe error propagation |
| **Procivis** | React Query mutations | Error callbacks, retry logic |
| **Affinidi** | API error responses | HTTP status codes, error messages |

### EUDI Example (Kotlin)

```kotlin
sealed class PresentationRequestInteractorPartialState {
    data class Success(val documents: List<RequestDataUi>) : ...
    data class Failure(val error: String) : ...
    object NoData : ...
}
```

### Procivis Example (TypeScript)

```typescript
const { mutate, error, isError } = useMutation({
  onError: (error) => {
    if (isRseLockedError(error)) {
      // Handle RSE locked state
    }
  },
});
```

---

## 10. Offline Capabilities

| Implementation | Offline Presentation | Offline Request Reception |
|----------------|---------------------|---------------------------|
| **EUDI** | ✓ (BLE/NFC) | ✓ (QR code) |
| **Procivis** | ✓ (BLE/NFC) | ✓ (QR code) |
| **Affinidi** | ✗ | ✗ |

---

## 11. Decision Matrix

### Choose EUDI When:

- ✓ Building for European Digital Identity ecosystem
- ✓ Hardware-backed key security is required
- ✓ Offline/proximity presentation is needed
- ✓ Native platform performance is important
- ✓ Regulatory compliance (eIDAS 2.0) is required

### Choose Procivis When:

- ✓ Cross-platform development efficiency is priority
- ✓ Multiple PEX versions must be supported
- ✓ Remote signing (RSE) is acceptable
- ✓ React Native expertise available
- ✓ Flexible protocol configuration needed

### Choose Affinidi When:

- ✓ Web-first application
- ✓ Quick integration time needed
- ✓ No mobile app development capability
- ✓ Cloud-managed security is acceptable
- ✓ Consent-based data sharing focus

---

## 12. Integration Complexity

| Aspect | EUDI | Procivis | Affinidi |
|--------|------|----------|----------|
| **Time to hello world** | Days | Days | Hours |
| **Mobile app required** | Yes | Yes | No |
| **Backend integration** | Minimal | Moderate | Required |
| **Customization depth** | High | High | Moderate |
| **Maintenance burden** | High (2 platforms) | Medium (1 codebase) | Low (SaaS) |

---

## 13. Security Trade-offs

| Aspect | Device-Local (EUDI) | Remote Signing (Procivis/Affinidi) |
|--------|---------------------|-----------------------------------|
| **Key compromise** | Requires device compromise | Requires service compromise |
| **Availability** | Works offline | Requires connectivity |
| **Attestation** | Hardware attestation possible | Service attestation |
| **Recovery** | Key loss = credential loss | Backup/recovery possible |
| **Auditability** | Device logs | Service logs |

---

## 14. Summary Recommendations

**For Government/Regulated Use:** EUDI
- Hardware-backed keys meet security requirements
- ISO 18013-5 for official documents
- PKI-based verifier trust

**For Enterprise Deployment:** Procivis
- Flexible protocol support
- Remote signing simplifies key management
- Cross-platform efficiency

**For Consumer Web Applications:** Affinidi
- Fastest integration path
- No mobile development required
- Consent-focused UX
