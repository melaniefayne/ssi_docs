# Procivis ONE Architecture for Verification

Procivis ONE implements verification through a React Native cross-platform architecture, with a strong separation between UI components and the core Rust-based SDK. The verification flow supports both Presentation Definition V1 and V2, providing flexibility for different credential query specifications.

---

## 1. Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│  React Native UI Layer                                       │
│  ┌─────────────────┐  ┌─────────────────┐                   │
│  │ ProofRequest    │  │ SelectCredential│                   │
│  │ Screen          │  │ Screen(s)       │                   │
│  └────────┬────────┘  └────────┬────────┘                   │
│           │                    │                             │
│  ┌────────▼────────────────────▼────────┐                   │
│  │ ShareCredential Navigator            │                   │
│  └────────────────┬─────────────────────┘                   │
├───────────────────┼─────────────────────────────────────────┤
│  Business Logic   │                                          │
│  ┌────────────────▼─────────────────────┐                   │
│  │ proof-request.ts (utilities)         │                   │
│  │ - preselectCredentialsForRequestGroups│                  │
│  │ - preselectCredentialsForPresentationDefinitionV2        │
│  └────────────────┬─────────────────────┘                   │
├───────────────────┼─────────────────────────────────────────┤
│  React Native Bridge                                         │
│  ┌────────────────▼─────────────────────┐                   │
│  │ @procivis/react-native-one-core      │                   │
│  │ - getPresentationDefinition()        │                   │
│  │ - getPresentationDefinitionV2()      │                   │
│  │ - holderSubmitProof()                │                   │
│  │ - holderSubmitProofV2()              │                   │
│  └────────────────┬─────────────────────┘                   │
├───────────────────┼─────────────────────────────────────────┤
│  Rust Core        │                                          │
│  ┌────────────────▼─────────────────────┐                   │
│  │ Procivis ONE Core Library            │                   │
│  │ - OID4VP protocol implementation     │                   │
│  │ - Presentation Definition parsing    │                   │
│  │ - Credential matching                │                   │
│  │ - RSE integration                    │                   │
│  └──────────────────────────────────────┘                   │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Key Components

### Invitation Processing

**Location:** `app/screens/credential/invitation-process-screen.tsx`

The entry point for verification requests. When a QR code or invitation URL is processed:

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

### ShareCredential Navigator

**Location:** `app/navigators/share-credential/share-credential-navigator.tsx`

Routes for the verification flow:

| Route | Purpose |
|-------|---------|
| `ProofRequest` | Main presentation request display |
| `SelectCredential` | V1 credential selection |
| `SelectCredentialV2` | V2 credential selection |
| `Processing` | Submission processing |

### Presentation Definition Handling

**Location:** `app/screens/credential/proof-request-screen.tsx`

Determines whether to use V1 or V2 presentation definition:

```typescript
const verificationProtocol = config.verificationProtocol[proof.protocol];
const protocolCapabilities = verificationProtocol?.capabilities;
const supportedPresentationDefinition =
  protocolCapabilities['supportedPresentationDefinition'];

// Render appropriate component
{supportedPresentationDefinition ===
  SupportedPresentationDefinitionVersion.V2 ? (
  <ProofPresentationV2 ... />
) : (
  <ProofPresentationV1 ... />
)}
```

---

## 3. Presentation Definition V1

**Location:** `app/components/proof-request/proof-presentation-v1.tsx`

### Data Structure

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

### Flow

1. Fetch presentation definition: `core.getPresentationDefinition(proofId)`
2. Preselect credentials using `preselectCredentialsForRequestGroups()`
3. Display request groups and fields
4. User selects/deselects fields
5. Navigate to SelectCredential if multiple options
6. Submit with `holderSubmitProof()`

---

## 4. Presentation Definition V2

**Location:** `app/components/proof-request/proof-presentation-v2.tsx`

### Data Structure

```typescript
interface PresentationDefinitionV2ResponseBindingDto {
  credentialSets: CredentialSet[];
  credentialQueries: Record<string, CredentialQueryResponseBindingDto>;
}

interface CredentialSet {
  required: boolean;
  options: string[][]; // credential query IDs
}

interface CredentialQueryResponseBindingDto {
  multiple: boolean;
  credentialOrFailureHint: {
    type_: 'APPLICABLE_CREDENTIALS';
    applicableCredentials: CredentialListItemBindingDto[];
  };
}
```

### Key Differences from V1

| Aspect | V1 | V2 |
|--------|----|----|
| Structure | Request groups with fields | Credential sets with queries |
| Multiple credentials | One per request | Multiple options per set |
| Required/Optional | Implicit | Explicit `required` flag |
| User selections | Claim paths | User path selections |

### Flow

1. Fetch V2 definition: `core.getPresentationDefinitionV2(proofId)`
2. Convert to credential sets: `credentialSetsFromPresentationDefinitionV2()`
3. Separate simple sets from option sets
4. Handle credential set selections
5. Submit with `holderSubmitProofV2()`

---

## 5. Credential Selection Utilities

**Location:** `app/utils/proof-request.ts`

### Automatic Preselection

```typescript
function preselectCredentialsForRequestGroups(
  requestGroups: PresentationDefinitionRequestGroupBindingDto[]
): RequestGroup[] {
  // For each requested credential:
  // 1. Find applicable credentials with ACCEPTED state
  // 2. Sort by preference (most recent, etc.)
  // 3. Preselect fields based on required status
}

function preselectCredentialsForPresentationDefinitionV2(
  credentialSets: CredentialSet[],
  credentialQueries: Record<string, CredentialQueryResponseBindingDto>
): SelectedCredentialV2[] {
  // For each credential set:
  // 1. Check if required
  // 2. Find applicable credentials
  // 3. Sort by issuance date (newest first)
  // 4. Preselect for required sets
}
```

### Selection Priority

```typescript
function pickPreselectedCredential(
  applicableCredentials: string[],
  inapplicableCredentials: string[],
  credentials: CredentialDetailBindingDto[]
): string | undefined {
  // Priority 1: Applicable + ACCEPTED
  // Priority 2: Inapplicable + ACCEPTED
  // Priority 3: First available
}
```

---

## 6. Proof Submission

**Location:** `app/screens/credential/proof-process-screen.tsx`

### V1 Submission

```typescript
const useProofAccept = () => {
  return useMutation({
    mutationFn: async ({
      interactionId,
      credentials,
    }: {
      interactionId: string;
      credentials: Record<string, PresentationSubmitCredentialRequestBindingDto[]>;
    }) => {
      await core.holderSubmitProof(interactionId, credentials);
    },
  });
};
```

### V2 Submission

```typescript
const useProofAcceptV2 = () => {
  return useMutation({
    mutationFn: async ({
      interactionId,
      credentials,
    }: {
      interactionId: string;
      credentials: CredentialQuerySelection;
    }) => {
      await core.holderSubmitProofV2(interactionId, credentials);
    },
  });
};

// Submission structure
type CredentialQuerySelection = Record<
  string,
  PresentationSubmitV2CredentialRequestBindingDto[]
>;

interface PresentationSubmitV2CredentialRequestBindingDto {
  credentialId: string;
  userSelections: string[]; // claim paths
}
```

---

## 7. Remote Secure Element (RSE) Integration

**Location:** `app/screens/rse/rse-sign-screen.tsx`

Procivis ONE uses a Remote Secure Element for cryptographic signing:

```typescript
// When signing is required
if (requiresRseSigning) {
  navigation.navigate('RSESign', {
    onSuccess: () => submitProof(),
    onError: handleSigningError,
  });
}
```

### RSE Flow

1. Detect signing requirement
2. Navigate to RSE PIN entry screen
3. User enters PIN
4. Ubiqu SDK performs remote signing
5. Return to submission flow
6. Handle locked state errors

**Location:** `app/utils/rse.ts`
```typescript
function isRseLockedError(error: unknown): boolean {
  // Detect RSE locked state for error handling
}
```

---

## 8. Verifier-Side: Proof Proposal

For applications acting as verifiers, Procivis ONE supports proof proposal:

**Location:** `app/screens/dashboard/qr-code-share-screen.tsx`

```typescript
const { proposeProof } = useProposeProof();

const startVerification = async () => {
  const proofId = await proposeProof({
    protocol: VerificationProtocol.ISO_MDL,
    // or other supported protocols
  });

  // Generate QR code with proof request URL
  // Monitor proof state for transitions
};
```

### Proof State Transitions

```
PENDING → REQUESTED → ACCEPTED/REJECTED
```

---

## 9. Navigation Structure

```typescript
// share-credential-routes.ts
type ShareCredentialParamList = {
  ProofRequest: {
    request: InvitationResultBindingDto;
  };
  SelectCredential: {
    requestedCredential: RequestedCredential;
    credentials: CredentialDetailBindingDto[];
    onSelect: (credentialId: string) => void;
  };
  SelectCredentialV2: {
    credentialQuery: CredentialQueryResponseBindingDto;
    onSelect: (credentialIds: string[]) => void;
  };
  Processing: {
    proofId: string;
    selectedCredentials: Record<string, ...>;
  };
};
```

---

## 10. Protocol Configuration

Procivis ONE supports multiple verification protocols configured at the application level:

```typescript
interface VerificationProtocolConfig {
  [protocol: string]: {
    capabilities: {
      supportedPresentationDefinition: SupportedPresentationDefinitionVersion;
      // ... other capabilities
    };
  };
}

enum SupportedPresentationDefinitionVersion {
  V1 = 'V1',
  V2 = 'V2',
}
```

Supported protocols include:
- `ISO_MDL` — ISO Mobile Driving License (OID4VP for mDL)
- Custom protocols as configured

---

## 11. Key Files Reference

| File | Purpose |
|------|---------|
| `invitation-process-screen.tsx` | Entry point for proof requests |
| `proof-request-screen.tsx` | Main proof request display, V1/V2 switching |
| `proof-presentation-v1.tsx` | V1 presentation definition UI |
| `proof-presentation-v2.tsx` | V2 presentation definition UI |
| `proof-request-select-credential-screen.tsx` | V1 credential picker |
| `proof-request-select-credential-v2-screen.tsx` | V2 credential picker |
| `proof-process-screen.tsx` | Submission and signing |
| `proof-request.ts` | Credential preselection logic |
| `credential-sharing.ts` | Presentation labels/UI config |
| `share-credential-navigator.tsx` | Navigation structure |
| `rse-sign-screen.tsx` | Cryptographic signing UI |

---

## 12. Architectural Characteristics

| Aspect | Implementation |
|--------|----------------|
| **Cross-platform** | React Native with native bridges |
| **Core library** | Rust-based, accessed via React Native bridge |
| **State management** | React hooks + React Query |
| **Navigation** | React Navigation |
| **Signing** | Remote Secure Element (RSE) |
| **PEX versions** | V1 and V2 supported |
| **Protocol flexibility** | Configurable via protocol capabilities |
