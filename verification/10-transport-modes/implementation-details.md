# Transport Modes — Implementation Details

This document details how EUDI (Android/iOS) and Procivis ONE implement different transport modes for presentation flows.

---

## EUDI Android

### Deep Link Handling

**Location:** `ui-logic/src/main/java/eu/europa/ec/uilogic/navigation/helper/DeepLinkHelper.kt`

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

### Manifest Configuration

**Location:** `build-logic/convention/src/main/kotlin/AndroidLibraryConventionPlugin.kt`

```kotlin
manifestPlaceholders.putAll(
    mapOf(
        "openid4vpScheme" to "openid4vp",
        "eudiOpenid4vpScheme" to "eudi-openid4vp",
        "mdocOpenid4vpScheme" to "mdoc-openid4vp",
        "haipOpenid4vpScheme" to "haip"
    )
)
```

### Presentation Mode Configuration

**Location:** `common-feature/src/main/java/eu/europa/ec/commonfeature/config/RequestUriConfig.kt`

```kotlin
sealed interface PresentationMode {
    data class OpenId4Vp(val uri: String, val initiatorRoute: String) : PresentationMode
    data class Ble(val initiatorRoute: String) : PresentationMode
}
```

### Remote Presentation (OID4VP)

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

### Proximity Presentation (BLE)

```kotlin
if (config is PresentationControllerConfig.Ble) {
    eudiWallet.startProximityPresentation()
}
```

### QR Engagement Generation

```kotlin
override suspend fun startQrEngagement(): QrEngagementPartialState {
    return try {
        eudiWallet.startQrEngagement()
        QrEngagementPartialState.Success
    } catch (e: Exception) {
        QrEngagementPartialState.Failure(e.localizedMessage ?: genericErrorMessage)
    }
}
```

### NFC Engagement Toggle

```kotlin
override fun toggleNfcEngagement(toggle: Boolean) {
    eudiWallet.enableNFCEngagement(toggle)
}
```

### Redirect Handling

```kotlin
onRedirect = { uri ->
    redirectUri = uri
    trySendBlocking(TransferEventPartialState.Redirect(uri = uri))
}
```

---

## EUDI iOS

### Deep Link Entry

**Location:** `Sources/Application/Application.swift`

```swift
.onOpenURL { url in
    deepLinkController.cacheDeepLink(url: url)
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

### Remote Session Coordinator

**Location:** `Modules/logic-core/Sources/Coordinator/RemoteSessionCoordinator.swift`

```swift
public protocol RemoteSessionCoordinator: Sendable {
    func initialize() async
    func requestReceived() async throws -> PresentationRequest
    func sendResponse(response: RequestItemConvertible) async
    func getStream() -> AsyncStream<PresentationState>
}
```

### Proximity Session Coordinator

**Location:** `Modules/logic-core/Sources/Coordinator/ProximitySessionCoordinator.swift`

```swift
public protocol ProximitySessionCoordinator: Sendable {
    func initialize() async throws
    func startQrEngagement() async throws -> UIImage
    func requestReceived() async throws -> PresentationRequest
    func sendResponse(response: RequestItemConvertible) async
}
```

### WalletKit Controller

**Location:** `Modules/logic-core/Sources/Controller/WalletKitController.swift`

```swift
// Same-device OID4VP
func startSameDevicePresentation(deepLink: URLComponents) async -> RemoteSessionCoordinator

// Cross-device OID4VP
func startCrossDevicePresentation(urlString: String) async -> RemoteSessionCoordinator

// BLE Proximity
func startProximityPresentation() async -> ProximitySessionCoordinator
```

### Presentation State

**Location:** `Modules/logic-core/Sources/Coordinator/Model/PresentationState.swift`

```swift
public enum PresentationState: Sendable {
    case loading
    case prepareQr                    // Proximity: preparing QR
    case qrReady(imageData: Data)     // Proximity: QR ready for display
    case requestReceived(PresentationRequest)
    case responseToSend(RequestItemConvertible)
    case responseSent(URL?)           // URL for redirect if any
    case error(Error)
}
```

---

## Procivis ONE

### Transport Configuration

**Location:** `app/navigators/app-navigator.tsx`

```typescript
const coreConfig = {
    transport: {
        BLE: {
            enabled: config.featureFlags.bleEnabled,
        },
        HTTP: {
            enabled: config.featureFlags.httpTransportEnabled,
        },
        MQTT: {
            enabled: config.featureFlags.mqttTransportEnabled,
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

### Universal Link Parsing

```typescript
export const parseUniversalLink = (url: string): string | undefined => {
    // Convert iOS universal links to standard format
    const parsed = new URL(url);
    if (parsed.host === 'wallet.example.com') {
        const path = parsed.pathname;
        // Extract actual invitation URL from path
        return decodeURIComponent(path.slice(1));
    }
    return undefined;
};
```

### HTTP Redirect Following

**Location:** `app/screens/credential/invitation-process-screen.tsx`

```typescript
const handleInvitationUrl = useCallback(async (url: string) => {
    // Follow HTTP redirects
    const targetUrl = url.startsWith('http')
        ? await RNBlobUtil.fetch('GET', url, {})
            .then(resp => resp.redirects[resp.redirects.length - 1] || url)
        : url;

    const invitationResponse = await invitationHandler(targetUrl);
    // ...
}, [invitationHandler]);
```

### QR Code Scanner

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

### Transport Availability Check

```typescript
const isBleEnabled = useMemo(
    () => config && getEnabledTransports(config).includes(Transport.Bluetooth),
    [config],
);

const isNfcEnabled = useMemo(
    () => config && config.featureFlags.nfcEnabled,
    [config],
);
```

---

## Comparison Table

| Feature | EUDI Android | EUDI iOS | Procivis ONE |
|---------|--------------|----------|--------------|
| **OID4VP schemes** | 4 schemes | 2 schemes | Universal links |
| **BLE proximity** | ✓ | ✓ | ✓ |
| **NFC engagement** | ✓ | ✓ | ✓ |
| **HTTP transport** | SDK | SDK | Configurable |
| **MQTT transport** | ✗ | ✗ | ✓ |
| **Redirect handling** | SDK + manual | SDK + manual | Manual |
| **QR generation** | SDK | SDK | Core library |

---

## Key Code References

| Implementation | File | Purpose |
|----------------|------|---------|
| EUDI Android | `DeepLinkHelper.kt` | Scheme handling |
| EUDI Android | `WalletCorePresentationController.kt` | Transport control |
| EUDI iOS | `DeepLinkController.swift` | Deep link routing |
| EUDI iOS | `ProximitySessionCoordinator.swift` | BLE management |
| Procivis | `deep-link.ts` | URL handling |
| Procivis | `app-navigator.tsx` | Transport config |
| Procivis | `qr-code-scanner-screen.tsx` | QR scanning |
