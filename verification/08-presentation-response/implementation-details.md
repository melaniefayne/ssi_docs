# Presentation Response — Implementation Details

This document details how EUDI (Android/iOS) and Procivis ONE construct and send presentation responses.

---

## EUDI Android

### Response Generation

**Location:** `core-logic/src/main/java/eu/europa/ec/corelogic/controller/WalletCorePresentationController.kt`

```kotlin
override fun sendRequestedDocuments(): SendRequestedDocumentsPartialState {
    return disclosedDocuments?.let { safeDisclosedDocuments ->
        var result: SendRequestedDocumentsPartialState =
            SendRequestedDocumentsPartialState.RequestSent

        processedRequest?.generateResponse(
            DisclosedDocuments(safeDisclosedDocuments.toList())
        )?.toKotlinResult()
            ?.onFailure {
                result = SendRequestedDocumentsPartialState.Failure(
                    error = it.localizedMessage ?: genericErrorMessage
                )
            }
            ?.onSuccess {
                eudiWallet.sendResponse(it.response)
                result = SendRequestedDocumentsPartialState.RequestSent
            }

        result
    } ?: SendRequestedDocumentsPartialState.Failure(error = genericErrorMessage)
}
```

### Disclosed Document Structure

```kotlin
data class DisclosedDocument(
    val documentId: String,
    val disclosedItems: List<DisclosedItem>,
    val keyUnlockData: KeyUnlockData?
)

sealed interface DisclosedItem {
    data class MsoMdocItem(
        val namespace: String,
        val elementIdentifier: String
    ) : DisclosedItem

    data class SdJwtVcItem(
        val path: List<String>
    ) : DisclosedItem
}
```

### Redirect Handling

```kotlin
onRedirect = { uri ->
    redirectUri = uri
    trySendBlocking(TransferEventPartialState.Redirect(uri = uri))
}
```

**Location:** `presentation-feature/src/main/java/eu/europa/ec/presentationfeature/ui/success/PresentationSuccessViewModel.kt`

```kotlin
override fun getNextScreenConfigNavigation(): ConfigNavigation {
    val redirectUri = interactor.redirectUri

    val deepLinkWithUriOrPopToDashboard = ConfigNavigation(
        navigationType = redirectUri?.let {
            NavigationType.Deeplink(it.toString(), interactor.initiatorRoute)
        } ?: NavigationType.PopTo(DashboardScreens.Dashboard)
    )

    return deepLinkWithUriOrPopToDashboard
}
```

---

## EUDI iOS

### Response Sending

**Location:** `Modules/logic-core/Sources/Coordinator/RemoteSessionCoordinator.swift`

```swift
public func sendResponse(response: RequestItemConvertible) async {
    await session.sendResponse(
        userAccepted: true,
        itemsToSend: response.items,
        onCancel: nil
    ) { url in
        self.sendableCurrentValueSubject.setValue(.responseSent(url))
    }
}
```

### Request Items Structure

**Location:** `Modules/logic-core/Sources/Model/RequestItemConvertible.swift`

```swift
public typealias RequestConvertibleItems = [String: [String: [RequestItem]]]

public protocol RequestItemConvertible: Sendable {
    var items: RequestConvertibleItems { get }
}

public struct RequestItem: Sendable, Equatable {
    public let elementPath: [String]
}
```

### Response State

```swift
public enum PresentationState: Sendable {
    case loading
    case requestReceived(PresentationRequest)
    case responseToSend(RequestItemConvertible)
    case responseSent(URL?)  // URL for redirect
    case error(Error)
}
```

### Success Handling

**Location:** `Modules/feature-presentation/Sources/Interactor/PresentationInteractor.swift`

```swift
public func onSendResponse() async -> RemoteSentResponsePartialState {
    guard
        let state = try? await sessionCoordinatorHolder
            .getActiveRemoteCoordinator().getState(),
        case PresentationState.responseToSend(let responseItem) = state
    else {
        return .failure(PresentationSessionError.invalidState)
    }

    do {
        try await self.sessionCoordinatorHolder
            .getActiveRemoteCoordinator()
            .sendResponse(response: responseItem)
        return .sent
    } catch {
        return .failure(error)
    }
}
```

---

## Procivis ONE

### Proof Submission V1

**Location:** `app/screens/credential/proof-process-screen.tsx`

```typescript
const { mutateAsync: acceptProof } = useMutation(
    async ({
        interactionId,
        credentials,
    }: {
        credentials: Record<string, PresentationSubmitCredentialRequestBindingDto[]>;
        interactionId: string;
    }) => core.holderSubmitProof(interactionId, credentials),
);
```

### Credential Selection Format

```typescript
type PresentationSubmitCredentialRequestBindingDto = {
    credentialId: string;
    submitClaims: string[];  // Field IDs to disclose
};
```

### Proof Submission V2

```typescript
const { mutateAsync: acceptProofV2 } = useMutation(
    async ({
        interactionId,
        credentials,
    }: {
        credentials: CredentialQuerySelection;
        interactionId: string;
    }) => core.holderSubmitProofV2(interactionId, credentials),
);
```

### V2 Selection Format

```typescript
type CredentialQuerySelection = Record<string, Array<{
    credentialId: string;
    userSelections: string[];  // JSON paths
}>>;
```

### Redirect Handling

```typescript
const redirectUri = proof?.redirectUri;

const redirectButtonHandler = useCallback(() => {
    if (!redirectUri) {
        return;
    }
    Linking.openURL(redirectUri)
        .then(closeButtonHandler)
        .catch((e) => {
            reportException(e, "Couldn't open redirect URI");
        });
}, [closeButtonHandler, redirectUri]);
```

---

## Comparison Summary

| Aspect | EUDI Android | EUDI iOS | Procivis ONE |
|--------|--------------|----------|--------------|
| **Response generation** | SDK | SDK | Core library |
| **Selection format** | DisclosedItem | RequestItem | Credential queries |
| **Submission** | SDK sendResponse | Session sendResponse | Core submit |
| **Redirect** | URI navigation | State callback | Linking.openURL |

---

## Key Code References

| Implementation | File | Function |
|----------------|------|----------|
| EUDI Android | `WalletCorePresentationController.kt` | `sendRequestedDocuments()` |
| EUDI Android | `PresentationSuccessViewModel.kt` | `getNextScreenConfigNavigation()` |
| EUDI iOS | `RemoteSessionCoordinator.swift` | `sendResponse()` |
| EUDI iOS | `PresentationInteractor.swift` | `onSendResponse()` |
| Procivis | `proof-process-screen.tsx` | `acceptProof` / `acceptProofV2` |
