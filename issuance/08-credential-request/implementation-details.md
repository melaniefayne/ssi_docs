# Credential Request -- Implementation Details

This document details how each wallet implementation constructs and sends credential requests, including draft version support, batch issuance configuration, and credential policies.

---

## EUDI Android (Reference Implementation)

### Draft Version

EUDI Android uses **OID4VCI v1.0 (Final)**. The SDK constructs credential requests using `credential_configuration_id` and follows the v1.0 specification for all endpoint interactions.

### Credential Request API

The EUDI Android SDK provides two primary methods for initiating credential issuance, both on the **OpenId4VciManager**:

**1. `issueDocumentByOffer()`**

Used when the wallet has received a credential offer (via QR code, deep link, or other delivery mechanism). The SDK resolves the offer, extracts the `credential_configuration_ids`, and constructs credential requests for each offered credential.

```
issueDocumentByOffer(
    offer: Offer,
    onEvent: (IssueEvent) -> Unit
)
```

The `Offer` object encapsulates the parsed credential offer, including the issuer URL, credential configuration IDs, and grant parameters.

**2. `issueDocumentByConfigurationIdentifier()`**

Used for wallet-initiated issuance, where the wallet has discovered available credentials from the issuer's metadata and the user selects a credential to request.

```
issueDocumentByConfigurationIdentifier(
    configurationId: CredentialConfigurationId,
    onEvent: (IssueEvent) -> Unit
)
```

Both methods take an `onEvent` callback that receives `IssueEvent` instances as the issuance progresses through its stages (see [09 -- Issuer Response](../09-issuer-response/) for event handling details).

### Credential Request Payload

The SDK constructs the credential request payload internally. The application layer does not directly assemble the JSON body. The SDK:

1. Resolves the credential configuration from issuer metadata.
2. Determines the format and format-specific parameters (`doctype`, `vct`, etc.).
3. Constructs the proof JWT (as described in [07 -- Proof Construction](../07-proof-construction/)).
4. Assembles the request body with `credential_configuration_id`, `proof` (or `proofs`), and format parameters.
5. Sends the POST request to the issuer's `credential_endpoint`.

### Batch Issuance

EUDI Android supports batch credential issuance through the **DocumentIssuanceConfig** and **CreateDocumentSettings** classes.

**Credential Policies:**

The SDK defines credential policies that control batch behavior:

- **`rotateUse`** -- The default policy. A single credential is issued and reused across presentations. The wallet rotates the credential's key material periodically but does not request multiple instances.
- **`oneTimeUse`** -- Multiple credential instances are issued, each intended for a single presentation. After presentation, the instance is discarded and cannot be reused.

**Batch Configuration:**

When using `oneTimeUse` policy, the batch size is configured via `CreateDocumentSettings`:

```kotlin
// File: core-logic/src/main/java/eu/europa/ec/corelogic/config/DocumentIssuanceConfig.kt

data class DocumentIssuanceConfig(
    val defaultRule: DocumentIssuanceRule,
    val documentSpecificRules: Map<DocumentIdentifier, DocumentIssuanceRule>
) {
    fun getRuleForDocument(documentIdentifier: DocumentIdentifier?): DocumentIssuanceRule =
        documentSpecificRules[documentIdentifier] ?: defaultRule
}

data class DocumentIssuanceRule(
    val policy: CredentialPolicy,
    val numberOfCredentials: Int,
)
```

The `numberOfCredentials` parameter specifies how many credential instances to request. For PID (Personal Identification Document) credentials, a typical configuration is 10 instances, providing a pool for presentations without requiring frequent re-issuance.

**Default Configuration (from WalletKitConfig):**

```swift
// File: Modules/logic-core/Sources/Config/WalletKitConfig.swift

var documentIssuanceConfig: DocumentIssuanceConfig {
    DocumentIssuanceConfig(
      defaultRule: DocumentIssuanceRule(
        policy: .rotateUse,
        numberOfCredentials: 1
      ),
      documentSpecificRules: [
        DocumentTypeIdentifier.mDocPid: DocumentIssuanceRule(
          policy: .oneTimeUse,
          numberOfCredentials: 10
        ),
        DocumentTypeIdentifier.sdJwtPid: DocumentIssuanceRule(
          policy: .oneTimeUse,
          numberOfCredentials: 10
        )
      ]
    )
  }
```

When batch issuance is requested, the SDK generates the specified number of key pairs, constructs a proof JWT for each, and sends a single credential request with the `proofs` (plural) field containing all JWTs.

---

## EUDI iOS (Reference Implementation)

### Draft Version

EUDI iOS uses **OID4VCI v1.0 (Final)**, matching the Android implementation. The SDK constructs credential requests using `credential_configuration_id`.

### Credential Request API

The EudiWalletKit SDK provides two issuance methods:

**1. `issueDocumentsByOfferUrl()`**

Used for offer-based issuance. The SDK resolves the offer URL, parses the credential offer, and constructs credential requests.

```swift
issueDocumentsByOfferUrl(
    _ offerUrl: String,
    config: DocumentIssuanceConfig
) async throws -> OfferResultPartialState
```

The method returns an `OfferResultPartialState` that indicates the outcome of the issuance (see [09 -- Issuer Response](../09-issuer-response/)).

**2. `issueDocument()`**

Used for wallet-initiated issuance or when requesting a specific credential type:

```swift
issueDocument(
    _ credentialConfigurationId: String,
    config: DocumentIssuanceConfig
) async throws -> IssuanceResult
```

### Batch Issuance

The iOS SDK supports the same credential policies as Android:

**`DocumentIssuanceConfig`:**

```swift
// File: Modules/logic-core/Sources/Config/DocumentIssuanceConfig.swift

struct DocumentIssuanceConfig {
  let defaultRule: DocumentIssuanceRule
  let documentSpecificRules: [DocumentTypeIdentifier: DocumentIssuanceRule]

  func rule(for documentIdentifier: DocumentTypeIdentifier?) -> DocumentIssuanceRule {
    guard let documentIdentifier, let rule = documentSpecificRules[documentIdentifier] else {
      return defaultRule
    }
    return rule
  }
}

struct DocumentIssuanceRule {
  let policy: CredentialPolicy
  let numberOfCredentials: Int
}
```

**Credential Policies:**

- **`.rotateUse`** (default) -- Single credential, batch size 1. The credential is reused across presentations.
- **`.oneTimeUse`** -- Multiple credential instances. For PID credentials, the typical batch size is 10.

The batch configuration mirrors the Android implementation, reflecting the shared specification compliance (v1.0 Final) and aligned design decisions across the EUDI reference wallet platforms.

---

## Procivis ONE

### Draft Version: Multi-Draft Support

Procivis ONE is the only implementation in this analysis that supports **multiple OID4VCI draft versions simultaneously**. The Core configuration declares support for:

| Protocol Identifier | Description |
|---------------------|-------------|
| `OPENID4VCI_DRAFT13` | Standard OID4VCI Draft 13 |
| `OPENID4VCI_DRAFT13_SWIYU` | Draft 13 with Swiss swiyu-specific extensions |
| `OPENID4VCI_FINAL1` | OID4VCI v1.0 Final |
| `OPENID4VCI_FINAL1_HAIP` | v1.0 Final with High Assurance Interoperability Profile |

This is a **significant differentiator**. While EUDI wallets target a single specification version, Procivis ONE can interoperate with issuers running any of these versions. The protocol version is selected based on the issuer's capabilities, detected during metadata resolution.

### Protocol Selection

The Core SDK determines which protocol version to use based on the issuer's metadata and the credential offer:

1. The offer URL or metadata endpoint is resolved.
2. The SDK inspects the metadata structure and field naming to determine the issuer's draft version.
3. The appropriate protocol handler is selected from the configured set.
4. The credential request is constructed using the field names and semantics of the detected version.

For Draft 13 issuers, the request uses `credential_identifier`. For v1.0 Final issuers, it uses `credential_configuration_id`. This mapping is handled internally by the SDK.

### Private Encryption Parameters

Each protocol variant in the Procivis configuration declares its own **credential response encryption parameters**. This allows different encryption algorithms and key types to be used with different issuer versions:

```
OPENID4VCI_DRAFT13: { encryption params for Draft 13 }
OPENID4VCI_FINAL1: { encryption params for v1.0 Final }
OPENID4VCI_FINAL1_HAIP: { encryption params for HAIP profile }
```

### swiyu-Specific Extensions

The `OPENID4VCI_DRAFT13_SWIYU` variant includes extensions specific to the Swiss swiyu trust infrastructure. These may include additional metadata fields, modified endpoint behaviors, or supplementary validation requirements defined by the swiyu ecosystem. This demonstrates how a base specification draft can be profiled for a national or regional deployment.

### Credential Request Construction

The One Core SDK constructs credential requests internally via the native bridge. The React Native application layer invokes issuance through high-level SDK methods and does not directly assemble request payloads. The SDK selects the appropriate field names, proof format, and encryption parameters based on the detected protocol version.

### Batch Issuance

Procivis ONE issues a **single credential per request**. Batch issuance (multiple instances in a single request) is not a current feature of the Procivis issuance flow. Each credential is requested individually, which simplifies the protocol handling but means one-time-use credential policies require multiple round-trips.

---

## Affinidi

### Flow Type

Affinidi uses the **Pre-Authorized Code flow** exclusively. The credential request follows the token exchange, with no authorization code flow or wallet-initiated issuance.

### Credential Request Sequence

1. The user claims a credential via a claim link (the credential offer).
2. The Vault exchanges the pre-authorized code for an access token.
3. The Vault constructs a credential request with the proof of possession.
4. The request is sent to the issuer's credential endpoint.
5. The issuer validates the proof, access token, and credential type.
6. The issuer returns the Verifiable Credential.

### Single Credential Per Offer

Each claim link in the Affinidi ecosystem is **single-use** and corresponds to a single credential. There is no batch issuance -- each claim link, when claimed, produces exactly one credential. If multiple credentials are needed, multiple claim links must be generated and claimed individually.

This design reflects the Affinidi model where credential offers are generated for specific claim scenarios (e.g., a specific document or verification result) rather than bulk provisioning.

### Credential Format

Affinidi primarily issues **JSON-LD Verifiable Credentials**. The credential request includes the credential type information, and the issuer returns a full JSON-LD VC with embedded proof (EcdsaSecp256k1Signature2019).

### Service Validation

On the issuer side, the Affinidi service performs:

1. Access token validation.
2. Proof of possession verification (signature and challenge validation).
3. Credential type lookup and claims assembly.
4. VC construction with issuer signature.
5. Offer status update (marked as claimed, preventing re-use).

---

## Summary

The four implementations approach credential requests from different positions in the specification landscape. The EUDI wallets (Android and iOS) align on v1.0 Final with batch issuance support and configurable credential policies. Procivis ONE uniquely supports multiple draft versions, enabling interoperability across the fragmented ecosystem. Affinidi operates a simpler model with single-credential pre-authorized flows and JSON-LD credentials. The choice of specification version has direct implications for interoperability, compliance, and the features available in the issuance flow.
