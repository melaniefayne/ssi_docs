# Holder Binding — Comparative Analysis

This document compares holder binding implementations across EUDI, Procivis, and Affinidi.

---

## Key Binding Approach

| Aspect | EUDI | Procivis | Affinidi |
|--------|------|----------|----------|
| **Key storage** | Hardware (TEE/SE) | Hardware + software | Platform managed |
| **Authentication** | Biometric | RSE PIN / biometric | Platform |
| **Proof type** | Format-specific | Format-specific | Format-specific |
| **Key attestation** | Supported | Supported | N/A |

---

## Supported Proof Formats

| Format | EUDI | Procivis | Affinidi |
|--------|------|----------|----------|
| JWT VP signature | ✓ | ✓ | ✓ |
| SD-JWT KB-JWT | ✓ | ✓ | ✓ |
| mDoc DeviceAuth | ✓ | ✓ | ✗ |
| Data Integrity | ✗ | ✓ | ✓ |

---

## Authentication Flow

### EUDI

```
User selects claims
    │
    ▼
checkForKeyUnlock() called
    │
    ▼
┌───────────────────────────────────┐
│   Biometric Prompt                │
│   "Authenticate to share"         │
│   [Fingerprint] [Face]            │
└───────────────────────────────────┘
    │
    ▼ (success)
Key unlocked for signing
    │
    ▼
generateResponse() with proof
```

### Procivis

```
User selects claims
    │
    ▼
holderSubmitProof() called
    │
    ▼
┌───────────────────────────────────┐
│   RSE PIN Entry (if required)     │
│   Enter PIN: ______               │
│   [Confirm]                       │
└───────────────────────────────────┘
    │
    ▼ (success)
Core generates proof
```

### Affinidi

```
User selects claims
    │
    ▼
Platform handles authentication
    │
    ▼
Proof generated in platform
```

---

## Hardware Security

| Feature | EUDI Android | EUDI iOS | Procivis | Affinidi |
|---------|--------------|----------|----------|----------|
| **Secure Element** | ✓ (Keystore) | ✓ (SE) | ✓ (RSE) | N/A |
| **Biometric lock** | ✓ | ✓ | ✓ | ✓ |
| **Key export** | Never | Never | Never | N/A |
| **Attestation** | ✓ | ✓ | ✓ | N/A |

---

## Error Handling

### EUDI Android

```kotlin
sealed class CheckKeyUnlockPartialState {
    data class UserAuthenticationRequired(
        val authenticationData: List<AuthenticationData>
    ) : CheckKeyUnlockPartialState()

    data object RequestIsReadyToBeSent : CheckKeyUnlockPartialState()

    data class Failure(val error: String) : CheckKeyUnlockPartialState()
}
```

### Procivis

```typescript
try {
    await acceptProof({ credentials, interactionId });
} catch (e) {
    if (isRSELockedError(e)) {
        // RSE needs PIN unlock
        setState(LoaderViewState.Error);
    } else {
        // Other error
        setState(LoaderViewState.Warning);
    }
}
```

---

## Trade-offs

### EUDI

**Strengths:**
- Hardware-backed security
- Standard biometric UX
- Key never leaves device

**Weaknesses:**
- Device-specific implementation
- Biometric hardware required
- Recovery complexity

### Procivis

**Strengths:**
- Remote Secure Element option
- Cross-platform consistency
- Flexible authentication

**Weaknesses:**
- RSE setup required
- Network dependency (for RSE)
- PIN management

### Affinidi

**Strengths:**
- Simple integration
- Platform-managed security
- Consistent UX

**Weaknesses:**
- Less transparency
- Platform dependency
- Limited hardware security

---

## Recommendations

### Choose EUDI-style if:
- Native mobile app
- Hardware security required
- Biometric UX preferred
- Offline operation needed

### Choose Procivis-style if:
- Cross-platform needed
- RSE infrastructure available
- Flexible deployment
- Enterprise scenarios

### Choose Affinidi-style if:
- Rapid development
- Platform management OK
- Web-focused deployment
- Simpler security model
