# Verifier Metadata — Implementation Details

This document details how EUDI (Android/iOS) and Procivis ONE handle verifier metadata discovery, validation, and display.

---

## EUDI Android

### Reader Authentication Handling

**Location:** `core-logic/src/main/java/eu/europa/ec/corelogic/controller/WalletCorePresentationController.kt`

The controller extracts verifier information from the processed request:

```kotlin
processedRequest?.let { safeProcessedRequest ->
    val requestedDocuments = safeProcessedRequest.requestedDocuments.getOrThrow()

    verifierName = requestedDocuments.requestedDocuments
        .firstOrNull()?.readerAuth?.readerCommonName

    val isTrusted = requestedDocuments.requestedDocuments
        .firstOrNull()?.readerAuth?.isVerified == true
    verifierIsTrusted = isTrusted

    emit(
        TransferEventPartialState.RequestReceived(
            processedRequest = safeProcessedRequest,
            requestedDocuments = requestedDocuments.requestedDocuments,
            verifierName = verifierName,
            verifierIsTrusted = verifierIsTrusted
        )
    )
}
```

**Key properties extracted:**

| Property | Source | Purpose |
|----------|--------|---------|
| `readerCommonName` | X.509 Subject CN | Display name |
| `isVerified` | Certificate chain validation | Trust indicator |

### Trusted Certificates Configuration

**Location:** `core-logic/src/main/java/eu/europa/ec/corelogic/config/WalletCoreConfigImpl.kt`

The wallet is configured with trusted reader certificates:

```kotlin
configureOpenId4Vp {
    withClientIdSchemes(
        listOf(
            ClientIdScheme.X509SanDns,
            ClientIdScheme.X509SanUri
        )
    )
}
```

The SDK validates reader certificates against configured trust anchors.

### UI Display

**Location:** `presentation-feature/src/main/java/eu/europa/ec/presentationfeature/ui/request/PresentationRequestViewModel.kt`

```kotlin
private fun onRequestReceived(
    requestedDocuments: List<RequestedDocument>,
    verifierName: String?,
    verifierIsTrusted: Boolean,
) {
    // Display verifier info to user
    viewState = viewState.copy(
        verifierName = verifierName ?: "Unknown Verifier",
        verifierIsTrusted = verifierIsTrusted
    )
}
```

---

## EUDI iOS

### Trust Configuration

**Location:** `Modules/logic-core/Sources/Config/WalletKitConfig.swift`

Reader certificate trust is configured at initialization:

```swift
var readerConfig: ReaderConfig {
    let certificates = [
        "pidissuerca02_cz",
        "pidissuerca02_ee",
        "pidissuerca02_eu",
        "pidissuerca02_lu",
        "pidissuerca02_nl",
        "pidissuerca02_pt",
        "pidissuerca02_ut",
        "r45_staging"
    ]
    let certsData: [Data] = certificates.compactMap {
        Data(name: $0, ext: "der")
    }
    return .init(trustedCerts: certsData)
}
```

These DER-encoded certificates form the trust anchors for reader validation.

### OID4VP Client ID Scheme Configuration

```swift
var vpConfig: OpenId4VpConfiguration {
    .init(clientIdSchemes: [.x509SanDns, .x509SanUri])
}
```

### Verifier Information Extraction

**Location:** `Modules/logic-core/Sources/Coordinator/RemoteSessionCoordinator.swift`

```swift
private func createRequest() -> PresentationRequest {
    PresentationRequest(
        items: session.disclosedDocuments,
        relyingParty: session.readerCertIssuer ?? LocalizableStringKey.unknownVerifier.toString,
        dataRequestInfo: session.readerCertValidationMessage ?? LocalizableStringKey.requestDataInfoNotice.toString,
        isTrusted: session.readerCertIssuerValid == true
    )
}
```

**Session properties:**

| Property | Type | Source |
|----------|------|--------|
| `readerCertIssuer` | `String?` | Certificate Common Name |
| `readerCertIssuerValid` | `Bool?` | Chain validation result |
| `readerCertValidationMessage` | `String?` | Validation status message |

### Presentation Request Model

**Location:** `Modules/logic-core/Sources/Coordinator/Model/PresentationRequest.swift`

```swift
public struct PresentationRequest: Sendable, Equatable {
    public let items: [DocElements]
    public let relyingParty: String      // Verifier display name
    public let dataRequestInfo: String    // Trust status message
    public let isTrusted: Bool           // Certificate validation result
}
```

### UI Indicator

The trust status flows through to the UI:

**Location:** `Modules/feature-presentation/Sources/UI/Presentation/Request/PresentationRequestViewModel.swift`

```swift
func getRelyingParty() -> LocalizableStringKey {
    guard let coordinator = self.coordinator else {
        return .unknownVerifier
    }
    return .custom(requestDataUiModel.relyingParty)
}

func getRelyingPartyIsTrusted() -> Bool {
    return requestDataUiModel.isTrusted
}
```

---

## Procivis ONE

### Invitation Processing

**Location:** `app/screens/credential/invitation-process-screen.tsx`

Procivis handles the initial URL processing:

```typescript
const handleInvitationUrl = useCallback(async (url: string) => {
    // Follow HTTP redirects if needed
    const targetUrl = url.startsWith('http')
        ? await RNBlobUtil.fetch('GET', url, {}).then(resp => resp.redirects[resp.redirects.length - 1] || url)
        : url;

    const invitationResponse = await invitationHandler(targetUrl);

    switch (invitationResponse.type) {
        case HandleInvitationResponseBindingEnum.PROOF_REQUEST:
            navigation.navigate('ProofRequest', {
                proofId: invitationResponse.proofId,
                interactionId: invitationResponse.interactionId,
            });
            break;
        // ...
    }
}, [invitationHandler, navigation]);
```

### Proof Detail Information

**Location:** `app/screens/credential/proof-request-screen.tsx`

Verifier information is extracted from the proof detail:

```typescript
const { data: proof } = useProofDetail(proofId);

// Verifier information from proof
const verifierName = proof?.verifierDid;
const interactionId = proof?.interactionId;
```

### Protocol Capabilities

Procivis checks protocol-level capabilities:

```typescript
const verificationProtocol = config.verificationProtocol[proof.protocol];
const protocolCapabilities = verificationProtocol?.capabilities;

// Check supported features
const supportedPresentationDefinition = (protocolCapabilities as any)[
    'supportedPresentationDefinition'
] as unknown as string[];
```

### Transport-Based Trust

For ISO mDL flows, Procivis uses BLE proximity as part of the trust model:

**Location:** `app/navigators/app-navigator.tsx`

```typescript
const coreConfig = {
    verificationProtocol: {
        ISO_MDL: {
            enabled: config.featureFlags.isoMdl,
        },
    },
    verificationEngagement: {
        NFC: {
            display: 'verificationEngagement.nfc',
            enabled: true,
            order: 2,
        },
    },
};
```

---

## Comparative Summary

| Aspect | EUDI Android | EUDI iOS | Procivis ONE |
|--------|--------------|----------|--------------|
| **Trust model** | X.509 certificate chain | X.509 certificate chain | Protocol-based |
| **Client ID schemes** | x509_san_dns, x509_san_uri | x509_san_dns, x509_san_uri | DID, redirect_uri |
| **Trust anchors** | Bundled CA certs | Bundled DER files | Core library config |
| **Verifier name source** | Certificate CN | Certificate CN | Protocol metadata / DID |
| **Trust indicator** | Boolean `isVerified` | Boolean `isTrusted` | Protocol state |
| **UI display** | Name + verified badge | Name + verified badge | Name |

---

## Key Code Paths

### EUDI Android Trust Validation Flow

```
Deep link/QR received
    │
    ▼
WalletCorePresentationController.setConfig()
    │
    ▼
eudiWallet.startRemotePresentation(uri)
    │
    ▼
SDK validates request signature
SDK validates certificate chain
SDK extracts readerAuth info
    │
    ▼
TransferEventPartialState.RequestReceived(
    verifierName: readerCommonName,
    verifierIsTrusted: isVerified
)
    │
    ▼
PresentationRequestViewModel displays
```

### EUDI iOS Trust Validation Flow

```
Deep link received
    │
    ▼
WalletKitController.startSameDevicePresentation()
    │
    ▼
PresentationSession validates request
    │
    ▼
RemoteSessionCoordinator.requestReceived()
    │
    ▼
PresentationRequest(
    relyingParty: readerCertIssuer,
    isTrusted: readerCertIssuerValid
)
    │
    ▼
PresentationRequestView displays
```

### Procivis Trust Flow

```
QR scan / Deep link
    │
    ▼
useInvitationHandler() processes URL
    │
    ▼
core.handleInvitation() returns proof context
    │
    ▼
useProofDetail() fetches verifier info
    │
    ▼
ProofRequestScreen displays
```

---

## Affinidi

Affinidi uses a cloud-based verification service (Affinidi Iota Framework). Based on public documentation:

- **Verifier registration**: Verifiers register with Affinidi and obtain credentials
- **Client metadata**: Provided during Iota configuration
- **Trust model**: Affinidi-mediated trust through the Iota service
- **No direct reader certificates**: Trust established through Affinidi's verification service

**Note:** Affinidi analysis is based on public documentation. Source code is not available for direct inspection.
