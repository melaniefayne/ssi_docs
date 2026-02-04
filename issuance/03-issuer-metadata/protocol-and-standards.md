# Issuer Metadata -- Protocol & Standards

This document specifies the OID4VCI Issuer Metadata endpoint, response structure, credential configuration format, and display properties in detail. All references are to the **OpenID for Verifiable Credential Issuance (OID4VCI)** specification (Draft 13 and v1.0 Final).

---

## Metadata Endpoint URL Construction

The issuer metadata endpoint URL is deterministically constructed from the `credential_issuer` identifier.

### Rule

```
metadata_url = {credential_issuer} + "/.well-known/openid-credential-issuer"
```

### Examples

| `credential_issuer` | Metadata URL |
|---------------------|-------------|
| `https://issuer.example.com` | `https://issuer.example.com/.well-known/openid-credential-issuer` |
| `https://example.com/issuers/v1` | `https://example.com/.well-known/openid-credential-issuer/issuers/v1` |

Note the second example: when the `credential_issuer` includes a path component, the `/.well-known` segment is inserted after the host, and the path is appended after it. This follows the path-aware well-known URI construction defined in RFC 8615.

**OID4VCI Reference:** Section 11.2, "Credential Issuer Metadata" (v1.0 Final).

### Request

```http
GET /.well-known/openid-credential-issuer HTTP/1.1
Host: issuer.example.com
Accept: application/json
```

The wallet performs an unauthenticated HTTP GET request. No access token or client credentials are required.

### Response

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "credential_issuer": "https://issuer.example.com",
  "authorization_servers": ["https://auth.example.com"],
  "credential_endpoint": "https://issuer.example.com/credential",
  "deferred_credential_endpoint": "https://issuer.example.com/credential_deferred",
  "notification_endpoint": "https://issuer.example.com/notification",
  "credential_configurations_supported": { ... },
  "display": [ ... ]
}
```

---

## Response Structure

The top-level metadata response contains the following fields.

**OID4VCI Reference:** Section 11.2.1, "Credential Issuer Metadata Parameters" (v1.0 Final).

### Top-Level Fields

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `credential_issuer` | Yes | String | The credential issuer identifier. Must exactly match the `credential_issuer` value in Credential Offers from this issuer. |
| `authorization_servers` | No | Array of Strings | Identifiers of OAuth 2.0 authorization servers the issuer uses. If omitted, the issuer's own identifier is used as the authorization server. |
| `credential_endpoint` | Yes | String | URL of the credential endpoint where the wallet sends credential requests. |
| `deferred_credential_endpoint` | No | String | URL of the deferred credential endpoint for polling when immediate issuance is not possible. |
| `notification_endpoint` | No | String | URL where the wallet sends notification of credential acceptance, rejection, or deletion. |
| `credential_configurations_supported` | Yes | Object | A map from credential configuration identifiers to their definitions. This is the core of the metadata. |
| `display` | No | Array | Issuer-level display properties (name, logo) applicable to the issuer as a whole rather than individual credentials. |

### Issuer-Level Display

```json
{
  "display": [
    {
      "name": "Example Government ID Authority",
      "locale": "en-US",
      "logo": {
        "uri": "https://issuer.example.com/logo.png",
        "alt_text": "Government ID Authority Logo"
      }
    }
  ]
}
```

This display metadata describes the issuer itself and is shown in the wallet when presenting the offer source (e.g., "Credential offered by: Example Government ID Authority").

---

## Credential Configuration Object

Each entry in `credential_configurations_supported` defines a single credential type the issuer can issue.

**OID4VCI Reference:** Section 11.2.3, "Credential Configuration Object" (v1.0 Final).

### Common Fields (All Formats)

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `format` | Yes | String | The credential format identifier. Determines format-specific fields. |
| `scope` | No | String | OAuth 2.0 scope value associated with this credential. Used in authorization requests for wallet-initiated flows. |
| `cryptographic_binding_methods_supported` | No | Array | Methods for binding the credential to the holder's key. |
| `credential_signing_alg_values_supported` | No | Array | Algorithms the issuer uses to sign the credential. |
| `proof_types_supported` | No | Object | Proof types the issuer accepts in credential requests. |
| `display` | No | Array | Display properties for this credential type. |

### Format: mso_mdoc

For ISO 18013-5 mobile documents:

```json
{
  "format": "mso_mdoc",
  "doctype": "org.iso.18013.5.1.mDL",
  "scope": "org.iso.18013.5.1.mDL",
  "cryptographic_binding_methods_supported": ["cose_key"],
  "credential_signing_alg_values_supported": ["ES256", "ES384", "ES512"],
  "proof_types_supported": {
    "jwt": {
      "proof_signing_alg_values_supported": ["ES256"]
    }
  },
  "claims": {
    "org.iso.18013.5.1": {
      "given_name": {},
      "family_name": {},
      "birth_date": {},
      "document_number": {},
      "driving_privileges": {}
    }
  }
}
```

| Field | Description |
|-------|-------------|
| `doctype` | The ISO document type identifier (e.g., `org.iso.18013.5.1.mDL`). |
| `claims` | Claims organized by namespace. Each namespace contains claim names mapped to claim metadata (display name, required status). |

**OID4VCI Reference:** Section 11.2.3.1, "mso_mdoc Format" (v1.0 Final).

### Format: jwt_vc_json

For W3C Verifiable Credentials encoded as JWTs:

```json
{
  "format": "jwt_vc_json",
  "credential_definition": {
    "type": ["VerifiableCredential", "UniversityDegreeCredential"]
  },
  "scope": "UniversityDegree",
  "cryptographic_binding_methods_supported": ["did:key", "did:jwk"],
  "proof_types_supported": {
    "jwt": {
      "proof_signing_alg_values_supported": ["ES256", "EdDSA"]
    }
  }
}
```

| Field | Description |
|-------|-------------|
| `credential_definition.type` | The W3C VC `type` array. Must include `VerifiableCredential` plus one or more specific types. |

**OID4VCI Reference:** Section 11.2.3.2, "jwt_vc_json Format" (v1.0 Final).

### Format: vc+sd-jwt

For SD-JWT Verifiable Credentials:

```json
{
  "format": "vc+sd-jwt",
  "vct": "eu.europa.ec.eudi.pid.1",
  "scope": "eu.europa.ec.eudi.pid.1",
  "cryptographic_binding_methods_supported": ["jwk"],
  "proof_types_supported": {
    "jwt": {
      "proof_signing_alg_values_supported": ["ES256"]
    }
  },
  "claims": {
    "given_name": {
      "display": [
        { "name": "Given Name", "locale": "en-US" }
      ]
    },
    "family_name": {
      "display": [
        { "name": "Family Name", "locale": "en-US" }
      ]
    }
  }
}
```

| Field | Description |
|-------|-------------|
| `vct` | The Verifiable Credential Type identifier for SD-JWT VCs. |
| `claims` | Claims defined at the top level (not namespaced like mso_mdoc). |

**OID4VCI Reference:** Section 11.2.3.3, "vc+sd-jwt Format" (v1.0 Final).

---

## Cryptographic Binding Methods

The `cryptographic_binding_methods_supported` array specifies how the issued credential is bound to the holder's key material.

| Method | Description | Typical Format |
|--------|-------------|----------------|
| `cose_key` | Key is embedded as a COSE_Key structure. | mso_mdoc |
| `jwk` | Key is embedded as a JSON Web Key. | vc+sd-jwt, jwt_vc_json |
| `did:key` | Key is referenced via a did:key DID. | jwt_vc_json |
| `did:jwk` | Key is referenced via a did:jwk DID. | jwt_vc_json |
| `did:web` | Key is referenced via a did:web DID. | jwt_vc_json |

The binding method determines:
- What key format the wallet must provide in the proof of possession.
- How the issuer embeds the holder's public key in the issued credential.
- How verifiers later extract and verify the holder's key binding.

---

## Proof Types Supported

The `proof_types_supported` field tells the wallet what proof format to include in the credential request.

**OID4VCI Reference:** Section 7.2, "Proof Types" (v1.0 Final).

### JWT Proof Type

```json
"proof_types_supported": {
  "jwt": {
    "proof_signing_alg_values_supported": ["ES256", "ES384"]
  }
}
```

When the issuer supports `jwt` proofs, the wallet constructs a JWT containing:
- `iss`: The client ID (if applicable)
- `aud`: The credential issuer identifier
- `iat`: Issued-at timestamp
- `nonce`: The `c_nonce` value from the token response
- In the header: `typ` set to `openid4vci-proof+jwt`, `alg` set to the signing algorithm, and the public key as `jwk` or `kid`

### CWT Proof Type

```json
"proof_types_supported": {
  "cwt": {
    "proof_signing_alg_values_supported": ["ES256"]
    "proof_alg_values_supported": [-7]
  }
}
```

CWT proofs use COSE signing and CBOR encoding, aligning with the mso_mdoc format.

---

## Display Properties Per Locale

Each credential configuration and the issuer itself can include localized display metadata.

**OID4VCI Reference:** Section 11.2.3, "display" sub-field (v1.0 Final).

### Full Display Object

```json
{
  "display": [
    {
      "name": "Mobile Driving License",
      "locale": "en-US",
      "logo": {
        "uri": "https://issuer.example.com/logos/mdl-en.png",
        "alt_text": "Mobile Driving License"
      },
      "description": "A digital version of your driving license",
      "background_color": "#12107c",
      "background_image": {
        "uri": "https://issuer.example.com/images/mdl-bg.png"
      },
      "text_color": "#ffffff"
    },
    {
      "name": "Mobiler Fuhrerschein",
      "locale": "de-DE",
      "logo": {
        "uri": "https://issuer.example.com/logos/mdl-de.png",
        "alt_text": "Mobiler Fuhrerschein"
      },
      "description": "Eine digitale Version Ihres Fuhrerscheins",
      "background_color": "#12107c",
      "text_color": "#ffffff"
    }
  ]
}
```

### Display Fields

| Field | Type | Description |
|-------|------|-------------|
| `name` | String | Human-readable credential name. |
| `locale` | String | BCP 47 language tag (e.g., `en-US`, `de-DE`). |
| `logo` | Object | Contains `uri` (URL to logo image) and `alt_text`. |
| `description` | String | Longer description of the credential. |
| `background_color` | String | CSS hex color for card background (e.g., `#12107c`). |
| `background_image` | Object | Contains `uri` for a background image. |
| `text_color` | String | CSS hex color for text on the card. |

### Locale Resolution

The wallet should:
1. Attempt to match the user's device locale exactly (e.g., `en-US`).
2. Fall back to language-only matching (e.g., `en`).
3. Fall back to the first display entry, or one without a `locale` field.

---

## Authorization Server Metadata

When the issuer metadata includes `authorization_servers`, the wallet must also fetch the authorization server's metadata.

### Endpoint

```
{authorization_server}/.well-known/oauth-authorization-server
```

**Reference:** RFC 8414, "OAuth 2.0 Authorization Server Metadata".

### Key Fields

| Field | Description |
|-------|-------------|
| `issuer` | The authorization server identifier. |
| `authorization_endpoint` | URL for the authorization endpoint. |
| `token_endpoint` | URL for the token endpoint. |
| `pushed_authorization_request_endpoint` | URL for PAR (RFC 9126). |
| `grant_types_supported` | Supported grant types. |
| `code_challenge_methods_supported` | PKCE methods (e.g., `S256`). |
| `dpop_signing_alg_values_supported` | DPoP algorithms (RFC 9449). |
| `response_types_supported` | OAuth response types (typically `code`). |

### Relationship Between Issuer and Authorization Server Metadata

```
Credential Offer
  -> credential_issuer
    -> /.well-known/openid-credential-issuer
      -> authorization_servers: ["https://auth.example.com"]
        -> /.well-known/oauth-authorization-server
          -> authorization_endpoint, token_endpoint, etc.
      -> credential_endpoint
      -> credential_configurations_supported
```

The wallet uses the credential issuer metadata for credential-specific information (formats, proofs, display) and the authorization server metadata for OAuth-specific information (endpoints, supported mechanisms).

---

## Specification References

| Topic | Reference | Notes |
|-------|-----------|-------|
| Issuer Metadata endpoint | OID4VCI Section 11.2 | Endpoint location and discovery |
| Metadata response structure | OID4VCI Section 11.2.1 | Top-level fields |
| Credential Configuration Object | OID4VCI Section 11.2.3 | Per-credential-type metadata |
| mso_mdoc format | OID4VCI Section 11.2.3.1 | ISO 18013-5 specific fields |
| jwt_vc_json format | OID4VCI Section 11.2.3.2 | W3C VC JWT specific fields |
| vc+sd-jwt format | OID4VCI Section 11.2.3.3 | SD-JWT VC specific fields |
| Proof types | OID4VCI Section 7.2 | JWT and CWT proof definitions |
| Display properties | OID4VCI Section 11.2.3 (display) | Localized display metadata |
| Well-known URIs | RFC 8615 | URI construction rules |
| Authorization Server Metadata | RFC 8414 | OAuth 2.0 server metadata |
