# Presentation Request — Comparative Analysis

This document compares how EUDI, Procivis ONE, and Affinidi handle the presentation request stage of the verification flow.

---

## 1. Entry Point Comparison

### EUDI (Android)

**Location:** `DeepLinkHelper.kt`

```kotlin
enum class DeepLinkType {
    OPENID4VP(
        schemas = listOf(
            BuildConfig.OPENID4VP_SCHEME,        // "openid4vp"
            BuildConfig.EUDI_OPENID4VP_SCHEME,   // "eudi-openid4vp"
            BuildConfig.MDOC_OPENID4VP_SCHEME,   // "mdoc-openid4vp"
            BuildConfig.HAIP_OPENID4VP_SCHEME    // "haip-openid4vp"
        )
    )
}

fun handleDeepLinkAction(uri: Uri): DeepLinkAction {
    // Parse URI scheme
    // Route to PresentationRequest screen
}
```

**Supported schemes:** `openid4vp`, `eudi-openid4vp`, `mdoc-openid4vp`, `haip-openid4vp`

### EUDI (iOS)

**Location:** `Application.swift`, `DeepLinkController.swift`

```swift
.onOpenURL { url in
    if hasDeepLink(url: url) {
        walletKitController.startSameDevicePresentation(deepLink: url)
    }
}

// Deep link detection
private func hasDeepLink(url: URL) -> Bool {
    let schemes = ["openid4vp", "haip-vp"]
    return schemes.contains(url.scheme ?? "")
}
```

### Procivis

**Location:** `invitation-process-screen.tsx`

```typescript
const handleInvitation = async (invitation: string) => {
    const invitationResult = await core.handleInvitation(invitation);

    if (invitationResult.type_ === 'PROOF_REQUEST') {
        managementNavigation.replace('ShareCredential', {
            params: { request: invitationResult },
            screen: 'ProofRequest',
        });
    }
};
```

**Mechanism:** Generic invitation URL handling, type detection in core library

### Affinidi

**Location:** Application API integration

```typescript
// Redirect flow - user redirected from application
// Application calls Iota Framework API
const { correlationId, transactionId, jwt } =
    await api.initiateDataSharingRequest({
        configurationId,
        mode: IotaConfigurationDtoModeEnum.Redirect,
        queryId,
        correlationId: uuidv4(),
        nonce,
        redirectUri,
    });

// User redirected to Vault with jwt
```

**Mechanism:** API-initiated, redirect to Affinidi Vault

---

## 2. URI Scheme Support

| Scheme | EUDI Android | EUDI iOS | Procivis | Affinidi |
|--------|--------------|----------|----------|----------|
| `openid4vp://` | ✓ | ✓ | ✓ | N/A |
| `eudi-openid4vp://` | ✓ | ✗ | ✗ | N/A |
| `mdoc-openid4vp://` | ✓ | ✗ | ✗ | N/A |
| `haip-openid4vp://` | ✓ | ✓ (`haip-vp`) | ✗ | N/A |
| Custom invitation URL | ✗ | ✗ | ✓ | ✓ |

---

## 3. Request Resolution

### EUDI

Request resolution happens in the SDK:

```kotlin
// Android: WalletCorePresentationController
fun setConfig(config: PresentationControllerConfig.OpenId4VP(uri)) {
    // SDK resolves URI
    // Fetches request_uri if present
    // Parses presentation_definition
    // Returns RequestReceived event
}
```

**Features:**
- Automatic request_uri resolution
- JWT validation for signed requests
- Client ID scheme support (x509_san_dns, x509_san_hash)
- Presentation definition parsing

### Procivis

Core library handles resolution:

```typescript
// Invitation handling returns parsed request
const invitationResult = await core.handleInvitation(invitation);
// invitationResult.type_ === 'PROOF_REQUEST'
// Contains parsed presentation definition
```

**Features:**
- Protocol auto-detection
- PEX v1 and v2 support
- Presentation definition parsing in core

### Affinidi

Server-side resolution:

```typescript
// Request defined in Affinidi Portal configuration
// Application fetches via API
const response = await api.fetchIotaVpResponse({
    configurationId,
    correlationId,
    transactionId,
    responseCode,
});
```

**Features:**
- Pre-configured presentation definitions
- Server-side query management
- Portal-based configuration

---

## 4. Client Identification

### EUDI

**Configuration:** `WalletCoreConfigImpl.kt`

```kotlin
configureOpenId4Vp {
    withClientIdSchemes(
        listOf(
            ClientIdScheme.X509SanDns,
            ClientIdScheme.X509SanHash
        )
    )
}
```

**Trust establishment:**
- Certificate chain validation
- Trust store with IACA certificates
- Verifier name from certificate CN
- Trust status displayed to user

**User sees:**
```
┌─────────────────────────────────┐
│  Verifier: Example Corp         │
│  ✓ Verified                     │
│                                 │
│  Requesting:                    │
│  • Given Name                   │
│  • Family Name                  │
└─────────────────────────────────┘
```

### Procivis

**Trust display:**

```typescript
// Verifier information from invitation
<Text>{proof.verifierName}</Text>
// Trust established through invitation source
```

### Affinidi

**Trust display:**

- Application registered in Affinidi Portal
- Vault displays registered application name
- OAuth-style consent screen

---

## 5. Request Data Extraction

### EUDI

**Data flow:**

```
URI → SDK → TransferEventPartialState.RequestReceived
         ↓
    requestedDocuments: List<RequestedDocument>
    verifierName: String?
    verifierIsTrusted: Boolean
```

**RequestedDocument structure:**
```kotlin
data class RequestedDocument(
    val docId: String,
    val requestedItems: Map<String, List<RequestedItem>>,
    val readerAuth: ReaderAuth?
)
```

### Procivis

**Data flow:**

```
invitation → core.handleInvitation() → InvitationResult
                                     ↓
    type_: 'PROOF_REQUEST'
    proofId: String
    protocol: VerificationProtocol
```

Then fetch presentation definition:
```typescript
const presentationDefinition = await core.getPresentationDefinition(proofId);
// or
const presentationDefinitionV2 = await core.getPresentationDefinitionV2(proofId);
```

### Affinidi

**Data flow:**

```
configurationId → initiateDataSharingRequest() → {correlationId, transactionId, jwt}
                                               ↓
User interaction with Vault
                                               ↓
responseCode → fetchIotaVpResponse() → {vpToken, nonce}
```

---

## 6. Proximity Request Handling (BLE/NFC)

### EUDI

**QR-based engagement:**

```kotlin
// Android
eudiWallet.startProximityPresentation()
// Generates QR code for device engagement
// Verifier scans wallet's QR

// State
TransferEventPartialState.QrEngagementReady(qrCode: String)
```

```swift
// iOS
let qrImage = try await proximitySessionCoordinator.startQrEngagement()
// Display QR for verifier to scan
```

**NFC engagement:**

```kotlin
eudiWallet.enableNFCEngagement(true)
```

### Procivis

```typescript
// QR code generation for verifier
const { proposeProof } = useProposeProof();
const proofId = await proposeProof({
    protocol: VerificationProtocol.ISO_MDL,
});
// Generate QR with proof request
```

### Affinidi

**Not supported** — Affinidi is HTTP-only, no proximity/offline modes.

---

## 7. Error Handling

### EUDI

```kotlin
sealed class TransferEventPartialState {
    data class Error(val error: String) : TransferEventPartialState()
}

// Specific error states in interactors
sealed class PresentationRequestInteractorPartialState {
    data class Failure(val error: String) : PresentationRequestInteractorPartialState()
    object NoData : PresentationRequestInteractorPartialState()
}
```

### Procivis

```typescript
try {
    const result = await core.handleInvitation(invitation);
} catch (error) {
    // Error handling via try/catch
    setError(error.message);
}
```

### Affinidi

```typescript
const response = await api.initiateDataSharingRequest(...);
if (response.error) {
    // HTTP error handling
    handleError(response.error);
}
```

---

## 8. Comparison Summary

| Aspect | EUDI | Procivis | Affinidi |
|--------|------|----------|----------|
| **Entry mechanism** | Deep link (custom schemes) | Invitation URL | API + Redirect |
| **Scheme variety** | 4 schemes | Generic | N/A (redirect) |
| **Request resolution** | SDK-handled | Core library | Server-side |
| **Client ID schemes** | X.509, DID | Protocol-based | OAuth |
| **Trust display** | Certificate-based | Invitation source | Portal registration |
| **Proximity support** | Full (BLE, NFC) | Full (BLE, NFC) | None |
| **Offline capable** | Yes (request receipt) | Yes (request receipt) | No |

---

## 9. Strengths and Weaknesses

### EUDI

**Strengths:**
- Rich URI scheme support
- Strong client authentication (X.509)
- Full proximity support
- Clear trust indicators

**Weaknesses:**
- Platform-specific implementation
- Complex configuration
- Requires native development

### Procivis

**Strengths:**
- Flexible protocol handling
- Cross-platform codebase
- PEX v1/v2 support
- BLE/NFC support

**Weaknesses:**
- Less standardized entry points
- React Native bridge overhead
- Fewer client ID schemes documented

### Affinidi

**Strengths:**
- Simplest integration (web SDK)
- Pre-configured queries via Portal
- No mobile development required
- Quick time to integration

**Weaknesses:**
- No offline/proximity support
- Cloud-dependent
- Less flexible client identification
- No signed requests documented

---

## 10. Implementation Recommendations

### For European Identity Use Cases

→ **Use EUDI**
- X.509 client authentication matches regulatory requirements
- Multiple scheme support for interoperability
- Proximity mode for in-person verification

### For Cross-Platform Efficiency

→ **Use Procivis**
- Single codebase for Android/iOS
- Flexible protocol support
- BLE/NFC available when needed

### For Web-Only Applications

→ **Use Affinidi**
- No mobile app required
- Fastest integration path
- Consent-focused UX
