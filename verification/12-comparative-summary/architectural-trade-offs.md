# Architectural Trade-offs

A deep dive into the design decisions and trade-offs made by each verification implementation.

---

## Key Architecture Model

### EUDI: Device-Centric

```
┌─────────────────────────────────────────────────────────────────┐
│                        EUDI Architecture                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────────┐     ┌─────────────────┐                   │
│   │   Android App   │     │    iOS App      │                   │
│   │   (Kotlin)      │     │    (Swift)      │                   │
│   └────────┬────────┘     └────────┬────────┘                   │
│            │                       │                             │
│   ┌────────▼────────┐     ┌────────▼────────┐                   │
│   │  Android SDK    │     │    iOS SDK      │                   │
│   │  (wallet-core)  │     │  (wallet-core)  │                   │
│   └────────┬────────┘     └────────┬────────┘                   │
│            │                       │                             │
│   ┌────────▼────────┐     ┌────────▼────────┐                   │
│   │ Android Keystore│     │ Secure Enclave  │                   │
│   │   (Hardware)    │     │   (Hardware)    │                   │
│   └─────────────────┘     └─────────────────┘                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Trade-off:** Security vs Development Cost

- **Benefit:** Keys never leave device hardware; highest security model
- **Cost:** Two completely separate codebases to maintain

### Procivis: Cross-Platform with Remote Security

```
┌─────────────────────────────────────────────────────────────────┐
│                     Procivis Architecture                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌───────────────────────────────────────────┐                 │
│   │          React Native Application          │                 │
│   │        (Single JavaScript codebase)        │                 │
│   └───────────────────┬───────────────────────┘                 │
│                       │                                          │
│   ┌───────────────────▼───────────────────────┐                 │
│   │         @procivis/react-native-one-core    │                 │
│   │              (Native bridge)               │                 │
│   └───────────────────┬───────────────────────┘                 │
│                       │                                          │
│   ┌───────────────────▼───────────────────────┐                 │
│   │              Procivis Core                 │                 │
│   │       (Rust library, cross-platform)       │                 │
│   └───────────────────┬───────────────────────┘                 │
│                       │                                          │
│   ┌───────────────────▼───────────────────────┐                 │
│   │      Remote Secure Element (RSE)           │                 │
│   │         (Network-based HSM)                │                 │
│   └───────────────────────────────────────────┘                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Trade-off:** Development Efficiency vs Security Architecture

- **Benefit:** Single codebase, consistent behavior across platforms
- **Cost:** Network dependency for cryptographic operations

### Affinidi: Platform-Managed

```
┌─────────────────────────────────────────────────────────────────┐
│                     Affinidi Architecture                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌───────────────────────────────────────────┐                 │
│   │           Web Application                  │                 │
│   │        (Browser-based client)              │                 │
│   └───────────────────┬───────────────────────┘                 │
│                       │                                          │
│   ┌───────────────────▼───────────────────────┐                 │
│   │          Affinidi SDK / TDK                │                 │
│   │         (API client library)               │                 │
│   └───────────────────┬───────────────────────┘                 │
│                       │                                          │
│   ┌───────────────────▼───────────────────────┐                 │
│   │        Affinidi Platform Services          │                 │
│   │   ┌───────────┐  ┌───────────┐            │                 │
│   │   │   Vault   │  │   Iota    │            │                 │
│   │   │(Credentials)│ │ Framework │            │                 │
│   │   └───────────┘  └───────────┘            │                 │
│   └───────────────────────────────────────────┘                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Trade-off:** Integration Speed vs Control

- **Benefit:** Fastest integration, managed infrastructure
- **Cost:** Platform dependency, less customization

---

## Protocol Support Trade-offs

### Presentation Definition Approach

| Implementation | Approach | Trade-off |
|----------------|----------|-----------|
| EUDI | PEX v1 + DCQL emerging | Stability vs newer features |
| Procivis | PEX v1 + v2 | Flexibility vs complexity |
| Affinidi | Platform abstraction | Simplicity vs protocol access |

### EUDI: Conservative Protocol Support

```kotlin
// EUDI supports established formats
when (val format = requestedDocument.format) {
    is Format.MsoMdoc -> handleMdoc(format)
    is Format.SdJwtVc -> handleSdJwt(format)
    // Limited to well-defined formats
}
```

**Trade-off:** Interoperability vs Feature Richness

- Prioritizes ISO/OID4VP compliance
- May lag behind emerging protocol features
- Higher confidence in implementation correctness

### Procivis: Flexible Protocol Support

```typescript
// Procivis handles multiple PEX versions
const isV2Request = proof?.claims?.some(
    c => c.credential?.proofInputClaims
);

if (isV2Request) {
    return handleV2Flow(proof);
} else {
    return handleV1Flow(proof);
}
```

**Trade-off:** Feature Coverage vs Maintenance

- Supports wider range of verifier implementations
- Higher testing surface area
- More complex state management

### Affinidi: Abstracted Protocol

```typescript
// Affinidi abstracts protocol details
const presentation = await iotaFramework
    .selectPresentation(configurationId)
    .authorize();
```

**Trade-off:** Simplicity vs Flexibility

- Developers don't need protocol expertise
- Platform controls protocol upgrades
- Less control over specific behaviors

---

## State Management Trade-offs

### EUDI: Event-Driven State

```kotlin
sealed class TransferEventPartialState {
    data object Connected : TransferEventPartialState()
    data class RequestReceived(
        val requestData: RequestData
    ) : TransferEventPartialState()
    data object ResponseSent : TransferEventPartialState()
    data class Redirect(val uri: URI) : TransferEventPartialState()
    data class Error(val error: String) : TransferEventPartialState()
    data object Disconnected : TransferEventPartialState()
}
```

**Characteristics:**
- Immutable state transitions
- Clear event lineage
- Platform-native patterns

**Trade-off:** Type safety at cost of verbosity

### Procivis: Mutation-Based State

```typescript
const { mutateAsync: acceptProof, isPending } = useMutation(
    async ({ interactionId, credentials }) =>
        core.holderSubmitProof(interactionId, credentials)
);
```

**Characteristics:**
- React Query patterns
- Optimistic updates possible
- Familiar to web developers

**Trade-off:** Developer familiarity vs state predictability

### Affinidi: Callback-Based State

```typescript
const iotaSession = new Affinidi.IotaSession({
    onStart: () => setLoading(true),
    onSuccess: (response) => handleSuccess(response),
    onError: (error) => handleError(error),
});
```

**Characteristics:**
- Simple callback model
- Platform manages transitions
- Limited intermediate states

**Trade-off:** Simplicity vs granular control

---

## Error Handling Philosophy

### EUDI: Typed Error Hierarchy

```kotlin
sealed class WalletError {
    sealed class Presentation : WalletError() {
        data class InvalidRequest(val reason: String) : Presentation()
        data class MissingCredential(val type: String) : Presentation()
        data class KeyAccessDenied : Presentation()
        data class SigningFailed(val cause: Throwable) : Presentation()
    }
}
```

**Philosophy:** Exhaustive error handling enforced by compiler

- All error cases must be handled
- Clear error categorization
- Verbose but safe

### Procivis: Exception with Categorization

```typescript
try {
    await core.holderSubmitProof(interactionId, credentials);
} catch (e) {
    if (isRSELockedError(e)) {
        // RSE-specific recovery
    } else if (isNetworkError(e)) {
        // Network retry logic
    } else {
        // Generic error handling
    }
}
```

**Philosophy:** Runtime categorization with helpers

- Flexible error handling
- Requires discipline to handle all cases
- Runtime discovery of edge cases

### Affinidi: Standardized Error Response

```json
{
  "error": "invalid_presentation_request",
  "error_description": "Required credential type not found",
  "error_code": "ERR-001"
}
```

**Philosophy:** API-style error responses

- Consistent error format
- Platform handles recovery where possible
- Limited client-side recovery options

---

## Transport Layer Trade-offs

### Online vs Proximity

| Capability | EUDI | Procivis | Affinidi |
|------------|------|----------|----------|
| Same-device HTTPS | ✓ | ✓ | ✓ |
| Cross-device QR | ✓ | ✓ | ✓ |
| BLE proximity | ✓ | ✓ | ✗ |
| NFC | ✓ | ✓ | ✗ |
| Offline operation | ✓ | Partial | ✗ |

### EUDI: Full Transport Spectrum

```kotlin
// EUDI handles all transport modes
enum class TransportMode {
    HTTP_REDIRECT,
    HTTP_POST,
    BLE,
    NFC,
    WIFI_AWARE
}
```

**Trade-off:** Use case coverage vs implementation complexity

- Supports regulatory requirements (e.g., ID verification)
- Complex session management across transports
- Hardware capability requirements

### Affinidi: Network-Only

```typescript
// Affinidi: web-native transport
const response = await fetch(verifierEndpoint, {
    method: 'POST',
    body: JSON.stringify(presentation)
});
```

**Trade-off:** Simplicity vs offline capability

- Simpler implementation and testing
- Always-online requirement
- Not suitable for in-person ID checks

---

## Security Model Trade-offs

### Key Storage Comparison

```
EUDI (Device Hardware):
┌─────────────────────────────────┐
│ Secure Enclave / Keystore       │
│ ┌─────────────────────────────┐ │
│ │  Private Key (non-export)   │ │
│ │  Sign operation in hardware │ │
│ └─────────────────────────────┘ │
└─────────────────────────────────┘

Procivis (Remote HSM):
┌─────────────────────────────────┐
│ Remote Secure Element           │
│ ┌─────────────────────────────┐ │
│ │  Private Key in HSM         │ │
│ │  API-based signing          │ │
│ └─────────────────────────────┘ │
└─────────────────────────────────┘

Affinidi (Platform Cloud):
┌─────────────────────────────────┐
│ Affinidi Cloud HSM              │
│ ┌─────────────────────────────┐ │
│ │  Managed key infrastructure │ │
│ │  Platform handles access    │ │
│ └─────────────────────────────┘ │
└─────────────────────────────────┘
```

### Security vs Convenience Matrix

| Factor | EUDI | Procivis | Affinidi |
|--------|------|----------|----------|
| Key extraction risk | None | None (HSM) | None (HSM) |
| Offline signing | ✓ | ✗ | ✗ |
| Multi-device | ✗ | ✓ | ✓ |
| Device loss impact | High | Low | Low |
| Recovery complexity | Complex | Simple | Simple |

---

## Recommendations by Scenario

### High-Security Government Use

**Choose: EUDI**

- Hardware-backed keys mandatory
- Offline capability required
- Accept higher development cost
- Plan for device lifecycle management

### Enterprise Cross-Platform

**Choose: Procivis**

- Balance security and development efficiency
- RSE provides enterprise-grade key management
- React Native skill availability
- Accept network dependency

### Consumer Web Application

**Choose: Affinidi**

- Prioritize time-to-market
- User base expects web experience
- Accept platform dependency
- Leverage managed infrastructure

### Hybrid Considerations

For organizations with diverse needs:

1. **Start simple** — Affinidi for MVP
2. **Add mobile** — Procivis for app users
3. **Add compliance** — EUDI for regulated flows
4. **Use OID4VP** — Interoperability across all
