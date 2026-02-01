# Issuer Metadata -- Comparative Analysis

This document compares how the four implementations handle issuer metadata: discovery patterns, multi-issuer support, caching strategies, credential format diversity, and the trade-offs between direct access and abstraction.

---

## Metadata Access Patterns

The implementations fall into three distinct categories for how they interact with issuer metadata.

### Direct Access (EUDI Android, EUDI iOS)

Both EUDI wallets fetch, parse, and map issuer metadata directly in their application code. The wallet's controller layer (`WalletCoreDocumentsController` on Android, `WalletKitController` on iOS) calls the SDK to retrieve `CredentialIssuerMetadata`, then maps it to domain-specific objects.

**Characteristics:**
- Application code has full visibility into the metadata structure.
- Format detection, display name resolution, and PID flag computation happen in application-controlled code.
- Developers can inspect, log, and debug metadata at every stage.
- Changes to the metadata structure may require updates to the mapping logic.

### SDK-Abstracted (Procivis ONE)

The One Core SDK handles metadata retrieval and processing internally. The application layer accesses high-level configuration through `useCoreConfig()` and receives normalized credential data during issuance, but does not interact with the raw metadata.

**Characteristics:**
- Application code is simpler -- no metadata parsing or format detection logic.
- Metadata-related changes are absorbed by SDK updates without app code changes.
- Debugging metadata issues requires SDK-level tooling or logging.
- The app cannot display or act on metadata details that the SDK does not surface.

### Configuration-Driven (Affinidi)

Affinidi replaces runtime metadata discovery with pre-configured issuance configurations. The issuer defines schemas and parameters through the Credential Issuance Service API before any credential is issued. The Affinidi Vault receives effective metadata as part of the claim link resolution.

**Characteristics:**
- No dynamic discovery -- metadata is established at configuration time.
- Schema validation happens at offer creation, not at credential request.
- Changes require updating the issuance configuration through the service API.
- Simpler for issuers who want a managed service rather than hosting metadata endpoints.

### Comparison Table

| Aspect | Direct Access (EUDI) | SDK-Abstracted (Procivis) | Configuration-Driven (Affinidi) |
|--------|---------------------|---------------------------|----------------------------------|
| App-layer visibility | Full | Limited | None (Vault internal) |
| Developer control | High | Low | N/A (issuer-side config) |
| Debugging ease | High | Low | Medium (service logs) |
| Maintenance burden | Higher (mapping code) | Lower (SDK handles) | Lowest (service manages) |
| Flexibility | High (any metadata field accessible) | Medium (SDK-exposed fields only) | Low (schema-constrained) |
| Coupling to spec changes | Direct | Indirect (via SDK) | Indirect (via service) |

---

## Multi-Issuer Support

| Implementation | Model | Mechanism |
|----------------|-------|-----------|
| EUDI Android | Multi-issuer with iteration | `getScopedDocuments()` iterates all configured issuers, calling `getIssuerMetadata()` for each |
| EUDI iOS | Multi-issuer with iteration | Same pattern as Android |
| Procivis ONE | SDK-managed | Core SDK manages issuer relationships internally |
| Affinidi | Per-configuration | Each issuance configuration is tied to a specific issuer setup |

### Analysis

The EUDI wallets provide the most transparent multi-issuer support. The application layer explicitly iterates over configured issuers, fetches metadata from each, and aggregates the results into a unified list of available credentials grouped by issuer. This enables a "credential store" UX where the user can browse all available credentials across all issuers.

The aggregation pattern creates a fan-out network request pattern: if the wallet has N configured issuers, it makes N metadata requests during scoped document discovery. This has latency implications that the iOS wallet mitigates with metadata caching.

Procivis ONE's multi-issuer support is opaque to the app layer. The core SDK may maintain relationships with multiple issuers, but the app does not control or observe the iteration pattern.

Affinidi's model is inherently single-issuer per configuration. Each Credential Issuance Service configuration represents one issuer with its own schemas and signing wallet. Supporting multiple issuers requires multiple configurations, managed independently.

---

## Metadata Caching Strategies

| Implementation | Caching Approach | Persistence | Invalidation |
|----------------|-----------------|-------------|--------------|
| EUDI Android | Not observed in reference code | N/A | N/A |
| EUDI iOS | In-memory cache (`cacheIssuerMetadata: true`) | Session-only (not persisted across app launches) | Time-based or explicit refresh |
| Procivis ONE | Handled by One Core SDK | Unknown (SDK internal) | Unknown (SDK internal) |
| Affinidi | Service-managed | Service-side | Configuration update |

### Analysis

Metadata caching is important for two reasons:

1. **Performance** -- Metadata requests add latency to offer resolution and credential discovery. Caching avoids redundant network requests.
2. **Offline resilience** -- Cached metadata allows the wallet to display credential information and potentially complete some protocol steps without network connectivity.

The EUDI iOS wallet is the only implementation with observable caching behavior at the app layer. The `cacheIssuerMetadata: true` configuration flag enables in-memory caching, which helps during the common pattern of fetching metadata multiple times during a single issuance flow (once during offer resolution, once during credential request construction).

The EUDI Android reference code does not show explicit caching, which means each call to `getIssuerMetadata()` may result in a network request. In practice, the underlying SDK or HTTP client may provide transparent caching, but this is not controlled at the application layer.

The lack of persistent caching in either EUDI wallet means that metadata must be re-fetched after each app launch, which affects cold-start performance for credential discovery.

---

## Credential Format Diversity

| Format | EUDI Android | EUDI iOS | Procivis ONE | Affinidi |
|--------|-------------|----------|--------------|----------|
| `mso_mdoc` (ISO 18013-5) | Yes | Yes | Yes (via SDK) | Not primary |
| `vc+sd-jwt` (SD-JWT VC) | Yes | Yes | Yes (via SDK) | Yes |
| `jwt_vc_json` (W3C JWT VC) | Not observed | Not observed | Yes (via SDK) | Yes |

### Analysis

The EUDI wallets focus on the two formats most relevant to the EU Digital Identity framework: `mso_mdoc` for ISO-standardized documents (driving licenses, national IDs) and `vc+sd-jwt` for newer credential types that benefit from selective disclosure.

Procivis ONE, through its core SDK, supports a broader range of formats because the SDK handles format-specific processing internally. The app layer receives normalized credential data regardless of the underlying format.

Affinidi supports SD-JWT and JWT-based formats, aligning with the W3C Verifiable Credentials ecosystem. The mso_mdoc format is less prominent in Affinidi's documentation, reflecting its focus on web and cloud-native credential flows rather than ISO mobile document standards.

The format support differences reflect the target markets of each implementation:
- **EUDI:** EU regulatory environment (eIDAS 2.0), where mso_mdoc and sd-jwt are mandated.
- **Procivis ONE:** Enterprise deployments requiring format flexibility.
- **Affinidi:** Web-native credential issuance, where JWT-based formats are natural.

---

## Strengths and Weaknesses

### EUDI Android

| Strengths | Weaknesses |
|-----------|------------|
| Direct metadata access gives full control over mapping and presentation | No observable caching -- potential for redundant network requests |
| Clear domain model (`ScopedDocumentDomain`) with typed fields | Metadata processing coupled to application code -- spec changes require app updates |
| Multi-issuer iteration is explicit and debuggable | Fan-out requests to N issuers create latency during discovery |
| PID flag detection enables regulatory compliance logic | Format support limited to two types (mso_mdoc, vc+sd-jwt) |

### EUDI iOS

| Strengths | Weaknesses |
|-----------|------------|
| Metadata caching reduces redundant network requests | Cache is session-only; no persistence across launches |
| Same direct access model as Android with Swift-native patterns | Same coupling to metadata structure as Android |
| Locale-aware display name resolution | Same format limitation as Android |
| Additional credential types (IBAN, HIID) show extensibility | Fan-out request pattern same as Android |

### Procivis ONE

| Strengths | Weaknesses |
|-----------|------------|
| App layer is decoupled from metadata structure -- SDK absorbs spec changes | No visibility into metadata at the app layer |
| Broad format support via core SDK | Debugging metadata issues requires SDK-level investigation |
| Multi-protocol support (Draft 13, Final 1, HAIP) handled transparently | Cannot customize metadata processing or display logic |
| Simpler app code -- no metadata parsing required | Dependent on SDK release cycle for metadata-related fixes |

### Affinidi

| Strengths | Weaknesses |
|-----------|------------|
| Schema validation at configuration time catches data issues early | No dynamic discovery -- metadata changes require reconfiguration |
| Managed service eliminates need to host metadata endpoints | Less flexibility for issuers who want fine-grained metadata control |
| Clear separation: issuer configures, service manages, Vault consumes | Vault's metadata processing is opaque (closed source) |
| Simpler for issuers who want turnkey credential issuance | Not suitable for scenarios requiring custom metadata endpoints |

---

## Decision Framework

| Factor | Best Choice | Reasoning |
|--------|------------|-----------|
| Need full control over metadata processing | EUDI (Android or iOS) | Direct access to raw metadata with application-controlled mapping |
| Want minimal app-layer metadata code | Procivis ONE | Core SDK handles all metadata interactions |
| Building a managed issuance service | Affinidi | Configuration-driven approach eliminates metadata endpoint hosting |
| Require metadata caching | EUDI iOS | Only implementation with observable, configurable caching |
| Need broad format support | Procivis ONE | Core SDK supports multiple formats transparently |
| EU regulatory environment | EUDI (either platform) | PID detection, eIDAS-aligned credential types |
| Multi-issuer credential discovery | EUDI (either platform) | Explicit iteration with `getScopedDocuments()` |
| Rapid issuer onboarding | Affinidi | Schema + configuration API, no metadata endpoint setup |
