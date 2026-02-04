# Presentation Definition — Comparative Analysis

This document compares how EUDI, Procivis ONE, and Affinidi handle presentation definitions and credential matching.

---

## 1. Query Language Support

| Implementation | PEX v1 | PEX v2 | DCQL |
|----------------|--------|--------|------|
| **EUDI** | Translated | Translated | Native |
| **Procivis** | Native | Native | Not supported |
| **Affinidi** | Native | Not documented | Not documented |

### EUDI Approach

EUDI uses DCQL internally but accepts PEX format, translating as needed:

```kotlin
// SDK handles translation
// Input: PEX presentation_definition
// Internal: DCQL-style credential matching
```

### Procivis Approach

Direct PEX support with version detection:

```typescript
// proof-request-screen.tsx
const supportedPresentationDefinition =
  protocolCapabilities['supportedPresentationDefinition'];

if (supportedPresentationDefinition === SupportedPresentationDefinitionVersion.V2) {
  return <ProofPresentationV2 ... />;
} else {
  return <ProofPresentationV1 ... />;
}
```

### Affinidi Approach

PEX defined in Affinidi Portal:

```json
// Portal configuration
{
  "id": "event_ticket",
  "input_descriptors": [
    {
      "id": "event_ticket",
      "name": "EventTicket VC",
      "constraints": {
        "fields": [
          {
            "path": ["$.type"],
            "filter": {
              "type": "array",
              "contains": {
                "pattern": "^EventTicketVC$"
              }
            }
          }
        ]
      }
    }
  ]
}
```

---

## 2. Definition Retrieval

### EUDI

Presentation definition embedded in or referenced from authorization request:

```kotlin
// Received with request
data class RequestReceived(
    val processedRequest: ProcessedRequest,
    val requestedDocuments: List<RequestedDocument>,
    val verifierName: String?,
    val verifierIsTrusted: Boolean
)

// RequestedDocument contains parsed requirements
data class RequestedDocument(
    val docId: String,
    val requestedItems: Map<String, List<RequestedItem>>
)
```

### Procivis

Fetched separately from core after invitation:

```typescript
// V1
const presentationDefinition = await core.getPresentationDefinition(proofId);
// Returns: PresentationDefinitionBindingDto

// V2
const presentationDefinitionV2 = await core.getPresentationDefinitionV2(proofId);
// Returns: PresentationDefinitionV2ResponseBindingDto
```

### Affinidi

Pre-configured in Portal, referenced by queryId:

```typescript
await api.initiateDataSharingRequest({
    configurationId,
    queryId,  // References portal-defined presentation definition
    // ...
});
```

---

## 3. Data Structures

### EUDI

**RequestedDocument:**
```kotlin
data class RequestedDocument(
    val docId: String,
    val requestedItems: Map<NameSpace, List<RequestedItem>>,
    val readerAuth: ReaderAuth?
)

data class RequestedItem(
    val elementIdentifier: String,
    val isRequired: Boolean
)
```

### Procivis V1

```typescript
interface PresentationDefinitionBindingDto {
    requestGroups: PresentationDefinitionRequestGroupBindingDto[];
    credentials: CredentialDetailBindingDto[];
}

interface PresentationDefinitionRequestGroupBindingDto {
    requestedCredentials: PresentationDefinitionRequestedCredentialBindingDto[];
}

interface PresentationDefinitionRequestedCredentialBindingDto {
    id: string;
    applicableCredentials: string[];
    inapplicableCredentials: string[];
    fields: PresentationDefinitionFieldBindingDto[];
}
```

### Procivis V2

```typescript
interface PresentationDefinitionV2ResponseBindingDto {
    credentialSets: CredentialSet[];
    credentialQueries: Record<string, CredentialQueryResponseBindingDto>;
}

interface CredentialSet {
    required: boolean;
    options: string[][];
}

interface CredentialQueryResponseBindingDto {
    multiple: boolean;
    credentialOrFailureHint: {
        type_: 'APPLICABLE_CREDENTIALS';
        applicableCredentials: CredentialListItemBindingDto[];
    };
}
```

### Affinidi

PEX structure as defined in Portal, accessed via API response.

---

## 4. Credential Matching

### EUDI

**Location:** SDK handles matching internally

```kotlin
// RequestTransformer.kt
fun transformToDomainItems(
    storageDocuments: List<DocumentPayload>,
    requestedDocuments: List<RequestedDocument>,
    requiredFields: RequiredFields
): List<RequestDataUi> {
    // For each requested document
    // Find matching stored credentials
    // Match by document type and available claims
}
```

**Matching logic:**
1. Match by document type/format
2. Match by namespace (mDoc) or claim paths
3. Filter revoked documents
4. Return applicable credentials

### Procivis

**Location:** `app/utils/proof-request.ts`

```typescript
function preselectCredentialsForRequestGroups(
    requestGroups: PresentationDefinitionRequestGroupBindingDto[]
): RequestGroup[] {
    return requestGroups.map(group => ({
        requestedCredentials: group.requestedCredentials.map(request => {
            const preselectedCredential = pickPreselectedCredential(
                request.applicableCredentials,
                request.inapplicableCredentials,
                credentials
            );
            return {
                ...request,
                selectedCredentialId: preselectedCredential,
                fields: preselectFields(request.fields)
            };
        })
    }));
}

function pickPreselectedCredential(
    applicableCredentials: string[],
    inapplicableCredentials: string[],
    credentials: CredentialDetailBindingDto[]
): string | undefined {
    // Priority 1: Applicable + ACCEPTED state
    const applicableAccepted = credentials.find(
        c => applicableCredentials.includes(c.id) && c.state === 'ACCEPTED'
    );
    if (applicableAccepted) return applicableAccepted.id;

    // Priority 2: Inapplicable + ACCEPTED
    const inapplicableAccepted = credentials.find(
        c => inapplicableCredentials.includes(c.id) && c.state === 'ACCEPTED'
    );
    if (inapplicableAccepted) return inapplicableAccepted.id;

    // Fallback: first available
    return applicableCredentials[0] ?? inapplicableCredentials[0];
}
```

### Affinidi

Matching performed in Vault:
- User's credentials matched against PEX query
- Vault UI shows matching credentials
- User selects which to share

---

## 5. Selective Disclosure Handling

### EUDI

**Per-claim selection in UI:**

```kotlin
// RequestDataUi contains selectable items
data class RequestDataUi(
    val documentPayload: DocumentPayload,
    val requestedItems: List<RequestItemUi>,
    val expanded: Boolean
)

data class RequestItemUi(
    val claimPath: ClaimPathDomain,
    val displayValue: String,
    val isSelected: Boolean,
    val isRequired: Boolean
)
```

**User can toggle optional claims:**
```kotlin
fun toggleItemSelection(documentId: String, claimPath: ClaimPathDomain) {
    // Toggle isSelected for optional items
    // Required items cannot be deselected
}
```

### Procivis

**Field selection:**

```typescript
// Fields have preselected state
interface RequestedCredentialField {
    id: string;
    key: string;
    required: boolean;
    selected: boolean;  // User can toggle if not required
}

// In V2, user selections track claim paths
interface PresentationSubmitV2CredentialRequestBindingDto {
    credentialId: string;
    userSelections: string[];  // Selected claim paths
}
```

### Affinidi

Selective disclosure managed in Vault consent screen:
- Vault shows requested claims
- User consents to specific data points
- Vault constructs VP with selected claims

---

## 6. Required vs Optional Claims

| Implementation | Required Handling | Optional Handling |
|----------------|-------------------|-------------------|
| **EUDI** | Cannot deselect | Checkbox toggle |
| **Procivis** | Pre-selected, locked | Checkbox toggle |
| **Affinidi** | Shown as required | Checkbox in Vault |

### EUDI

```kotlin
// RequestItemUi
val isRequired: Boolean  // From presentation definition
val isSelected: Boolean  // User selection (locked if required)
```

### Procivis

```typescript
// Field selection logic
if (field.required) {
    return { ...field, selected: true };  // Cannot deselect
} else {
    return { ...field, selected: userPreference };
}
```

---

## 7. Multiple Credential Handling

### When Multiple Credentials Match

**EUDI:**
- Shows all matching credentials
- User selects which to present
- Each credential can have different claim selections

**Procivis:**
- Navigate to SelectCredential screen
- Grid view of matching credentials
- V2 supports multiple selection for same query

**Affinidi:**
- Vault shows matching credentials
- User picks from available options

### Credential Set Options (OR logic)

**Procivis V2:**

```typescript
interface CredentialSet {
    required: boolean;
    options: string[][];  // Each inner array is one valid combination
}

// Example: Accept passport OR (id-card AND utility-bill)
{
    options: [
        ["passport"],
        ["id-card", "utility-bill"]
    ]
}
```

---

## 8. Error States

### No Matching Credentials

**EUDI:**
```kotlin
sealed class PresentationRequestInteractorPartialState {
    object NoData : PresentationRequestInteractorPartialState()
    // Displayed: "No matching credentials found"
}
```

**Procivis:**
```typescript
// CredentialQueryResponseBindingDto
{
    credentialOrFailureHint: {
        type_: 'FAILURE_HINT',
        failureHint: 'No credentials match the request'
    }
}
```

**Affinidi:**
- Vault displays message if no matching credentials
- User cannot proceed without match

### Partial Match (Some Fields Missing)

All implementations show which fields are available vs missing, allowing user to decide whether to proceed with partial data.

---

## 9. Comparison Summary

| Aspect | EUDI | Procivis | Affinidi |
|--------|------|----------|----------|
| **Query format** | DCQL (native), PEX (translated) | PEX v1/v2 (native) | PEX (Portal) |
| **Definition location** | In request | Fetched from core | Pre-configured |
| **Matching engine** | SDK | Core library | Vault |
| **Claim selection** | Per-item toggle | Per-field toggle | Vault UI |
| **Multiple credentials** | User selection | Navigate to picker | Vault selection |
| **Logical grouping** | credential_sets (DCQL) | submission_requirements | submission_requirements |

---

## 10. Implementation Recommendations

### For DCQL/OID4VP v1.0 Compliance

→ **Use EUDI approach**
- Native DCQL support
- Clean credential/claim structure
- Future-proof for OID4VP evolution

### For PEX v1/v2 Interoperability

→ **Use Procivis approach**
- Direct PEX support
- Both versions handled
- Wide ecosystem compatibility

### For Simplified Integration

→ **Use Affinidi approach**
- Portal-based definition management
- No client-side PEX parsing
- Visual PEX editor available
