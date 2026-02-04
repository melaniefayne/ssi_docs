# OID4VCI Draft 13 vs. v1.0 (Final) -- Deep Comparison

This document provides a thorough comparison of the two OID4VCI specification versions that matter most to the current ecosystem: **Draft 13** (the version widely deployed before ratification) and **v1.0 Final** (the ratified standard, published September 2025). Understanding the differences between these versions is essential for building interoperable credential issuance systems, choosing the right version for a deployment, and migrating between versions.

---

## Context: Why Two Versions Matter

The OID4VCI specification was developed as a series of editor's drafts by the OpenID Foundation. Draft 13 (circa 2023) became the de facto implementation target for early adopters, including the initial EUDI Reference Wallet implementations and the Swiss swiyu ecosystem. Many production and pilot deployments were built against Draft 13 before the specification was ratified.

When v1.0 Final was ratified in September 2025, it incorporated lessons learned from Draft 13 deployments and introduced several breaking changes. The result is a split ecosystem:

- **Draft 13 deployments** are operational and serving real users but are based on a non-final specification.
- **v1.0 Final deployments** are standards-compliant and future-proof but must interoperate with a world where many issuers still run Draft 13.
- **Multi-draft implementations** (notably Procivis ONE) support both versions simultaneously, bridging the gap at the cost of implementation complexity.

---

## Credential Identification

The most visible breaking change between Draft 13 and v1.0 Final is how the wallet identifies which credential it is requesting.

### Draft 13

Draft 13 offers two mechanisms for credential identification in the request body:

**Option A: `credential_identifier`**

```json
{
  "credential_identifier": "eu.europa.ec.eudi.pid.1"
}
```

The `credential_identifier` is a value provided by the issuer in the credential offer or token response. It references a specific credential configuration without requiring the wallet to include format details.

**Option B: Format + Type**

```json
{
  "format": "mso_mdoc",
  "doctype": "org.iso.18013.5.1.mDL"
}
```

The wallet specifies the credential format and format-specific type identifier (`doctype` for mDoc, `vct` for SD-JWT VC, `credential_definition` for JSON-LD VC). This approach does not require a prior identifier from the offer.

### v1.0 Final

v1.0 Final replaces `credential_identifier` with `credential_configuration_id`:

```json
{
  "credential_configuration_id": "eu.europa.ec.eudi.pid.1"
}
```

The `credential_configuration_id` serves the same purpose as `credential_identifier` but with a clearer name that explicitly references the issuer's `credential_configurations_supported` metadata. The format + type approach is still available as an alternative.

### Breaking Change

| Aspect | Draft 13 | v1.0 Final |
|--------|----------|------------|
| Identifier field | `credential_identifier` | `credential_configuration_id` |
| Source | Credential offer or token response | Credential offer (`credential_configuration_ids` array) |
| Metadata reference | Implicit | Explicit (matches keys in `credential_configurations_supported`) |

This is a **wire-level breaking change**: a wallet sending `credential_identifier` to a v1.0 Final issuer will receive an error, and vice versa. Implementations that must support both versions need to detect the issuer's version and use the correct field name.

---

## Credential Offer Structure

The credential offer structure also changed between versions.

### Draft 13

```json
{
  "credential_issuer": "https://issuer.example.com",
  "credentials": [
    "eu.europa.ec.eudi.pid.1"
  ],
  "grants": { ... }
}
```

Draft 13 uses a `credentials` array in the offer, where each entry is either a string (credential identifier) or an object with format and type details.

### v1.0 Final

```json
{
  "credential_issuer": "https://issuer.example.com",
  "credential_configuration_ids": [
    "eu.europa.ec.eudi.pid.1"
  ],
  "grants": { ... }
}
```

v1.0 Final renames the array to `credential_configuration_ids` and requires each entry to be a string that maps to a key in the issuer's `credential_configurations_supported` metadata.

---

## Request Payload Comparison

The following table compares the full credential request payload between Draft 13 and v1.0 Final.

| Field | Draft 13 | v1.0 Final | Notes |
|-------|----------|------------|-------|
| `credential_identifier` | Supported | **Removed** | Renamed to `credential_configuration_id` |
| `credential_configuration_id` | Not present | Supported | Replaces `credential_identifier` |
| `format` | Supported (alternative to identifier) | Supported (alternative to config ID) | Unchanged semantics |
| `doctype` | Supported (with `format: "mso_mdoc"`) | Supported | Unchanged |
| `vct` | Supported (with `format: "vc+sd-jwt"`) | Supported | Unchanged |
| `credential_definition` | Supported (with JSON-LD formats) | Supported | Unchanged |
| `proof` | Supported | Supported | Structure unchanged |
| `proofs` | Supported | Supported | Structure unchanged |
| `credential_response_encryption` | Supported | Supported | Encryption parameters for response |

---

## Notification Endpoint

### Draft 13

The notification endpoint is **optional** in Draft 13. Issuers may or may not support it, and wallets should check the issuer metadata for `notification_endpoint` before attempting to send notifications.

### v1.0 Final

The notification endpoint has **mandatory support** in v1.0 Final. Issuers that declare the endpoint must handle notification messages from wallets. The notification mechanism allows the wallet to inform the issuer about credential lifecycle events:

```json
POST /notification
{
  "notification_id": "3fwe98fj",
  "event": "credential_accepted"
}
```

Supported events:

| Event | Meaning |
|-------|---------|
| `credential_accepted` | The wallet accepted and stored the credential. |
| `credential_deleted` | The wallet deleted the credential. |
| `credential_failure` | The wallet failed to process the credential. |

This change improves issuer visibility into credential lifecycle but requires wallets to implement notification sending.

---

## Batch Credential Issuance

### Draft 13

Batch issuance is supported through the `proofs` field (plural). The wallet sends multiple proof JWTs, and the issuer returns multiple credentials. The mechanism is functional but the specification provides limited guidance on batch behavior and error handling for partial batches.

### v1.0 Final

v1.0 Final provides **enhanced batch credential issuance** with clearer semantics:

- Explicit `credentials` array in the response, matching the proofs array by index.
- Defined behavior for partial success (some credentials issued, some failed).
- Clearer guidance on batch size limits and issuer-imposed constraints.

---

## c_nonce Handling

### Draft 13

The `c_nonce` is returned in the token response and optionally rotated in credential responses. The specification allows flexibility in when and how the nonce is refreshed.

### v1.0 Final

v1.0 Final tightens `c_nonce` handling:

- The `c_nonce` may be returned in the token response or in credential error responses.
- The `c_nonce_expires_in` parameter is more consistently defined.
- Nonce rotation in credential responses follows clearer rules: if a new `c_nonce` is returned, the wallet must use it for subsequent requests.

---

## Proof Types

### Draft 13

Draft 13 defines the `jwt` proof type and allows for extensibility. The specification includes the basic JWT proof structure but provides limited guidance on additional proof types.

### v1.0 Final

v1.0 Final refines the proof type framework:

- The `jwt` proof type remains the primary mechanism with unchanged structure.
- The `attestation` proof type is introduced for key attestation-based proofs (relevant for HAIP).
- Clearer extensibility framework for adding new proof types.
- Better defined error responses for proof-related failures.

---

## HAIP Profile Alignment

The High Assurance Interoperability Profile (HAIP) is a profile of OID4VCI designed for high-assurance use cases, particularly eIDAS/EUDI compliance. HAIP is defined on top of v1.0 Final.

### HAIP Requirements (v1.0 Final Only)

| Requirement | Description |
|-------------|-------------|
| **PAR mandatory** | Pushed Authorization Requests (RFC 9126) are required. The wallet must send authorization requests to the PAR endpoint. |
| **DPoP mandatory** | Demonstrating Proof of Possession (RFC 9449) is required for all token and credential requests. |
| **Hardware key attestation** | The wallet must provide attestation that its signing key is stored in certified hardware (TEE, Secure Element, or equivalent). |
| **ES256 required** | The P-256 curve with ECDSA (ES256) is the mandatory algorithm for interoperability. |
| **Credential format restrictions** | HAIP mandates support for mso_mdoc and vc+sd-jwt formats. |

### Impact on Draft 13 Deployments

HAIP is not backported to Draft 13. Deployments that must meet HAIP requirements (including eIDAS/EUDI compliance) must migrate to v1.0 Final. This is the primary technical driver for migration from Draft 13.

---

## Credential Response Encryption

### Draft 13

Credential response encryption is supported through the `credential_response_encryption` parameter in the request. The wallet specifies its encryption key and preferred algorithm, and the issuer encrypts the credential response.

### v1.0 Final

The mechanism is unchanged in structure but the metadata declaration is refined. The issuer's metadata more clearly declares whether response encryption is required, optional, or unsupported, and the algorithm negotiation is better specified.

---

## Full Comparison Table

| Aspect | Draft 13 (circa 2023) | v1.0 Final (September 2025) |
|--------|----------------------|----------------------------|
| **Status** | Editor's draft (superseded) | Ratified standard |
| **Credential identifier field** | `credential_identifier` | `credential_configuration_id` |
| **Offer credentials array** | `credentials` | `credential_configuration_ids` |
| **Notification endpoint** | Optional | Mandatory support |
| **Batch issuance** | Supported (basic) | Supported (enhanced, with partial success) |
| **c_nonce handling** | Flexible | Tightened rules |
| **Proof types** | `jwt` | `jwt`, `attestation`, extensible framework |
| **HAIP alignment** | Not aligned | Aligned (HAIP defined on v1.0) |
| **Response encryption** | Supported | Supported (refined metadata) |
| **PAR** | Optional | Optional (mandatory under HAIP) |
| **DPoP** | Optional | Optional (mandatory under HAIP) |
| **Hardware key attestation** | Not specified | Optional (mandatory under HAIP) |
| **Ecosystem adoption** | Wide (early adopters, swiyu, early EUDI) | Growing (EUDI reference wallet, new deployments) |

---

## Why Choose One Over Another

### When to Use Draft 13

- **Backward compatibility** -- The issuer or verifier ecosystem already runs Draft 13 and migration is not yet planned.
- **swiyu ecosystem** -- The Swiss swiyu trust infrastructure is built on Draft 13 (Procivis ONE supports this as `OPENID4VCI_DRAFT13_SWIYU`).
- **Deployed pilots** -- Existing pilots and production systems running Draft 13 cannot be migrated without coordination across all participants.
- **Interim interoperability** -- The wallet must work with issuers that have not yet upgraded.

### When to Use v1.0 Final

- **Standards compliance** -- The deployment requires adherence to a ratified standard.
- **eIDAS / EUDI compliance** -- The EUDI Architecture Reference Framework mandates v1.0 Final (with HAIP profile) for European Digital Identity Wallet compliance.
- **Future-proofing** -- New deployments should target v1.0 Final to avoid future migration costs.
- **HAIP requirements** -- Hardware key attestation, mandatory PAR, and mandatory DPoP are only available under v1.0 Final + HAIP.
- **Notification support** -- The deployment requires credential lifecycle notifications between wallet and issuer.

### When to Support Both (Multi-Draft)

- **Multi-ecosystem wallets** -- The wallet must interoperate with issuers running different draft versions (the Procivis ONE approach).
- **Migration period** -- During a transition from Draft 13 to v1.0 Final, both versions must be supported simultaneously.
- **Maximum interoperability** -- The wallet prioritizes working with the largest number of issuers, regardless of their specification version.

---

## Migration Path: Draft 13 to v1.0 Final

For implementations migrating from Draft 13 to v1.0 Final, the key changes are:

1. **Rename `credential_identifier` to `credential_configuration_id`** in credential request construction.
2. **Rename `credentials` to `credential_configuration_ids`** in credential offer parsing.
3. **Implement notification endpoint support** (sending notifications to the issuer).
4. **Update c_nonce handling** to follow the tightened rotation rules.
5. **Add HAIP support** if required: implement PAR, DPoP, hardware key attestation, and restrict algorithms to ES256.
6. **Test against v1.0 Final issuers** to verify field naming and endpoint behavior.

The proof structure (`proof` / `proofs` with JWT type) is unchanged between versions, which means the proof construction code from [07 -- Proof Construction](../07-proof-construction/) does not need modification for the migration.
