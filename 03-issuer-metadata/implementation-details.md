# Issuer Metadata -- Implementation Details

This document details how each wallet implementation retrieves, parses, and uses issuer metadata. It covers code paths, format handling, caching strategies, and the mapping between OID4VCI metadata structures and internal domain models.

---

## EUDI Android

### Metadata Retrieval

The EUDI Android wallet retrieves issuer metadata through a layered architecture that separates concerns between the UI, business logic, and SDK layers.

**Primary code path for scoped document discovery:**

```
WalletCoreDocumentsController.getScopedDocuments()
  -> For each configured issuer:
    -> OpenId4VciManager.getIssuerMetadata(issuerUrl)
      -> HTTP GET {issuerUrl}/.well-known/openid-credential-issuer
      -> Parse response as CredentialIssuerMetadata
  -> Map to List<ScopedDocumentDomain>
```

`getScopedDocuments()` iterates over all issuers declared in the wallet's configuration. For each issuer, it calls `OpenId4VciManager.getIssuerMetadata()`, which performs the HTTP request to the well-known endpoint and deserializes the response into a `CredentialIssuerMetadata` object.

```kotlin
// File: core-logic/src/main/java/eu/europa/ec/corelogic/controller/WalletCoreDocumentsController.kt

override suspend fun getScopedDocuments(locale: Locale): FetchScopedDocumentsPartialState {
    return withContext(dispatcher) {
        runCatching {

            val metadata: Map<String, CredentialIssuerMetadata> =
                openId4VciManagers.mapValues { (_, manager) ->
                    manager.getIssuerMetadata().getOrThrow()
                }

            val documents: List<ScopedDocumentDomain> =
                metadata.flatMap { (issuer, meta) ->
                    meta.credentialConfigurationsSupported.map { (id, config) ->

                        val name: String = config.credentialMetadata.getLocalizedDisplayName(
                            userLocale = locale,
                            fallback = id.value
                        )

                        val isPid = when (config) {
                            is MsoMdocCredential -> config.docType.toDocumentIdentifier() == DocumentIdentifier.MdocPid
                            is SdJwtVcCredential -> config.type.toDocumentIdentifier() == DocumentIdentifier.SdJwtPid
                            else -> false
                        }

                        val formatType = when (config) {
                            is MsoMdocCredential -> config.docType
                            is SdJwtVcCredential -> config.type
                            else -> null
                        }

                        ScopedDocumentDomain(
                            name = name,
                            configurationId = id.value,
                            credentialIssuerId = issuer,
                            formatType = formatType,
                            isPid = isPid
                        )
                    }
                }

            if (documents.isNotEmpty()) {
                FetchScopedDocumentsPartialState.Success(documents = documents)
            } else {
                FetchScopedDocumentsPartialState.Failure(errorMessage = genericErrorMessage)
            }
        }
    }.getOrElse {
        FetchScopedDocumentsPartialState.Failure(
            errorMessage = it.localizedMessage ?: genericErrorMessage
        )
    }
}
```

### Metadata Mapping

The raw `CredentialIssuerMetadata` is mapped to `ScopedDocumentDomain`, the wallet's internal domain model for representing available credentials. The mapping extracts:

| Source (CredentialIssuerMetadata) | Target (ScopedDocumentDomain) | Notes |
|-----------------------------------|-------------------------------|-------|
| Configuration key | `configId` | The credential configuration identifier (e.g., `org.iso.18013.5.1.mDL`) |
| `display[0].name` (localized) | Display name | Resolved against device locale; falls back to first entry |
| `format` | Format enum (`MsoMdoc` / `SdJwtVc`) | Determines downstream processing |
| Configuration key pattern | `isPid` flag | Set to `true` if the configuration ID matches known PID identifiers |
| `credential_issuer` | Grouped by issuer | Results are grouped by issuer URL for UI presentation |

### Format Detection

The wallet recognizes and handles two credential formats from the metadata:

- **`MsoMdoc`** -- Mapped from `format: "mso_mdoc"`. Used for ISO 18013-5 documents.
- **`SdJwtVc`** -- Mapped from `format: "vc+sd-jwt"`. Used for SD-JWT Verifiable Credentials.

Format detection is performed via pattern matching on the `format` field of each credential configuration. Unrecognized formats are logged and skipped.

### Supported Credential Types

Based on the configured issuers and their metadata, the EUDI Android reference wallet supports the following credential configuration IDs:

| Configuration ID | Format | Description |
|-----------------|--------|-------------|
| `org.iso.18013.5.1.mDL` | mso_mdoc | Mobile Driving License |
| `eu.europa.ec.eudi.pid.1` | mso_mdoc | Person Identification Data (mdoc) |
| `eu.europa.ec.eudi.pid.1` | vc+sd-jwt | Person Identification Data (SD-JWT) |
| `eu.europa.ec.eudi.tax.1` | mso_mdoc | Tax credential |
| `eu.europa.ec.eudi.age_over_18.1` | mso_mdoc | Age verification (over 18) |
| `eu.europa.ec.eudi.cor.1` | mso_mdoc | Certificate of Residence |

Each of these has a corresponding entry in the issuer's `credential_configurations_supported`, with format-specific fields (doctype for mso_mdoc, vct for vc+sd-jwt).

### PID Detection

The `isPid` flag is set based on matching the configuration ID against known PID patterns. This flag drives the PID requirement validation in `DocumentOfferInteractor` (see [02 -- Credential Offer: Implementation Details](../02-credential-offer/implementation-details.md)).

---

## EUDI iOS

### Metadata Retrieval

The iOS wallet follows the same architectural pattern as Android, with Swift equivalents.

**Primary code path:**

```
WalletKitController.getScopedDocuments()
  -> For each configured issuer:
    -> EudiWallet.getIssuerMetadata(issuerUrl)
      -> HTTP GET {issuerUrl}/.well-known/openid-credential-issuer
      -> Parse response as CredentialIssuerMetadata
  -> Map to [ScopedDocumentDomain]
```

### Metadata Caching

The EUDI iOS wallet supports metadata caching through a configuration flag:

```
cacheIssuerMetadata: true
```

The actual VCI configuration with caching enabled:

```swift
// File: Modules/logic-core/Sources/Config/WalletKitConfig.swift

var vciConfig: [String: OpenId4VciConfiguration] {

    let openId4VciConfigurations: [OpenId4VciConfiguration] = {
      switch configLogic.appBuildVariant {
      case .DEMO:
        return [
          .init(
            credentialIssuerURL: "https://issuer.eudiw.dev",
            clientId: "wallet-dev",
            keyAttestationsConfig: .init(walletAttestationsProvider: walletKitAttestationProvider),
            authFlowRedirectionURI: URL(string: "eu.europa.ec.euidi://authorization")!,
            usePAR: true,
            useDpopIfSupported: true,
            cacheIssuerMetadata: true
          ),
          // ... additional issuer configurations
        ]
      // ...
      }
    }()

    return openId4VciConfigurations.reduce(
      into: [String: OpenId4VciConfiguration]()
    ) { dict, config in
      guard
        let issuer = config.credentialIssuerURL,
        let url = URL(string: issuer),
        let host = url.host
      else {
        return
      }
      dict[host] = config
    }
  }
```

When enabled, the wallet caches the fetched `CredentialIssuerMetadata` and reuses it for subsequent requests to the same issuer. This avoids redundant network requests during:
- Offer resolution (when the wallet fetches metadata to validate the offer)
- Scoped document discovery (when the user browses available credentials)
- Credential request construction (when the wallet needs format and proof type information)

The cache is in-memory and does not persist across app launches. Cache invalidation is time-based or triggered by explicit refresh actions.

### Format Detection

Format detection uses a Swift `switch` statement on the credential configuration's format:

```
switch configuration.format {
  case .msoMdoc:
    // Handle ISO 18013-5 format
    // Extract doctype, claims by namespace
  case .sdJwtVc:
    // Handle SD-JWT VC format
    // Extract vct, claims
  default:
    // Unknown format, skip
}
```

The pattern matching is exhaustive for the two supported formats and explicitly handles the default case by logging and skipping unsupported formats.

### Supported Credential Types

The EUDI iOS reference wallet supports the same core credential types as Android, plus additional types reflecting its configuration:

| Configuration ID | Format | Description |
|-----------------|--------|-------------|
| `org.iso.18013.5.1.mDL` | mso_mdoc | Mobile Driving License |
| `eu.europa.ec.eudi.pid.1` | mso_mdoc | Person Identification Data (mdoc) |
| `eu.europa.ec.eudi.pid.1` | vc+sd-jwt | Person Identification Data (SD-JWT) |
| `urn:eu.europa.ec.eudi:iban:1` | mso_mdoc | IBAN credential |
| `hiid:1` | mso_mdoc | Health Insurance ID |

The presence of `urn:eu.europa.ec.eudi:iban:1` and `hiid:1` reflects the iOS wallet's configuration for additional EU credential types beyond the Android reference set.

### Localized Display Name Resolution

The iOS wallet resolves display names by matching the device's current locale against the `display` array entries:

1. Extract the device's preferred language (e.g., `en`).
2. Search the `display` array for an entry whose `locale` starts with the device language.
3. If found, use that entry's `name`.
4. If not found, use the first entry's `name` (regardless of its `locale`).

---

## Procivis ONE

### SDK Abstraction

Procivis ONE takes a fundamentally different approach to issuer metadata: **the application layer does not interact with metadata endpoints directly**. The One Core SDK handles all OID4VCI protocol interactions internally, including metadata retrieval, parsing, and caching.

**Application-level code path:**

```
useCoreConfig()
  -> Returns core configuration object
  -> Includes issuance protocol configurations
```

The `useCoreConfig()` hook provides the application with a configuration object that includes high-level information about supported protocols and features. However, the detailed issuer metadata (credential configurations, proof types, display properties) is consumed internally by the core SDK during the invitation handling and credential issuance flows.

### Issuance Protocol Configuration

The app layer declares issuance protocol configurations in `app-navigator.tsx`. These configurations tell the core SDK which OID4VCI versions and profiles to support:

```
Issuance protocol configs (app-navigator.tsx)
  -> OID4VCI Draft 13
  -> OID4VCI v1.0 Final
  -> HAIP profile
```

The specific protocol version used for a given issuance interaction is determined by the core SDK based on the issuer's metadata and the offer's characteristics.

### Implications of Abstraction

Because the metadata layer is abstracted:

- **The app cannot display granular metadata** (e.g., specific proof types or cryptographic requirements) until the core SDK surfaces it through its API.
- **Metadata caching is handled internally** by the core SDK. The app layer has no visibility into or control over cache behavior.
- **Format detection and handling** happens within the core SDK. The app receives credential data in a normalized format regardless of whether the underlying format is mso_mdoc, jwt_vc_json, or vc+sd-jwt.
- **Error handling** for metadata-related failures (unreachable endpoint, invalid metadata, unsupported format) is surfaced as generic errors from the core SDK rather than as specific metadata errors.

This design prioritizes simplicity for app developers at the cost of flexibility and debugging visibility.

---

## Affinidi

### Configuration-Driven Approach

Affinidi does not use the dynamic `/.well-known/openid-credential-issuer` discovery pattern in the same way as the EUDI wallets. Instead, issuer metadata is established through **pre-configuration**.

Before any credential can be issued, the issuer service must:

1. **Create a signing wallet** -- A managed wallet that holds the issuer's signing keys.
2. **Define credential schemas** -- JSON schemas that define the data structure of each credential type (claim names, types, required fields).
3. **Create an issuance configuration** -- Links the signing wallet to the supported schemas and defines issuance parameters.

```
Affinidi Credential Issuance Service
  -> Issuance Configuration
    -> Signing Wallet (holds issuer's keys)
    -> Supported Schemas (define credential structure)
    -> Issuance Parameters (revocability, claim modes)
```

### Schema-Driven Validation

When the issuer creates a credential offer (via the Credential Issuance Service API), the service validates the provided user claims against the schema defined in the issuance configuration:

- All required fields must be present.
- Field types must match the schema definition.
- Additional fields not in the schema are rejected.

This validation happens at offer creation time, before the holder is involved. The EUDI wallets perform this validation at credential request time (on the issuer side) rather than at offer creation.

### How Metadata Reaches the Vault

The Affinidi Vault (holder's wallet) does receive OID4VCI-compatible metadata when resolving a claim link, but this metadata is generated by the Credential Issuance Service based on the pre-configured issuance configuration rather than being a static document at a well-known endpoint.

The effective flow:

```
Issuer (via TDK)
  -> Creates issuance configuration (schemas, wallet, parameters)
  -> Creates credential offer (user claims, claim mode)
  -> Returns claim link

Holder (Affinidi Vault)
  -> Opens claim link
  -> Credential Issuance Service returns offer + metadata
  -> Vault processes metadata (formats, proofs, display)
  -> Vault proceeds with credential request
```

### Implications of Configuration-Driven Metadata

| Aspect | Dynamic Discovery (EUDI) | Configuration-Driven (Affinidi) |
|--------|--------------------------|----------------------------------|
| Issuer flexibility | Issuer can update metadata at any time; wallets discover changes automatically | Configuration must be updated through the service API |
| Validation timing | Metadata validated at runtime by each wallet | Schema validation at offer creation time |
| Multi-issuer | Wallet queries multiple issuers independently | Each issuer has its own configuration in the service |
| Offline capability | Cached metadata can be used offline | Requires service connectivity |
| Developer experience | Wallet developers must handle metadata parsing | Schema and configuration defined once; service handles the rest |

---

## Summary: Metadata Access Patterns

| Aspect | EUDI Android | EUDI iOS | Procivis ONE | Affinidi |
|--------|-------------|----------|--------------|----------|
| **Discovery method** | `/.well-known/openid-credential-issuer` | `/.well-known/openid-credential-issuer` | Abstracted by One Core SDK | Configuration-driven via service API |
| **App-layer access** | Direct -- maps to domain model | Direct -- maps to domain model | Indirect -- via `useCoreConfig()` | N/A -- Vault internal |
| **Caching** | Not observed in reference code | `cacheIssuerMetadata: true` | Handled by core SDK | Service-managed |
| **Format support** | MsoMdoc, SdJwtVc | MsoMdoc, SdJwtVc | Determined by core SDK | Schema-defined |
| **Multi-issuer** | Iterates configured issuers | Iterates configured issuers | Core SDK manages | Per-configuration |
| **Localization** | Device locale matching | Device locale matching | Core SDK handles | Service-provided |
