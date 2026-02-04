# Credential Offer -- Comparative Analysis

This document compares how the four implementations handle credential offers across delivery mechanisms, transport diversity, TxCode handling, regulatory constraints, and flow initiation models.

---

## Offer Delivery Mechanisms

| Mechanism | EUDI Android | EUDI iOS | Procivis ONE | Affinidi |
|-----------|-------------|----------|--------------|----------|
| QR Code | Yes | Yes | Yes | Yes (via claim link) |
| Deep Link | Yes | Yes | Yes | Yes (claim link) |
| Universal Link | Via App Links | Via Universal Links | Yes | N/A |
| NFC | Possible (not in reference code) | Possible (not in reference code) | Not observed | N/A |
| BLE | No | No | Yes | No |
| MQTT | No | No | Yes | No |
| Claim Link (email/SMS) | No | No | No | Yes (primary mechanism) |

### Analysis

The EUDI wallets and Procivis ONE both support the standard QR code and deep link delivery mechanisms. Procivis ONE extends this with BLE and MQTT transports, which enable offer delivery in scenarios where HTTP connectivity is unavailable or where message-broker-mediated communication is preferred.

Affinidi's claim link mechanism is conceptually similar to a deep link but is generated and managed by the cloud service rather than being a direct OID4VCI URI. The claim link resolves to the Affinidi Vault, which then handles the OID4VCI protocol internally.

---

## Transport Diversity

| Implementation | Transport Model | Protocols Supported |
|----------------|----------------|---------------------|
| EUDI Android | HTTP-only | HTTPS for all OID4VCI interactions |
| EUDI iOS | HTTP-only | HTTPS for all OID4VCI interactions |
| Procivis ONE | Multi-transport | HTTP, BLE, MQTT (with priority: MQTT > BLE > HTTP) |
| Affinidi | HTTP-only (cloud) | HTTPS between Vault and Credential Issuance Service |

### Analysis

Procivis ONE's multi-transport architecture is the most distinctive feature in this comparison. The transport detection logic in `getInvitationUrlTransports()` inspects the URL to determine available transports and applies the priority order MQTT > BLE > HTTP.

This design reflects a fundamentally different deployment assumption: Procivis ONE anticipates scenarios where the wallet and issuer may not have direct HTTP connectivity (e.g., offline field operations, IoT device provisioning, air-gapped networks). The EUDI wallets and Affinidi assume persistent internet connectivity.

The trade-off is complexity. Multi-transport support requires:
- Transport negotiation logic in the wallet
- Issuer infrastructure supporting multiple protocols
- Additional testing surface for each transport combination

For most credential issuance scenarios (web-based, mobile-connected), HTTP is sufficient. The multi-transport capability becomes valuable in specialized enterprise and government deployments.

---

## TxCode Handling

| Aspect | EUDI Android | EUDI iOS | Procivis ONE | Affinidi |
|--------|-------------|----------|--------------|----------|
| Numeric input mode | Yes | Yes | Yes | Yes (TX_CODE mode) |
| Text input mode | No | No | Yes | N/A (mode abstracted) |
| Length enforcement | 4-6 digits | Per `length` param | Per `length` param | Managed by service |
| Input debounce | No | 250ms debounce | Not observed | N/A |
| Auto-submit on length | Not observed | Yes (when `length` specified) | Not observed | N/A |

### Analysis

The EUDI Android wallet takes the most restrictive approach to TxCode handling: numeric input only, constrained to 4-6 digits. This aligns with EU regulatory expectations for transaction codes in identity credential issuance but does not conform to the full OID4VCI specification, which defines both `numeric` and `text` input modes.

The EUDI iOS wallet adds a 250ms debounce on the TxCode input field (`OfferCodeView`), which is a UX refinement that prevents rapid re-validation during typing. It also supports auto-submission when the specified length is reached.

Procivis ONE is the most specification-compliant implementation, supporting both `NUMERIC` and `TEXT` input modes as defined by OID4VCI. This broader support enables interoperability with issuers that use alphanumeric transaction codes.

Affinidi abstracts TxCode handling into its TX_CODE claim mode. The transaction code is generated and managed by the Credential Issuance Service, and the Vault prompts the user for input. The implementation details of input validation are internal to the Vault.

---

## PID Requirement

| Implementation | PID Enforcement | Regulatory Basis |
|----------------|----------------|------------------|
| EUDI Android | Yes -- must hold PID or offer must include PID | eIDAS 2.0 |
| EUDI iOS | Yes -- same enforcement | eIDAS 2.0 |
| Procivis ONE | No | N/A |
| Affinidi | No | N/A |

### Analysis

The PID requirement is a significant architectural constraint unique to the EUDI wallets. Before accepting a credential offer, the wallet validates that either:

1. The wallet already contains a valid PID (Person Identification Data) credential, or
2. The current offer includes a PID credential among its offered types.

This means a freshly installed EUDI wallet cannot accept an offer for, say, a mobile driving license or a tax credential until it has first obtained a PID. The PID serves as a foundational identity credential that underpins all subsequent credential issuance.

This constraint has no equivalent in Procivis ONE or Affinidi, which allow any credential to be issued independently. The PID requirement is a policy decision driven by EU regulation rather than a protocol requirement -- the OID4VCI specification itself does not mandate any prerequisite credentials.

**Implication for developers:** If building on the EUDI architecture for non-EU deployments, this validation check can be removed or reconfigured. If building for EU compliance, this check is mandatory and must be preserved.

---

## Wallet-Initiated vs Issuer-Initiated Support

| Implementation | Issuer-Initiated | Wallet-Initiated | Discovery Mechanism |
|----------------|-----------------|------------------|---------------------|
| EUDI Android | Yes (primary) | Yes | `getScopedDocuments()` queries all configured issuers |
| EUDI iOS | Yes (primary) | Yes | `getScopedDocuments()` queries all configured issuers |
| Procivis ONE | Yes (primary) | Limited | Core SDK handles; app layer focused on invitations |
| Affinidi | Yes (exclusive) | No | Offer always originates from Credential Issuance Service |

### Analysis

The EUDI wallets provide the most complete support for both flow types. Their `getScopedDocuments()` mechanism proactively queries all configured issuers to discover available credentials, enabling a "credential store" experience where users can browse and request credentials without waiting for an issuer to create an offer.

Procivis ONE's architecture is primarily oriented toward issuer-initiated flows, where the wallet receives an invitation (offer) and processes it. The One Core SDK may support wallet-initiated flows internally, but the app layer (`invitation-process-screen.tsx`) is structured around processing incoming invitations.

Affinidi's cloud-centric model is exclusively issuer-initiated. The Credential Issuance Service creates offers, and the Vault claims them. There is no mechanism for the Vault to browse available credentials across issuers.

---

## Strengths and Weaknesses

### EUDI Android

| Strengths | Weaknesses |
|-----------|------------|
| Full OID4VCI compliance for core offer flow | TxCode limited to numeric only (4-6 digits) |
| PID enforcement provides regulatory compliance | HTTP-only transport limits deployment scenarios |
| Wallet-initiated discovery via scoped documents | PID requirement may be unnecessarily restrictive for non-EU use |
| Clear separation of concerns (ViewModel / Interactor / Controller / SDK) | |

### EUDI iOS

| Strengths | Weaknesses |
|-----------|------------|
| Dual URI scheme support (`openid-credential-offer`, `haip-vci`) | Same TxCode numeric restriction as Android |
| Universal Link support for seamless web-to-app flow | HTTP-only transport |
| 250ms debounce on TxCode input improves UX | Same PID restriction as Android |
| Same wallet-initiated discovery as Android | |

### Procivis ONE

| Strengths | Weaknesses |
|-----------|------------|
| Multi-transport (HTTP, BLE, MQTT) enables diverse deployment scenarios | Multi-transport adds complexity to testing and deployment |
| Full TxCode spec compliance (numeric + text) | Wallet-initiated flow support is limited at the app layer |
| HTTP redirect following resolves short URLs and redirect chains | App layer depends heavily on One Core SDK; less visibility into protocol details |
| Transport priority system (MQTT > BLE > HTTP) is well-defined | |

### Affinidi

| Strengths | Weaknesses |
|-----------|------------|
| FIXED_HOLDER mode provides cryptographic holder binding stronger than TxCode | No wallet-initiated flow |
| Cloud service abstracts protocol complexity from issuer developers | Vault resolution logic is opaque (closed implementation) |
| Claim mode concept (TX_CODE / FIXED_HOLDER) provides clear security model | Requires cloud service dependency -- no fully offline issuance |
| Revocability is a first-class offer parameter | Limited to HTTP delivery via claim links |

---

## Decision Framework

When choosing an implementation approach for credential offer handling, consider the following factors:

| Factor | Best Choice | Reasoning |
|--------|------------|-----------|
| EU regulatory compliance | EUDI | PID enforcement, eIDAS 2.0 alignment |
| Offline or low-connectivity deployment | Procivis ONE | BLE and MQTT transports |
| Cloud-first, API-driven issuance | Affinidi | Credential Issuance Service abstracts protocol |
| Maximum OID4VCI spec compliance | Procivis ONE | Supports both TxCode modes, multi-transport |
| Wallet-initiated credential discovery | EUDI | `getScopedDocuments()` across multiple issuers |
| Strong holder binding without out-of-band codes | Affinidi | FIXED_HOLDER mode with DID-based binding |
