# Issuer Response -- Comparative Analysis

This document compares how EUDI Android, EUDI iOS, Procivis ONE, and Affinidi handle issuer responses to credential requests.

---

## Deferred Issuance Support

| Aspect | EUDI Android | EUDI iOS | Procivis ONE | Affinidi |
|--------|-------------|---------|-------------|---------|
| Deferred issuance supported | Yes | Yes | Protocol-level support | No |
| Deferred state type | `IssueEvent.DocumentDeferred` | `OfferResultPartialState.deferredSuccess` | N/A at app layer | N/A |
| Persistent storage of deferred records | Yes (SDK-managed) | Yes (SDK-managed) | SDK-internal | N/A |
| User-triggered retrieval | Yes | Yes (`resumePendingIssuance()`) | SDK-managed | N/A |
| UX indicator | "In Progress" status | "In Progress" status | N/A | N/A |
| Background polling | Implementation-dependent | Implementation-dependent | N/A | N/A |

### Analysis

**EUDI Android and iOS** provide first-class deferred issuance support. The SDK manages the deferred record lifecycle (persistence, retrieval, status tracking) and exposes clear states to the application layer. The user sees deferred credentials in their document list with an "In Progress" indicator, providing transparency about the issuance state.

**Procivis ONE** supports deferred issuance at the protocol level (the Core SDK can handle `transaction_id` responses), but the application-layer UX (credential-accept-result-screen.tsx) presents issuance as a synchronous success/failure outcome. Deferred retrieval, if it occurs, is managed internally by the SDK.

**Affinidi** does not implement deferred issuance. The claim link model assumes immediate credential delivery -- the issuer service constructs and returns the VC synchronously. If the issuer cannot produce the credential immediately, the Affinidi flow does not have a mechanism to defer and retry.

---

## Partial Success Handling

| Aspect | EUDI Android | EUDI iOS | Procivis ONE | Affinidi |
|--------|-------------|---------|-------------|---------|
| Partial success supported | Yes | Yes | No | No |
| State type | `IssueDocumentsPartialState.PartialSuccess` | `OfferResultPartialState.partialSuccess` | N/A | N/A |
| Issued + non-issued tracking | Yes (`issued` + `nonIssued` lists) | Yes | N/A | N/A |
| UX for partial results | Shows issued credentials, indicates pending/failed | Shows issued credentials, indicates pending/failed | Single success/failure per credential | Single success/failure per claim |

### Analysis

Partial success handling is a distinguishing capability of the EUDI implementations. When a credential offer contains multiple credential types and some succeed while others fail or are deferred, the EUDI wallets can present a nuanced result to the user.

**Example scenario:** A multi-credential offer includes both a PID and an mDL. The PID is issued immediately (it matches pre-verified identity data), but the mDL is deferred (it requires driving record verification). The EUDI wallet shows the PID as issued and the mDL as "In Progress."

Procivis ONE and Affinidi do not face this scenario because they process single credentials per request/offer. The concept of partial success does not apply when there is only one credential to succeed or fail.

---

## Dynamic Issuance

| Aspect | EUDI Android | EUDI iOS | Procivis ONE | Affinidi |
|--------|-------------|---------|-------------|---------|
| Dynamic issuance (OID4VP during OID4VCI) | No | **Yes** | No | No |
| Trigger | N/A | `authorizePresentationUrl` in issuer response | N/A | N/A |
| State type | N/A | `OfferResultPartialState.dynamicIssuance` | N/A | N/A |
| Resumption method | N/A | `resumePendingIssuance()` | N/A | N/A |

### Analysis

Dynamic issuance is unique to the EUDI iOS implementation and represents a significant architectural capability. It allows the issuer to require the holder to present existing credentials during the issuance flow, blending OID4VCI (issuance) and OID4VP (presentation) protocols.

**Why this matters:**
- It enables trust elevation during issuance. An issuer can verify the holder's existing credentials before issuing new ones.
- It supports use cases where issuance depends on the holder possessing specific existing credentials (e.g., proof of residency before issuing a local permit).
- It creates a feedback loop where the wallet's existing credential portfolio influences what new credentials can be issued.

The absence of this feature in EUDI Android is notable, as it suggests the feature is either iOS-specific or has not yet been implemented on Android. Procivis ONE and Affinidi do not implement this pattern.

---

## Redirect Mechanisms

| Aspect | EUDI Android | EUDI iOS | Procivis ONE | Affinidi |
|--------|-------------|---------|-------------|---------|
| Post-issuance redirect | No explicit redirect | No explicit redirect | Yes (`redirectUri`) | No |
| Redirect trigger | N/A | N/A | "Back to Service" button on result screen | N/A |
| Redirect target | N/A | N/A | Issuer's web service | N/A |

### Analysis

Procivis ONE's `redirectUri` support is unique and addresses a practical UX need: when the issuance flow is initiated from a web service (e.g., a government portal), the user should be able to return to that service after the credential is stored. Without a redirect mechanism, the user is stranded in the wallet app and must manually navigate back to the service.

The EUDI wallets do not implement explicit post-issuance redirects. After issuance, the user remains in the wallet app on the success screen. This is sufficient for QR-code-initiated flows (where the user started in the wallet) but may be suboptimal for web-initiated flows.

---

## Notification Support

| Aspect | EUDI Android | EUDI iOS | Procivis ONE | Affinidi |
|--------|-------------|---------|-------------|---------|
| Notification endpoint support | Yes (v1.0 Final) | Yes (v1.0 Final) | Yes (v1.0 Final variants) | No |
| `credential_accepted` | Yes | Yes | Yes | No |
| `credential_deleted` | Yes | Yes | Yes | No |
| `credential_failure` | Yes | Yes | Yes | No |

### Analysis

Notification support aligns with draft version support. The EUDI wallets and Procivis ONE (in v1.0 Final mode) support the notification endpoint, sending lifecycle events to the issuer. Affinidi does not implement the notification endpoint, relying instead on its own service-level tracking (marking offers as claimed).

The practical value of notifications depends on the issuer's use of them. For government issuers that need audit trails, notifications provide valuable lifecycle visibility. For simpler issuance scenarios, notifications add protocol complexity without clear benefit.

---

## Error Handling Approaches

| Aspect | EUDI Android | EUDI iOS | Procivis ONE | Affinidi |
|--------|-------------|---------|-------------|---------|
| Error granularity | Per-document (`DocumentFailed` event) | Per-document (within partial state) | Classified (RSE locked, invalid URL, other) | Service-level error |
| RSE lock detection | N/A | N/A | Yes (`isRSELockedError()`) | N/A |
| Invalid offer detection | SDK-internal | SDK-internal | Yes (`isInvalidInvitationUrlError()`) | Offer status check |
| Nonce refresh on error | SDK-internal (automatic) | SDK-internal (automatic) | SDK-internal (automatic) | Service-level |
| Credential rejection | No | No | Yes (if `SupportsRejection`) | No |

### Analysis

**EUDI Android** provides per-document error granularity through the event system. Each document in a multi-credential offer can fail independently, and the `DocumentFailed` event carries specific error information. The SDK handles nonce refresh errors internally, retrying with a new nonce when the issuer provides one in the error response.

**EUDI iOS** provides similar per-document error handling within the partial state system. Failed credentials are tracked alongside successful ones.

**Procivis ONE** takes a different approach by classifying errors into categories with distinct UX paths:
- RSE locked errors require specific recovery (RSE unlock), not a simple retry.
- Invalid invitation URL errors indicate the offer itself is invalid, not just the request.
- Other errors receive generic warning treatment.

This error classification is more actionable than generic error reporting because it directs the user toward the correct recovery action.

**Affinidi** handles errors at the service level. If the credential claim fails, the Vault reports the error. The offer status tracking (claimed vs. available) prevents re-use of consumed offers, which is a form of error prevention rather than error recovery.

---

## Credential Rejection

| Aspect | EUDI Android | EUDI iOS | Procivis ONE | Affinidi |
|--------|-------------|---------|-------------|---------|
| User can reject offered credential | No | No | Yes (protocol-dependent) | No |
| Rejection communicated to issuer | N/A | N/A | Yes | N/A |
| Feature gate | N/A | N/A | `IssuanceProtocolFeatureEnum.SupportsRejection` | N/A |

### Analysis

Credential rejection is a Procivis ONE feature that allows the user to decline a credential after reviewing its claims. This is distinct from simply not completing the issuance flow -- rejection is an explicit protocol message sent to the issuer, informing them that the holder does not want the credential.

**Use cases for rejection:**
- The user reviews the credential claims and finds them incorrect.
- The user does not want the credential for privacy reasons.
- The user was offered a credential they did not request (e.g., an issuer pushes an unwanted credential type).

The EUDI wallets and Affinidi do not support credential rejection. If the user does not want a credential, they simply do not complete the issuance flow (or do not accept the credential in the Vault). The issuer does not receive an explicit rejection signal.

---

## Strengths and Weaknesses

| Implementation | Strengths | Weaknesses |
|----------------|-----------|------------|
| **EUDI Android** | Granular event-based state management. Partial success handling for multi-credential offers. SDK-managed deferred credential storage. Per-document error tracking. Success screen orchestration via dedicated interactor. | No dynamic issuance. No post-issuance redirect. No credential rejection. Event callback complexity adds implementation overhead. |
| **EUDI iOS** | Dynamic issuance (OID4VP integration) is a unique and powerful capability. Partial success and deferred handling match Android. Clean async API with OfferResultPartialState. | Dynamic issuance adds flow complexity. No post-issuance redirect. No credential rejection. Dynamic issuance is iOS-only, creating platform asymmetry. |
| **Procivis ONE** | Error classification provides actionable recovery guidance. Redirect URI enables web-service integration. Credential rejection gives users explicit control. Clean three-state result screen. | No partial success handling (single credential per request). Limited deferred issuance UX. No dynamic issuance. |
| **Affinidi** | Simplest model -- single credential, immediate delivery, explicit user acceptance. Revocation support post-issuance. | No deferred issuance. No partial success. No notification support. No credential rejection. No error recovery mechanisms beyond retry. |

---

## Summary

The issuer response handling reveals significant architectural differences across implementations. The EUDI wallets provide the most comprehensive response handling, with partial success, deferred issuance, and (on iOS) dynamic issuance. Procivis ONE offers practical features (redirect, rejection, error classification) that address real deployment needs. Affinidi follows the simplest model, which works well for its single-credential, immediate-delivery use case but lacks the sophistication needed for complex multi-credential or deferred scenarios. The choice of implementation should consider whether the deployment requires deferred issuance, partial success handling, dynamic issuance, or credential rejection -- as these capabilities are not uniformly available.
