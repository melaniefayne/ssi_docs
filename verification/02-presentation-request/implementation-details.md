# Presentation Request — Implementation Details

This document details how EUDI (Android/iOS) and Procivis ONE receive and process presentation requests.

---

## EUDI Android

### QR Code Scanning

**Location:** `app/src/main/java/eu/europa/ec/euidi/ui/scanner/`

The wallet uses the camera to scan QR codes containing OID4VP requests:

```kotlin
// QR code detected callback
onQrCodeDetected = { qrContent ->
    // Parse as OID4VP URI
    if (qrContent.startsWith("openid4vp://") ||
        qrContent.startsWith("eudi-openid4vp://")) {
        handlePresentationRequest(qrContent)
    }
}
```

### Deep Link Handling

**Location:** `ui-logic/src/main/java/eu/europa/ec/uilogic/navigation/helper/DeepLinkHelper.kt`

```kotlin
fun handleDeepLinkAction(
    navController: NavController,
    action: DeepLinkAction,
    arguments: String?
) {
    when (action.type) {
        DeepLinkType.OPENID4VP -> {
            val screen = PresentationScreens.PresentationRequest
            navController.navigate(screen.routeWithArgs(arguments))
        }
    }
}
```

### Request URI Config

**Location:** `common-feature/src/main/java/eu/europa/ec/commonfeature/config/RequestUriConfig.kt`

```kotlin
data class RequestUriConfig(
    val uri: String,
    val initiatorRoute: String
)

sealed interface PresentationMode {
    data class OpenId4Vp(val uri: String, val initiatorRoute: String) : PresentationMode
    data class Ble(val initiatorRoute: String) : PresentationMode
}
```

### Presentation Controller Initialization

**Location:** `core-logic/src/main/java/eu/europa/ec/corelogic/controller/WalletCorePresentationController.kt`

```kotlin
override fun setConfig(config: PresentationControllerConfig) {
    _config = config
}

private fun addListener(listener: EudiWalletListenerWrapper) {
    val config = requireInit { _config }
    eudiWallet.addTransferEventListener(listener)

    if (config is PresentationControllerConfig.OpenId4VP) {
        eudiWallet.startRemotePresentation(config.uri.toUri())
    }
}
```

---

## EUDI iOS

### Deep Link Processing

**Location:** `Sources/Application/Application.swift`

```swift
@main
struct Application: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
                .onOpenURL { url in
                    deepLinkController.cacheDeepLink(url: url)
                }
        }
    }
}
```

### Deep Link Controller

**Location:** `Modules/logic-ui/Sources/Controller/DeepLinkController.swift`

```swift
func handleDeepLinkAction(
    routerHost: RouterHost,
    remoteSessionCoordinator: RemoteSessionCoordinator?
) {
    guard let action = getCachedDeepLinkAction() else { return }
    clearCachedDeepLinkAction()

    switch action.type {
    case .openid4vp, .haip_vp:
        guard let coordinator = remoteSessionCoordinator else {
            fatalError("OpenId4VP requires RemoteSessionCoordinator")
        }
        routerHost.push(
            with: .featurePresentationModule(
                .presentationRequest(
                    presentationCoordinator: coordinator,
                    originator: .featureDashboardModule(.dashboard)
                )
            )
        )
    }
}
```

### Session Initialization

**Location:** `Modules/logic-core/Sources/Controller/WalletKitController.swift`

```swift
func startSameDevicePresentation(deepLink: URLComponents) async -> RemoteSessionCoordinator {
    let session = walletKit.openId4VpSession(url: deepLink.url!)
    let coordinator = RemoteSessionCoordinatorImpl(session: session)
    await coordinator.initialize()
    return coordinator
}

func startCrossDevicePresentation(urlString: String) async -> RemoteSessionCoordinator {
    let session = walletKit.openId4VpSession(url: URL(string: urlString)!)
    let coordinator = RemoteSessionCoordinatorImpl(session: session)
    await coordinator.initialize()
    return coordinator
}
```

---

## Procivis ONE

### QR Scanner Screen

**Location:** `app/screens/dashboard/qr-code-scanner-screen.tsx`

```typescript
const handleCodeScan = useCallback(
    (scannedCode: Code[]) => {
        if (!code) {
            setCode(scannedCode[0].value);
        }
    },
    [code, setCode],
);

useEffect(() => {
    if (code) {
        navigation.goBack();
        handleInvitationUrl(code);
    }
}, [code, navigation, handleInvitationUrl]);
```

### Invitation URL Handling

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

### Invitation Processing

**Location:** `app/screens/credential/invitation-process-screen.tsx`

```typescript
const handleInvitationUrl = useCallback(async (url: string) => {
    // Follow HTTP redirects if URL starts with http
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
        case HandleInvitationResponseBindingEnum.CREDENTIAL_ISSUANCE:
            // Handle issuance...
            break;
    }
}, [invitationHandler, navigation]);
```

---

## Comparison Summary

| Aspect | EUDI Android | EUDI iOS | Procivis ONE |
|--------|--------------|----------|--------------|
| **QR scanning** | Camera API | Camera API | Vision Camera |
| **Deep link entry** | Intent filter | onOpenURL | Linking API |
| **URI parsing** | DeepLinkHelper | DeepLinkController | parseUniversalLink |
| **Session init** | SDK method | Coordinator | Core handler |
| **State flow** | ViewModel | SwiftUI state | React state |

---

## Key Code References

| Implementation | File | Function |
|----------------|------|----------|
| EUDI Android | `DeepLinkHelper.kt` | `handleDeepLinkAction()` |
| EUDI Android | `WalletCorePresentationController.kt` | `setConfig()` |
| EUDI iOS | `DeepLinkController.swift` | `handleDeepLinkAction()` |
| EUDI iOS | `WalletKitController.swift` | `startSameDevicePresentation()` |
| Procivis | `deep-link.ts` | `useInvitationHandling()` |
| Procivis | `invitation-process-screen.tsx` | `handleInvitationUrl()` |
