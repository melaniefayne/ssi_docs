# Presentation Definition — Implementation Details

This document details how EUDI (Android/iOS) and Procivis ONE parse and process presentation definitions.

---

## EUDI Android

### Requested Document Model

**Location:** `core-logic/src/main/java/eu/europa/ec/corelogic/model/`

The SDK parses presentation definitions into `RequestedDocument` objects:

```kotlin
data class RequestedDocument(
    val documentId: String,
    val docType: String,
    val requestedItems: Map<String, Boolean>,  // claim -> required
    val readerAuth: ReaderAuth?
)
```

### Request Transformer

**Location:** `common-feature/src/main/java/eu/europa/ec/commonfeature/ui/request/transformer/RequestTransformer.kt`

```kotlin
fun transformToDomainItems(
    storageDocuments: List<IssuedDocument>,
    resourceProvider: ResourceProvider,
    uuidProvider: UuidProvider,
    requestDocuments: List<RequestedDocument>,
): Result<List<DocumentPayloadDomain>> = runCatching {

    requestDocuments.forEach { requestDocument ->
        // Find matching credential in storage
        val storageDocument = storageDocuments.first {
            it.id == requestDocument.documentId
        }

        // Get requested item paths
        val requestedItemsPaths = requestDocument.requestedItems.keys
            .map { it.toClaimPath() }

        // Filter available claims by request
        val filteredPaths = claimsPaths.filter { available ->
            requestedItemsPaths.any { requested ->
                requested.isPrefixOf(available)
            }
        }

        // Build UI model with matched claims
        // ...
    }
}
```

### Claim Path Domain

**Location:** `core-logic/src/main/java/eu/europa/ec/corelogic/model/ClaimPathDomain.kt`

```kotlin
data class ClaimPathDomain(val path: List<String>) {

    fun isPrefixOf(other: ClaimPathDomain): Boolean {
        if (path.size > other.path.size) return false
        return path.zip(other.path).all { (a, b) -> a == b }
    }

    companion object {
        fun toSdJwtVcPath(pathString: String): List<String> {
            return pathString.split("/")
        }

        fun toElementIdentifier(pathString: String): String {
            return pathString.split("/").lastOrNull() ?: pathString
        }
    }
}
```

---

## EUDI iOS

### Presentation Request Model

**Location:** `Modules/logic-core/Sources/Coordinator/Model/PresentationRequest.swift`

```swift
public struct PresentationRequest: Sendable, Equatable {
    public let items: [DocElements]
    public let relyingParty: String
    public let dataRequestInfo: String
    public let isTrusted: Bool
}
```

### Document Elements

The SDK provides parsed elements by format:

```swift
public enum DocElements: Sendable, Equatable {
    case msoMdoc(MsoMdocElements)
    case sdJwt(SdJwtElements)
}

public struct MsoMdocElements: Sendable, Equatable {
    public let docType: String
    public let displayName: String?
    public let claims: [DocClaim]
}

public struct SdJwtElements: Sendable, Equatable {
    public let vct: String
    public let displayName: String?
    public let claims: [DocClaim]
}
```

### Request Parsing in Coordinator

**Location:** `Modules/logic-core/Sources/Coordinator/RemoteSessionCoordinator.swift`

```swift
public func requestReceived() async throws -> PresentationRequest {
    guard session.disclosedDocuments.isEmpty == false else {
        throw session.uiError ?? .init(description: "Failed to Find known documents to send")
    }
    return createRequest()
}

private func createRequest() -> PresentationRequest {
    PresentationRequest(
        items: session.disclosedDocuments,
        relyingParty: session.readerCertIssuer ?? LocalizableStringKey.unknownVerifier.toString,
        dataRequestInfo: session.readerCertValidationMessage ?? "",
        isTrusted: session.readerCertIssuerValid == true
    )
}
```

---

## Procivis ONE

### Presentation Definition V1

**Location:** `app/components/proof-request/proof-presentation-v1.tsx`

```typescript
const presentationDefinition = useMemoAsync(async () => {
    const definition = await core.getPresentationDefinition(proofId);
    onPresentationDefinitionLoaded();

    // Refresh revocation status
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

### V1 Structure

```typescript
interface PresentationDefinitionV1 {
    requestGroups: Array<{
        id: string;
        name?: string;
        requestedCredentials: Array<{
            id: string;
            name?: string;
            fields: Array<{
                id: string;
                name: string;
                required: boolean;
                keyMap: Record<string, string>;
            }>;
            applicableCredentials: string[];
            inapplicableCredentials: string[];
        }>;
    }>;
}
```

### Presentation Definition V2

**Location:** `app/components/proof-request/proof-presentation-v2.tsx`

```typescript
const presentationDefinition = useMemoAsync(async () => {
    const definition = await core.getPresentationDefinitionV2(proofId);
    onPresentationDefinitionLoaded();

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

### V2 Structure

```typescript
interface PresentationDefinitionV2 {
    credentialSets: Array<{
        required: boolean;
        options: string[][];  // Each option is array of query IDs
        valid: boolean;
    }>;
    credentialQueries: Record<string, {
        multiple: boolean;
        claims: Array<{
            path: string;
            required: boolean;
        }>;
        credentialOrFailureHint: {
            type_: 'APPLICABLE_CREDENTIALS' | 'FAILURE';
            applicableCredentials?: Array<{
                id: string;
                issuanceDate: string;
            }>;
        };
    }>;
}
```

### Protocol Detection

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

## Comparison Summary

| Aspect | EUDI Android | EUDI iOS | Procivis ONE |
|--------|--------------|----------|--------------|
| **Definition parsing** | SDK handles | SDK handles | Core library |
| **Version support** | Single | Single | V1 and V2 |
| **Matching** | SDK + transformer | SDK + converter | Core library |
| **Path format** | ClaimPathDomain | Array | String paths |
| **Credential grouping** | Per document | Per document | Request groups / Sets |

---

## Key Code References

| Implementation | File | Function |
|----------------|------|----------|
| EUDI Android | `RequestTransformer.kt` | `transformToDomainItems()` |
| EUDI Android | `ClaimPathDomain.kt` | `isPrefixOf()` |
| EUDI iOS | `RemoteSessionCoordinator.swift` | `requestReceived()` |
| EUDI iOS | `PresentationRequest.swift` | Model definition |
| Procivis | `proof-presentation-v1.tsx` | `presentationDefinition` |
| Procivis | `proof-presentation-v2.tsx` | `presentationDefinition` |
