# Issuer Response -- Implementation Details

This document details how each wallet implementation handles the issuer's response to a credential request, including callback systems, state management, deferred credential retrieval, and the UX paths presented to the user.

---

## EUDI Android (Reference Implementation)

### IssueEvent Callback System

The EUDI Android SDK communicates issuance progress and outcomes through the **IssueEvent** sealed class hierarchy. Each event represents a distinct state in the issuance lifecycle, and the application layer handles them via the `onEvent` callback provided to `issueDocumentByOffer()` or `issueDocumentByConfigurationIdentifier()`.

**Event types:**

| Event | Description |
|-------|-------------|
| `IssueEvent.Started` | The issuance flow has begun. The SDK is resolving the offer and preparing requests. |
| `IssueEvent.DocumentRequiresCreateSettings` | The SDK needs document creation settings (credential policy, batch size) before proceeding. The app provides `CreateDocumentSettings` in response. |
| `IssueEvent.DocumentRequiresUserAuth` | The proof signing key requires user authentication. The app must present a BiometricPrompt and return the authentication result (see [07 -- Proof Construction](../07-proof-construction/)). |
| `IssueEvent.DocumentIssued` | A credential was successfully issued and stored. Contains the document ID. |
| `IssueEvent.DocumentDeferred` | A credential was deferred. Contains the deferred document record for later retrieval. |
| `IssueEvent.DocumentFailed` | A credential request failed. Contains the error information. |
| `IssueEvent.Finished` | The entire issuance flow has completed (all documents processed). |

### Partial State Handling

After the issuance flow completes, the overall outcome is represented by **IssueDocumentsPartialState**:

| State | Description |
|-------|-------------|
| `Success` | All requested credentials were issued successfully. Contains `documentIds` -- a list of stored document identifiers. |
| `DeferredSuccess` | All requested credentials were deferred. Contains `deferredDocuments` -- a map of credential configuration IDs to deferred document records. |
| `PartialSuccess` | Some credentials were issued and some were not. Contains two lists: `issued` (successfully stored document IDs) and `nonIssued` (credential IDs that failed or were deferred). |
| `Failure` | All credential requests failed. Contains the error information. |

The `PartialSuccess` state is particularly important for multi-credential offers. If an offer contains both a PID and an mDL, and the PID is issued immediately but the mDL is deferred, the result is `PartialSuccess` with the PID in the `issued` list and the mDL in the deferred or non-issued list.

The `IssueEvent.Finished` handler determines the final state:

```kotlin
// File: core-logic/src/main/java/eu/europa/ec/corelogic/controller/WalletCoreDocumentsController.kt

is IssueEvent.Finished -> {

    if (deferredDocuments.isNotEmpty()) {
        trySendBlocking(IssueDocumentsPartialState.DeferredSuccess(deferredDocuments))
        return@OnIssueEvent
    }

    if (event.issuedDocuments.isEmpty()) {
        trySendBlocking(
            IssueDocumentsPartialState.Failure(
                errorMessage = documentErrorMessage
            )
        )
        return@OnIssueEvent
    }

    if (event.issuedDocuments.size == totalDocumentsToBeIssued) {
        trySendBlocking(
            IssueDocumentsPartialState.Success(
                documentIds = event.issuedDocuments
            )
        )
        return@OnIssueEvent
    }

    trySendBlocking(
        IssueDocumentsPartialState.PartialSuccess(
            documentIds = event.issuedDocuments,
            nonIssuedDocuments = nonIssuedDocuments
        )
    )
}
```

### Deferred Document Handling

When a credential is deferred, the SDK stores the deferred document record (including the `transaction_id`, issuer endpoint, and credential metadata) in persistent storage. The wallet can later attempt to retrieve the deferred credential.

The retrieval is typically triggered by:
- User action (tapping on the "In Progress" credential in the document list).
- Background polling (the wallet periodically checks for deferred credentials).

### Success Screen

The **DocumentIssuanceSuccessInteractor** orchestrates the success UX after issuance completes. It determines which success screen to display based on the `IssueDocumentsPartialState`:

- **Full success** -- Shows the issued credential(s) with a success message.
- **Partial success** -- Shows issued credentials and indicates which credentials are pending or failed.
- **Deferred success** -- Shows the credential(s) with an "In Progress" status indicator.

---

## EUDI iOS (Reference Implementation)

### OfferResultPartialState

The EUDI iOS SDK represents issuance outcomes through the **OfferResultPartialState** enum:

| State | Description |
|-------|-------------|
| `success` | All credentials were issued successfully. |
| `partialSuccess` | Some credentials were issued and some were not. Contains both successful and unsuccessful results. |
| `deferredSuccess` | All credentials were deferred. |
| `dynamicIssuance` | The issuer requires an OID4VP presentation before completing issuance. This is a unique EUDI iOS feature. |
| `failure` | All credential requests failed. |

### Dynamic Issuance (OID4VP Integration)

The `dynamicIssuance` state is a notable feature unique to the EUDI iOS implementation. When the issuer requires the wallet to present existing credentials before issuing new ones, the issuance flow is paused and an OID4VP (OpenID for Verifiable Presentations) flow is initiated.

**How it works:**

1. The wallet sends a credential request.
2. The issuer responds with an `authorizePresentationUrl` instead of a credential.
3. The SDK detects this response and emits the `dynamicIssuance` state.
4. The wallet starts an OID4VP presentation flow, presenting the requested credentials to the issuer.
5. After the presentation is verified, the issuance resumes.
6. The wallet calls `resumePendingIssuance()` to complete the credential request.

```
Wallet                          Issuer                          Verifier
  |                               |                               |
  |  POST /credential             |                               |
  |------------------------------>|                               |
  |                               |                               |
  |  "Present credentials first"  |                               |
  |  authorizePresentationUrl     |                               |
  |<------------------------------|                               |
  |                               |                               |
  |  Start OID4VP flow            |                               |
  |-------------------------------------------------------------->|
  |                               |                               |
  |  Present requested VCs        |                               |
  |-------------------------------------------------------------->|
  |                               |                               |
  |  Presentation verified        |                               |
  |<--------------------------------------------------------------|
  |                               |                               |
  |  resumePendingIssuance()      |                               |
  |------------------------------>|                               |
  |                               |  Issue credential             |
  |  Credential issued            |                               |
  |<------------------------------|                               |
  |                               |                               |
```

The actual implementation in `DocumentOfferInteractorImpl`:

```swift
// File: Modules/feature-issuance/Sources/Interactor/DocumentOfferInteractor.swift

let issuedDocuments = try await walletController.issueDocumentsByOfferUrl(
    offerUri: uri,
    docTypes: docOffers,
    txCodeValue: txCodeValue
)

if issuedDocuments.isEmpty {
    return .failure(WalletCoreError.unableToIssueAndStore)
} else if issuedDocuments.first(where: { $0.isDeferred }) != nil {
    return .deferredSuccess(retrieveDeferredRoute(/* ... */))
} else if let authorizePresentationUrl = issuedDocuments.first?.authorizePresentationUrl {
    guard
      let presentationUrl = authorizePresentationUrl.toCompatibleUrl(),
      let presentationComponents = URLComponents(url: presentationUrl, resolvingAgainstBaseURL: true) else {
      return .failure(WalletCoreError.unableToIssueAndStore)
    }
    let session = await walletController.startSameDevicePresentation(deepLink: presentationComponents)
    return .dynamicIssuance(session)
} else if issuedDocuments.count == docOffers.count {
    let documentIdentifiers = issuedDocuments.compactMap { $0.id }
    return await fetchAndHandleDocuments(
      successNavigation: successNavigation,
      documentIdentifiers: documentIdentifiers
    )
} else {
    let documentIdentifiers = issuedDocuments.compactMap { $0.id }
    return await fetchAndHandleDocuments(
      successNavigation: successNavigation,
      documentIdentifiers: documentIdentifiers,
      isPartialState: true
    )
}
```

**Use case example:** The issuer wants to issue a PID but requires the wallet to present an existing identity document (e.g., a national ID card already stored in the wallet) to verify the holder's identity before issuance. The dynamic issuance flow enables this without leaving the issuance protocol.

### Deferred Credential Retrieval

Deferred credentials are retrieved via `resumePendingIssuance()`. The wallet stores the deferred record and allows the user to trigger retrieval later. The deferred credential appears in the document list with an "In Progress" status.

### Three UX Paths

The EUDI iOS wallet presents three distinct UX paths based on the issuance outcome:

1. **Immediate success** -- The credential is issued and stored. The user sees a success screen with the credential details.
2. **Deferred ("In Progress")** -- The credential is pending. The user sees the credential in their document list with a visual indicator that it is not yet issued. They can tap it to attempt retrieval.
3. **Dynamic issuance** -- The wallet prompts the user to present existing credentials. After presentation, issuance completes automatically or the user triggers resumption.

---

## Procivis ONE

### Credential Accept Result Screen

The Procivis ONE wallet handles issuer responses through the **credential-accept-result-screen.tsx** React Native component, which presents the outcome to the user.

### Three Result States

| State | Description | UX |
|-------|-------------|-----|
| `Success` | The credential was issued and stored successfully. | Green success indicator. Displays credential summary. Optional "Back to Service" button if `redirectUri` is provided. |
| `Error` (RSE locked) | The issuance failed because the Remote Signing Element is locked (typically from too many failed PIN attempts). | Error screen with RSE-specific messaging. User must unlock the RSE before retrying. |
| `Warning` (other errors) | The issuance encountered a non-fatal error or an unexpected condition. | Warning screen with error details. User can retry or dismiss. |

The state determination logic:

```typescript
// File: app/screens/credential/credential-accept-result-screen.tsx

const { error, redirectUri } = route.params;

const state = useMemo(() => {
    if (!error) {
      return LoaderViewState.Success;
    }
    if (isRSELockedError(error)) {
      return LoaderViewState.Error;
    }
    return LoaderViewState.Warning;
  }, [error]);

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

### Error Classification

The Procivis ONE implementation classifies errors using dedicated helper functions:

- **`isRSELockedError()`** -- Detects when the Remote Signing Element is locked due to repeated failed PIN attempts. This is a distinct error condition that requires specific recovery (RSE unlock) rather than a simple retry.
- **`isInvalidInvitationUrlError()`** -- Detects when the credential offer URL is invalid, expired, or already consumed. This triggers a specific error message indicating the offer is no longer valid.

### Redirect URI

The credential accept result screen supports an optional **`redirectUri`** parameter. When present, a "Back to Service" button is displayed that navigates the user back to the issuing service's web interface. This is useful for scenarios where the issuance is initiated from a web service and the user should return to that service after the credential is stored.

### Credential Rejection

Procivis ONE supports **credential rejection** when the issuer's protocol supports it. The feature is gated by `IssuanceProtocolFeatureEnum.SupportsRejection`:

- If the issuer supports rejection, the wallet can decline the offered credential.
- The rejection is communicated back to the issuer via the protocol.
- This is relevant for scenarios where the user reviews the credential claims before acceptance and decides not to accept.

Not all issuance protocols support rejection. The wallet checks the feature flag before exposing the rejection option in the UI.

---

## Affinidi

### Credential Delivery

In the Affinidi ecosystem, the issuer service validates the proof, constructs the Verifiable Credential, and returns it to the Vault in the credential response.

### Issuance Flow Completion

1. The issuer service validates the proof of possession (signature and challenge verification).
2. The service constructs the VC with the holder's claims and the issuer's signature.
3. The offer status is updated to **claimed**, preventing re-use of the claim link.
4. The VC is returned to the Vault.

### Single-Claim Per Offer

Each credential offer (claim link) in Affinidi corresponds to exactly one credential. Once the credential is claimed, the offer is marked as consumed and cannot be used again. This is a design constraint that ensures each claim link produces exactly one credential, preventing duplication or unauthorized re-issuance.

### User Acceptance

After the Vault receives the credential, the user **accepts** the credential for storage. This is an explicit user action -- the credential is not automatically stored. The acceptance step allows the user to review the credential's claims before committing it to their Vault.

### Revocation Support

Affinidi supports **post-issuance revocation**. After a credential has been issued and stored, the issuer can revoke it. Revocation is handled through a revocation registry that verifiers check during credential presentation. This is a post-issuance lifecycle feature that does not directly affect the initial issuance response handling but is part of the overall credential lifecycle.

---

## Summary

The four implementations handle issuer responses with different levels of granularity and UX sophistication. The EUDI wallets (Android and iOS) provide the most granular state management, with distinct handling for success, deferred, partial success, and the iOS-unique dynamic issuance flow. Procivis ONE offers clean three-state result handling with RSE-specific error classification and redirect support. Affinidi follows the simplest model with single-credential delivery and explicit user acceptance. The key architectural differences are in partial success handling (EUDI-specific), dynamic issuance (iOS-specific), and credential rejection (Procivis-specific).
