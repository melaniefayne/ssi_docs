# Credential Selection — Implementation Details

This document details how EUDI (Android/iOS) and Procivis ONE implement credential matching, user selection, and selective disclosure.

---

## EUDI Android

### Request Transformation

**Location:** `common-feature/src/main/java/eu/europa/ec/commonfeature/ui/request/transformer/RequestTransformer.kt`

The transformer matches requested items to stored credentials:

```kotlin
fun transformToDomainItems(
    storageDocuments: List<IssuedDocument>,
    resourceProvider: ResourceProvider,
    uuidProvider: UuidProvider,
    requestDocuments: List<RequestedDocument>,
): Result<List<DocumentPayloadDomain>> = runCatching {
    requestDocuments.forEach { requestDocument ->
        // Find matching storage document
        val storageDocument =
            storageDocuments.first { it.id == requestDocument.documentId }

        // Get requested claim paths
        val requestedItemsPaths = requestDocument.requestedItems.keys
            .map { it.toClaimPath() }

        // Filter available claims by prefix matching
        val filteredPaths = claimsPaths.filter { available ->
            requestedItemsPaths.any { requested ->
                requested.isPrefixOf(available)
            }
        }
        // ...
    }
}
```

### Claim Path Handling

**Location:** `core-logic/src/main/java/eu/europa/ec/corelogic/model/ClaimPathDomain.kt`

```kotlin
data class ClaimPathDomain(val path: List<String>) {
    fun isPrefixOf(other: ClaimPathDomain): Boolean {
        if (path.size > other.path.size) return false
        return path.zip(other.path).all { (a, b) -> a == b }
    }
}
```

### Format-Specific Conversion

**Location:** `core-logic/src/main/java/eu/europa/ec/corelogic/extension/DocumentClaimExtensions.kt`

**SD-JWT:**
```kotlin
is SdJwtVcClaim -> {
    val currentPath: List<String> = parentPath + this.identifier
    if (children.isEmpty() || this.value is Collection<*>) {
        listOf(ClaimPathDomain(currentPath))
    } else {
        children.flatMap { child ->
            child.toClaimPaths(currentPath)  // Recursive for nested
        }
    }
}
```

**mDoc:**
```kotlin
is MsoMdocClaim -> {
    listOf(ClaimPathDomain(listOf(this.identifier)))
}
```

### User Selection UI Model

**Location:** `common-feature/src/main/java/eu/europa/ec/commonfeature/ui/request/model/RequestDocumentItemUi.kt`

```kotlin
data class RequestDocumentItemUi(
    val header: RequestDocumentHeaderUi,
    val sections: List<RequestDocumentSectionUi>,
    val isSelected: Boolean,
    val isExpanded: Boolean
)

data class RequestDocumentSectionUi(
    val claims: List<RequestDocumentClaimUi>
)

data class RequestDocumentClaimUi(
    val id: String,
    val name: String,
    val value: String,
    val isRequired: Boolean,
    val isSelected: Boolean
)
```

### Revocation Filtering

```kotlin
.filterNot {
    walletCoreDocumentsController.isDocumentRevoked(it.docId)
}
```

### Disclosed Document Construction

```kotlin
fun createDisclosedDocuments(items: List<RequestDocumentItemUi>): DisclosedDocuments {
    val mDocItems: MutableList<MsoMdocItem> = mutableListOf()
    val sdJwtItems: MutableList<SdJwtVcItem> = mutableListOf()

    selectedItemsForDocument.forEach { selectedItem ->
        when (documentPayload.domainDocFormat) {
            is DomainDocumentFormat.SdJwtVc -> sdJwtItems.add(
                SdJwtVcItem(
                    path = ClaimPathDomain.toSdJwtVcPath(selectedItemId)
                )
            )
            is DomainDocumentFormat.MsoMdoc -> mDocItems.add(
                MsoMdocItem(
                    namespace = documentPayload.domainDocFormat.namespace,
                    elementIdentifier = ClaimPathDomain.toElementIdentifier(selectedItemId)
                )
            )
        }
    }

    return DisclosedDocument(
        documentId = documentPayload.docId,
        disclosedItems = mDocItems.distinctBy { it.elementIdentifier } + sdJwtItems,
        keyUnlockData = null
    )
}
```

---

## EUDI iOS

### Request Model Conversion

**Location:** `Modules/feature-common/Sources/UI/Request/Model/RequestDataUIModel.swift`

```swift
public extension Array where Element == DocElements {
    func toUiModels(with walletKitController: WalletKitController) -> [RequestDataUiModel] {
        self.compactMap { element in

            var title: String {
                return switch element {
                case .msoMdoc(let msoMdocElements):
                    msoMdocElements.displayName.ifNilOrEmpty { msoMdocElements.docType }
                case .sdJwt(let sdJwtElements):
                    sdJwtElements.displayName.ifNilOrEmpty { sdJwtElements.vct }
                }
            }

            var type: DocumentElementType {
                return switch element {
                case .msoMdoc:
                    .mdoc
                case .sdJwt:
                    .sdjwt
                }
            }
            // ...
        }
    }
}
```

### Selective Disclosure Field Extraction

```swift
private extension Array where Element == DocClaim {
    func selectiveDisclosableFields(
        id: String,
        type: DocumentElementType,
        walletKitController: WalletKitController
    ) -> [DocumentElementClaim] {
        self.reduce(into: [DocumentElementClaim]()) { partialResult, claim in
            partialResult.append(
                contentsOf: walletKitController.parseDocClaim(
                    docId: id,
                    groupId: UUID().uuidString,
                    docClaim: claim,
                    type: type,
                    parser: { /* date formatting */ }
                )
            )
        }.sortByName()
    }
}
```

### Request Item Preparation

**Location:** `Modules/feature-common/Sources/UI/Request/Model/RequestDataUIModel.swift`

```swift
public extension Array where Element == RequestDataUiModel {
    func prepareRequest() -> RequestItemsWrapper {
        // Filter to selected items only
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
                        return [path[1]]  // Extract element identifier
                    case .sdjwt:
                        guard let path = claim.domainModel?.path else {
                            return []
                        }
                        return path  // Full JSON path
                    default:
                        return []
                    }
                }

                let requestItem: RequestItem = .init(elementPath: path)
                var nameSpaceDict = partialResult.items[documentId, default: [:]]
                nameSpaceDict[nameSpace, default: []].appendIfNotExists(requestItem)
                partialResult.items[documentId] = nameSpaceDict
            }

        return requestConvertible
    }
}
```

### Revocation Filtering

**Location:** `Modules/feature-presentation/Sources/Interactor/PresentationInteractor.swift`

```swift
public func onRequestReceived() async -> Result<OnlineAuthenticationRequestSuccessModel, Error> {
    do {
        let response = try await sessionCoordinatorHolder
            .getActiveRemoteCoordinator()
            .requestReceived()

        // Filter out revoked credentials
        let revokedDocuments = (try? await walletKitController.fetchRevokedDocuments()) ?? []
        let documents = response.items.filter { item in
            !revokedDocuments.contains(where: { $0 == item.docId })
        }

        guard !documents.isEmpty else {
            return .failure(WalletCoreError.unableFetchDocuments)
        }

        return .success(OnlineAuthenticationRequestSuccessModel(
            relyingParty: response.relyingParty,
            items: documents.toUiModels(with: walletKitController),
            isTrusted: response.isTrusted
        ))
    } catch {
        return .failure(error)
    }
}
```

---

## Procivis ONE

### Presentation Definition V1 Processing

**Location:** `app/components/proof-request/proof-presentation-v1.tsx`

```typescript
const presentationDefinition = useMemoAsync(async () => {
    const definition = await core.getPresentationDefinition(proofId);
    onPresentationDefinitionLoaded();

    // Refresh revocation status of applicable credentials
    const credentialIds = new Set<string>(
        definition.requestGroups.flatMap(({ requestedCredentials }) =>
            requestedCredentials.flatMap(
                ({ applicableCredentials }) => applicableCredentials,
            ),
        ),
    );
    await checkRevocation(Array.from(credentialIds));
    return definition;
}, [checkRevocation, core, proofId]);
```

### Pre-selection Logic

**Location:** `app/utils/proof-request.ts`

```typescript
const pickPreselectedCredential = (
    credentialRequest: PresentationDefinitionRequestedCredentialBindingDto,
    allCredentials: CredentialDetailBindingDto[],
): CredentialDetailBindingDto['id'] | undefined => {
    // First priority: ACCEPTED + APPLICABLE
    const applicableValidCredential = allCredentials.find(
        ({ id, state }) =>
            state === CredentialStateBindingEnum.ACCEPTED &&
            credentialRequest.applicableCredentials.includes(id),
    );
    if (applicableValidCredential) {
        return applicableValidCredential.id;
    }

    // Second priority: ACCEPTED + INAPPLICABLE
    const inapplicableValidCredential = allCredentials.find(
        ({ id, state }) =>
            state === CredentialStateBindingEnum.ACCEPTED &&
            credentialRequest.inapplicableCredentials.includes(id),
    );
    if (inapplicableValidCredential) {
        return inapplicableValidCredential.id;
    }

    // Fallback: any applicable or inapplicable credential
    return (
        credentialRequest.applicableCredentials[0] ??
        credentialRequest.inapplicableCredentials[0]
    );
};
```

### Field Selection (V1)

```typescript
const onSelectField = useCallback(
    (requestCredentialId: string) =>
    (fieldId: string, selected: boolean) => {
        setSelectedCredentials((current) => {
            return selectedCredentialsWithUpdatedSelection(
                current,
                requestCredentialId,
                fieldId,
                selected,
            );
        });
    },
    [setSelectedCredentials],
);

const selectedCredentialsWithUpdatedSelection = (
    currentSelectedCredentials: Record<string, PresentationSubmitCredentialRequestBindingDto | undefined>,
    updatedCredentialId: string,
    updatedFieldId: string,
    selected: boolean,
) => {
    const prevSelection = currentSelectedCredentials[updatedCredentialId];
    let submitClaims = [...prevSelection.submitClaims];

    if (selected) {
        submitClaims.push(updatedFieldId);
    } else {
        submitClaims = submitClaims.filter(claimId => claimId !== updatedFieldId);
    }

    return {
        ...currentSelectedCredentials,
        [updatedCredentialId]: { ...prevSelection, submitClaims },
    };
};
```

### Presentation Definition V2 Processing

**Location:** `app/components/proof-request/proof-presentation-v2.tsx`

```typescript
const presentationDefinition = useMemoAsync(async () => {
    const definition = await core.getPresentationDefinitionV2(proofId);

    const credentialIds = new Set<string>(
        Object.values(definition.credentialQueries)
            .flatMap((query) => {
                if (query.credentialOrFailureHint.type_ !== 'APPLICABLE_CREDENTIALS') {
                    return;
                }
                return query.credentialOrFailureHint.applicableCredentials.flatMap(
                    ({ id }) => id,
                );
            })
            .filter(nonEmptyFilter),
    );
    await checkRevocation(Array.from(credentialIds));
    return definition;
}, [checkRevocation, core, proofId]);
```

### V2 Pre-selection

**Location:** `app/utils/credential-sharing-v2.tsx`

```typescript
export const preselectCredentialsForPresentationDefinitionV2 = (
    presentationDefinition: PresentationDefinitionV2ResponseBindingDto,
) => {
    const preselected = presentationDefinition.credentialSets.reduce<SetCredentialQuerySelection>(
        (acc, set, index) => {
            if (set.required) {
                acc[index] = set.options[0]?.reduce<CredentialQuerySelection>(
                    (acc2, queryId) => {
                        const credentialQuery = presentationDefinition.credentialQueries[queryId];
                        if (
                            credentialQuery &&
                            credentialQuery.credentialOrFailureHint.type_ === 'APPLICABLE_CREDENTIALS'
                        ) {
                            // Sort by issuance date (newest first)
                            credentialQuery.credentialOrFailureHint.applicableCredentials.sort(
                                objectByTimestampSorter('issuanceDate', false),
                            );
                            acc2[queryId] = [{
                                credentialId: credentialQuery.credentialOrFailureHint.applicableCredentials[0].id,
                                userSelections: [],
                            }];
                        }
                        return acc2;
                    },
                    {},
                );
            }
            return acc;
        },
        {},
    );
    return preselected;
};
```

### V2 Field Selection with Paths

```typescript
const onSelectField =
    (setId: string) =>
    (requestQueryId: string) =>
    (
        credentialId: string,
        fieldPath: string,  // JSON path
        selected: boolean,
    ) => {
        setSelectedCredentials((current) =>
            selectedCredentialsWithUpdatedSelection(
                current,
                setId,
                requestQueryId,
                credentialId,
                fieldPath,
                selected,
            ),
        );
    };
```

### Multiple Credential Selection (V2)

**Location:** `app/screens/credential/proof-request-select-credential-v2-screen.tsx`

```typescript
const onCredentialSelected = useCallback(
    (credentialId: string, selected: boolean) => {
        setSelectedCredentialIds((prevSelection) => {
            let newSelection = [...prevSelection];
            if (credentialQuery.multiple) {
                // Multi-select mode
                if (selected) {
                    newSelection = uniq(newSelection.concat(credentialId));
                } else {
                    newSelection = newSelection.filter((id) => id !== credentialId);
                }
            } else if (selected) {
                // Single-select mode
                newSelection = [credentialId];
            }
            return newSelection;
        });
    },
    [credentialQuery],
);
```

---

## Comparison Summary

| Feature | EUDI Android | EUDI iOS | Procivis ONE |
|---------|--------------|----------|--------------|
| **Matching algorithm** | SDK + transformer | SDK + converter | Core library |
| **Path system** | ClaimPathDomain | Array of strings | Field IDs / JSON paths |
| **Pre-selection** | SDK handles | SDK handles | Custom logic |
| **Multi-credential** | Per descriptor | Per descriptor | V2: `multiple` flag |
| **Revocation check** | Controller method | Async fetch | Hook-based |
| **Format support** | mDoc, SD-JWT | mDoc, SD-JWT | Multiple |
| **Nested claims** | Prefix matching | Type-based | Fully nested |

---

## Key Code References

| Implementation | File | Function/Class |
|----------------|------|----------------|
| EUDI Android | `RequestTransformer.kt` | `transformToDomainItems()` |
| EUDI Android | `ClaimPathDomain.kt` | `isPrefixOf()` |
| EUDI iOS | `RequestDataUIModel.swift` | `toUiModels()`, `prepareRequest()` |
| EUDI iOS | `PresentationInteractor.swift` | `onRequestReceived()` |
| Procivis | `proof-presentation-v1.tsx` | `presentationDefinition` |
| Procivis | `proof-request.ts` | `pickPreselectedCredential()` |
| Procivis | `credential-sharing-v2.tsx` | `preselectCredentialsForPresentationDefinitionV2()` |
