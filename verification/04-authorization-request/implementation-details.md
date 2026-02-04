# Authorization Request — Implementation Details

This document details how EUDI (Android/iOS) and Procivis ONE parse, validate, and process OID4VP authorization requests.

---

## EUDI Android

### Deep Link Parsing

**Location:** `ui-logic/src/main/java/eu/europa/ec/uilogic/navigation/helper/DeepLinkHelper.kt`

The app registers for multiple OID4VP URI schemes:

```kotlin
enum class DeepLinkType(val schemas: List<String>, val host: String? = null) {
    OPENID4VP(
        schemas = listOf(
            BuildConfig.OPENID4VP_SCHEME,        // openid4vp
            BuildConfig.EUDI_OPENID4VP_SCHEME,   // eudi-openid4vp
            BuildConfig.MDOC_OPENID4VP_SCHEME,   // mdoc-openid4vp
            BuildConfig.HAIP_OPENID4VP_SCHEME    // haip-openid4vp
        )
    ),
    // ...
}
```

### Request Processing Flow

**Location:** `core-logic/src/main/java/eu/europa/ec/corelogic/controller/WalletCorePresentationController.kt`

```kotlin
private fun addListener(listener: EudiWalletListenerWrapper) {
    val config = requireInit { _config }
    eudiWallet.addTransferEventListener(listener)

    if (config is PresentationControllerConfig.OpenId4VP) {
        eudiWallet.startRemotePresentation(config.uri.toUri())
    }
}
```

The SDK (`eudi-lib-android-wallet-core`) handles:
1. URI parsing
2. Request URI fetching (if present)
3. JWT validation (if signed)
4. Presentation definition extraction

### SDK Configuration

**Location:** `core-logic/src/main/java/eu/europa/ec/corelogic/config/WalletCoreConfigImpl.kt`

```kotlin
configureOpenId4Vp {
    withClientIdSchemes(
        listOf(
            ClientIdScheme.X509SanDns,
            ClientIdScheme.X509SanUri
        )
    )
    withSchemes(
        listOf(
            "openid4vp",
            "eudi-openid4vp",
            "mdoc-openid4vp",
            "haip-openid4vp"
        )
    )
    withFormats(
        Format.MsoMdoc.ES256,
        Format.SdJwtVc.ES256
    )
}
```

### Request State Handling

```kotlin
sealed class TransferEventPartialState {
    data object Connecting : TransferEventPartialState()
    data class Error(val error: String) : TransferEventPartialState()
    data class RequestReceived(
        val processedRequest: ProcessedRequest,
        val requestedDocuments: List<RequestedDocument>,
        val verifierName: String?,
        val verifierIsTrusted: Boolean
    ) : TransferEventPartialState()
    // ...
}
```

The `ProcessedRequest` object contains the parsed authorization request with:
- Presentation definition (parsed input descriptors)
- Client metadata
- Response configuration

---

## EUDI iOS

### Deep Link Entry Point

**Location:** `Sources/Application/Application.swift`

```swift
.onOpenURL { url in
    deepLinkController.handleDeepLinkAction(
        routerHost: routerHost,
        remoteSessionCoordinator: remoteSessionCoordinator
    )
}
```

**Location:** `Modules/logic-ui/Sources/Controller/DeepLinkController.swift`

```swift
func handleDeepLinkAction(
    routerHost: RouterHost,
    remoteSessionCoordinator: RemoteSessionCoordinator?
) {
    switch action.type {
    case .openid4vp, .haip_vp:
        guard let remoteSessionCoordinator else {
            fatalError("DeepLink Action OpenId4VP Requires Remote Session Coordinator")
        }
        routerHost.push(
            with: .featurePresentationModule(
                .presentationRequest(
                    presentationCoordinator: remoteSessionCoordinator,
                    originator: .featureDashboardModule(.dashboard)
                )
            )
        )
    // ...
    }
}
```

### Session Initialization

**Location:** `Modules/logic-core/Sources/Controller/WalletKitController.swift`

```swift
func startSameDevicePresentation(deepLink: URLComponents) async -> RemoteSessionCoordinator {
    let coordinator = RemoteSessionCoordinatorImpl(
        session: walletKit.openId4VpSession(
            url: deepLink.url!
        )
    )
    await coordinator.initialize()
    return coordinator
}
```

### OID4VP Configuration

**Location:** `Modules/logic-core/Sources/Config/WalletKitConfig.swift`

```swift
var vpConfig: OpenId4VpConfiguration {
    .init(clientIdSchemes: [.x509SanDns, .x509SanUri])
}
```

### Request Parsing via EudiWalletKit

The EudiWalletKit library (v0.19.4) handles:
1. URL parsing and scheme validation
2. `request_uri` fetching with HTTP client
3. JWT decoding and signature validation
4. X.509 certificate chain validation
5. Presentation definition parsing

Request data flows through:

```swift
public func requestReceived() async throws -> PresentationRequest {
    guard session.disclosedDocuments.isEmpty == false else {
        throw session.uiError ?? .init(description: "Failed to Find known documents to send")
    }
    return createRequest()
}
```

---

## Procivis ONE

### URL Processing

**Location:** `app/hooks/navigation/deep-link.ts`

```typescript
export const useInvitationHandling = () => {
    const navigation = useNavigation<RootNavigationProp>();
    return useCallback(
        (url: string) => {
            const invitationUrl = parseUniversalLink(url) ?? url;
            navigation.navigate('CredentialManagement', {
                params: { params: { invitationUrl }, screen: 'Processing' },
                screen: 'Invitation',
            });
        },
        [navigation],
    );
};
```

### Redirect Handling

**Location:** `app/screens/credential/invitation-process-screen.tsx`

```typescript
const handleInvitationUrl = useCallback(async (url: string) => {
    // Follow HTTP redirects for request_uri
    const targetUrl = url.startsWith('http')
        ? await RNBlobUtil.fetch('GET', url, {})
            .then(resp => resp.redirects[resp.redirects.length - 1] || url)
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

### Core Library Processing

The `@procivis/react-native-one-core` library processes the request:

```typescript
// Handle invitation returns parsed request type
const invitationResponse = await core.handleInvitation(url);

// For proof requests, get the presentation definition
const presentationDefinition = await core.getPresentationDefinition(proofId);
// or V2:
const presentationDefinitionV2 = await core.getPresentationDefinitionV2(proofId);
```

### Protocol Version Detection

**Location:** `app/screens/credential/proof-request-screen.tsx`

```typescript
const verificationProtocol = config.verificationProtocol[proof.protocol];
const protocolCapabilities = verificationProtocol?.capabilities;

const supportedPresentationDefinition = (protocolCapabilities as any)[
    'supportedPresentationDefinition'
] as unknown as string[];

if (supportedPresentationDefinition?.includes('V2')) {
    setPresentationDefinitionVersion('V2');
} else {
    setPresentationDefinitionVersion('V1');
}
```

---

## Comparative Request Processing Flow

### EUDI Android

```
Deep link → DeepLinkHelper.handleDeepLinkAction()
    │
    ▼
PresentationRequestScreen
    │
    ▼
PresentationRequestInteractor.setConfig(RequestUriConfig)
    │
    ▼
WalletCorePresentationController.setConfig(OpenId4VP(uri))
    │
    ▼
eudiWallet.startRemotePresentation(uri)
    │
    ▼
SDK: Parse → Fetch request_uri → Validate JWT → Parse presentation_definition
    │
    ▼
TransferEventPartialState.RequestReceived
```

### EUDI iOS

```
Deep link → Application.onOpenURL
    │
    ▼
DeepLinkController.handleDeepLinkAction()
    │
    ▼
WalletKitController.startSameDevicePresentation(deepLink)
    │
    ▼
EudiWalletKit.openId4VpSession(url)
    │
    ▼
SDK: Parse → Fetch → Validate → Parse
    │
    ▼
RemoteSessionCoordinator.requestReceived() → PresentationRequest
```

### Procivis ONE

```
Deep link/QR → useInvitationHandling()
    │
    ▼
InvitationProcessScreen
    │
    ▼
handleInvitationUrl() → Follow HTTP redirects
    │
    ▼
core.handleInvitation(url)
    │
    ▼
Core: Parse → Fetch → Validate → Return ProofRequest type
    │
    ▼
ProofRequestScreen
    │
    ▼
core.getPresentationDefinition(proofId)
```

---

## Key Differences

| Aspect | EUDI Android | EUDI iOS | Procivis ONE |
|--------|--------------|----------|--------------|
| **URI schemes** | 4 schemes | 2 schemes | Universal links |
| **Request fetching** | SDK handles | SDK handles | Manual + Core |
| **JWT validation** | SDK (X.509) | SDK (X.509) | Core library |
| **Redirect following** | SDK | SDK | Explicit code |
| **PD version** | Single | Single | V1 and V2 |
| **Error handling** | State events | Async throws | Binding enum |

---

## Error Handling

### EUDI Android

```kotlin
sealed class TransferEventPartialState {
    data class Error(val error: String) : TransferEventPartialState()
}
```

Errors from SDK parsing flow through the state machine.

### EUDI iOS

```swift
public func requestReceived() async throws -> PresentationRequest {
    guard session.disclosedDocuments.isEmpty == false else {
        throw session.uiError ?? .init(description: "Failed to Find known documents")
    }
    // ...
}
```

Errors thrown as Swift errors, caught in interactors.

### Procivis ONE

```typescript
try {
    const invitationResponse = await invitationHandler(url);
} catch (e) {
    // Handle parsing/validation errors
    reportException(e, 'Invitation handling failure');
}
```

---

## Configuration Summary

| Configuration | EUDI Android | EUDI iOS | Procivis |
|---------------|--------------|----------|----------|
| Client ID schemes | x509_san_dns, x509_san_uri | x509_san_dns, x509_san_uri | did, redirect_uri |
| Response modes | direct_post | direct_post | direct_post |
| VP formats | mso_mdoc, vc+sd-jwt | mso_mdoc, vc+sd-jwt | Multiple |
| Signing algorithms | ES256 | ES256 | ES256, EdDSA |
