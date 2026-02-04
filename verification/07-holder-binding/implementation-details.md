# Holder Binding — Implementation Details

This document details how EUDI (Android/iOS) and Procivis ONE implement holder binding proofs during presentation.

---

## EUDI Android

### Key Unlock Flow

**Location:** `core-logic/src/main/java/eu/europa/ec/corelogic/controller/WalletCorePresentationController.kt`

```kotlin
override fun checkForKeyUnlock() = flow {
    disclosedDocuments?.let { documents ->
        val authenticationData = mutableListOf<AuthenticationData>()

        if (eudiWallet.config.userAuthenticationRequired) {
            val keyUnlockDataMap = documents.associateWith { disclosedDocument ->
                eudiWallet.getDefaultKeyUnlockData(
                    documentId = disclosedDocument.documentId
                )
            }

            for ((doc, kud) in keyUnlockDataMap) {
                val cryptoObject = kud?.getCryptoObjectForSigning()
                authenticationData.add(
                    AuthenticationData(
                        crypto = BiometricCrypto(cryptoObject),
                        onAuthenticationSuccess = {
                            disclosedDocuments?.addOrReplace(
                                value = doc.copy(keyUnlockData = kud),
                                replaceCondition = { it.documentId == doc.documentId }
                            )
                        }
                    )
                )
            }

            emit(CheckKeyUnlockPartialState.UserAuthenticationRequired(authenticationData))
        } else {
            emit(CheckKeyUnlockPartialState.RequestIsReadyToBeSent)
        }
    }
}
```

### Biometric Authentication

**Location:** `presentation-feature/src/main/java/eu/europa/ec/presentationfeature/ui/loading/PresentationLoadingViewModel.kt`

```kotlin
private fun checkForKeyUnlock() {
    viewModelScope.launch {
        presentationLoadingInteractor.checkForKeyUnlock()
            .collect { response ->
                when (response) {
                    is CheckKeyUnlockPartialState.UserAuthenticationRequired -> {
                        setState {
                            copy(
                                authenticationData = response.authenticationData
                            )
                        }
                    }
                    is CheckKeyUnlockPartialState.RequestIsReadyToBeSent -> {
                        sendRequestedDocuments()
                    }
                }
            }
    }
}
```

### Response Generation with Proof

```kotlin
override fun sendRequestedDocuments(): SendRequestedDocumentsPartialState {
    return disclosedDocuments?.let { safeDisclosedDocuments ->
        var result: SendRequestedDocumentsPartialState =
            SendRequestedDocumentsPartialState.RequestSent

        // Generate response with holder proof
        processedRequest?.generateResponse(
            DisclosedDocuments(safeDisclosedDocuments.toList())
        )?.toKotlinResult()
            ?.onFailure {
                result = SendRequestedDocumentsPartialState.Failure(
                    error = it.localizedMessage ?: genericErrorMessage
                )
            }
            ?.onSuccess {
                // SDK generates VP with key binding proof
                eudiWallet.sendResponse(it.response)
                result = SendRequestedDocumentsPartialState.RequestSent
            }
        result
    } ?: SendRequestedDocumentsPartialState.Failure(error = genericErrorMessage)
}
```

---

## EUDI iOS

### Authentication Flow

**Location:** `Modules/feature-presentation/Sources/UI/Presentation/Request/PresentationRequestViewModel.swift`

```swift
override func getSuccessRoute() -> AppRoute? {
    guard let coordinator = self.coordinator else { return nil }

    return .featureCommonModule(
        .biometry(
            config: UIConfig.Biometry(
                navigationTitle: .biometryConfirmRequest,
                caption: .requestDataShareBiometryCaption,
                quickPinOnlyCaption: .requestDataShareQuickPinCaption,
                navigationSuccessType: .push(
                    .featurePresentationModule(
                        .presentationLoader(
                            relyingParty: getRelyingParty().toString,
                            relyingPartyisTrusted: getRelyingPartyIsTrusted(),
                            presentationCoordinator: coordinator,
                            originator: getOriginator(),
                            items: viewState.items.filterSelectedRows()
                        )
                    )
                ),
                navigationBackType: .pop,
                isPreAuthorization: false,
                shouldInitializeBiometricOnCreate: true
            )
        )
    )
}
```

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

### Request Items Preparation

**Location:** `Modules/feature-common/Sources/UI/Request/Model/RequestDataUIModel.swift`

```swift
public extension Array where Element == RequestDataUiModel {
    func prepareRequest() -> RequestItemsWrapper {
        let models = self.filter { $0.section.isSelected }

        let requestConvertible = models
            .reduce(into: [PresentationExpandableListItem]()) { partialResult, document in
                partialResult.append(contentsOf: document.section.listItems)
            }
            .reduce(into: RequestItemsWrapper()) { partialResult, claim in
                var path: [String] {
                    switch claim.domainModel?.type {
                    case .mdoc:
                        guard let path = claim.domainModel?.path, path.count > 1 else {
                            return []
                        }
                        return [path[1]]
                    case .sdjwt:
                        guard let path = claim.domainModel?.path else {
                            return []
                        }
                        return path
                    default:
                        return []
                    }
                }

                let requestItem: RequestItem = .init(elementPath: path)
                // Build structured request items...
            }

        return requestConvertible
    }
}
```

---

## Procivis ONE

### Proof Submission V1

**Location:** `app/screens/credential/proof-process-screen.tsx`

```typescript
const { mutateAsync: acceptProof } = useProofAccept();

const handleProofSubmit = useCallback(async () => {
    if (accepted.current) {
        return;
    }
    try {
        accepted.current = true;

        if ('credentials' in params) {
            const credentials = params.credentials;
            await acceptProof({
                credentials,
                interactionId,
            });
        }

        setState(LoaderViewState.Success);
    } catch (e) {
        if (isRSELockedError(e)) {
            setState(LoaderViewState.Error);
        } else {
            setState(LoaderViewState.Warning);
        }
        setError(e);
    }
}, [params, acceptProof, interactionId]);
```

### Proof Submission V2

```typescript
const { mutateAsync: acceptProofV2 } = useProofAcceptV2();

// V2 submission with credential queries
if ('credentialsV2' in params) {
    const credentials = params.credentialsV2;
    await acceptProofV2({
        credentials,
        interactionId,
    });
}
```

### Core Hook for V2

```typescript
const useProofAcceptV2 = () => {
    const queryClient = useQueryClient();
    const { core } = useONECore();

    return useMutation(
        async ({
            interactionId,
            credentials,
        }: {
            credentials: CredentialQuerySelection;
            interactionId: string;
        }) => core.holderSubmitProofV2(interactionId, credentials),
        {
            onSuccess: async () => {
                await queryClient.invalidateQueries(PROOF_DETAIL_QUERY_KEY);
                await queryClient.invalidateQueries(HISTORY_LIST_QUERY_KEY);
            },
        },
    );
};
```

### RSE (Remote Secure Element) Handling

```typescript
if (isRSELockedError(e)) {
    // RSE requires PIN unlock for signing
    setState(LoaderViewState.Error);
    // Prompt user for RSE PIN
}
```

---

## Comparison Summary

| Aspect | EUDI Android | EUDI iOS | Procivis ONE |
|--------|--------------|----------|--------------|
| **Authentication** | Biometric prompt | LAContext | RSE PIN |
| **Key access** | KeyUnlockData | SDK handles | Core handles |
| **Proof generation** | SDK | SDK | Core library |
| **Error handling** | State machine | Async throws | Mutation errors |

---

## Key Code References

| Implementation | File | Function |
|----------------|------|----------|
| EUDI Android | `WalletCorePresentationController.kt` | `checkForKeyUnlock()` |
| EUDI Android | `PresentationLoadingViewModel.kt` | Biometric flow |
| EUDI iOS | `PresentationRequestViewModel.swift` | `getSuccessRoute()` |
| EUDI iOS | `RemoteSessionCoordinator.swift` | `sendResponse()` |
| Procivis | `proof-process-screen.tsx` | `handleProofSubmit()` |
| Procivis | Custom hook | `useProofAcceptV2()` |
