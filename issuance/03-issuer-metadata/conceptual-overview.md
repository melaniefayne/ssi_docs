# Issuer Metadata -- Conceptual Overview

**Issuer metadata** is a machine-readable document published by a credential issuer that describes its capabilities: what credentials it can issue, in what formats, with what cryptographic requirements, and how to display them to the user. It is the wallet's primary source of truth for understanding how to interact with a specific issuer.

---

## The Well-Known Endpoint

Every OID4VCI-compliant issuer publishes its metadata at a standardized URL derived from the issuer's identifier:

```
{credential_issuer}/.well-known/openid-credential-issuer
```

For example, if the `credential_issuer` value from a Credential Offer is `https://issuer.example.com`, the wallet fetches metadata from:

```
https://issuer.example.com/.well-known/openid-credential-issuer
```

This follows the well-known URI pattern established by RFC 8615, which provides a standardized location for host-level metadata. The wallet performs a simple HTTP GET request to this URL and receives a JSON document containing the issuer's full configuration.

The well-known endpoint serves the same architectural role as OpenID Connect's `/.well-known/openid-configuration` endpoint -- it enables dynamic discovery of an issuer's capabilities without requiring out-of-band configuration exchange.

---

## Credential Configurations Supported

The most important field in the issuer metadata is `credential_configurations_supported`. This is a map of credential configuration identifiers to their definitions, describing every type of credential the issuer can issue.

```json
{
  "credential_issuer": "https://issuer.example.com",
  "credential_configurations_supported": {
    "org.iso.18013.5.1.mDL": {
      "format": "mso_mdoc",
      "doctype": "org.iso.18013.5.1.mDL",
      "scope": "org.iso.18013.5.1.mDL",
      "cryptographic_binding_methods_supported": ["cose_key"],
      "credential_signing_alg_values_supported": ["ES256"],
      "proof_types_supported": {
        "jwt": {
          "proof_signing_alg_values_supported": ["ES256"]
        }
      },
      "display": [
        {
          "name": "Mobile Driving License",
          "locale": "en-US",
          "logo": {
            "uri": "https://issuer.example.com/logos/mdl.png",
            "alt_text": "Mobile Driving License logo"
          },
          "background_color": "#12107c",
          "text_color": "#ffffff"
        }
      ]
    },
    "eu.europa.ec.eudi.pid.1": {
      "format": "vc+sd-jwt",
      "vct": "eu.europa.ec.eudi.pid.1",
      "scope": "eu.europa.ec.eudi.pid.1",
      "cryptographic_binding_methods_supported": ["jwk"],
      "proof_types_supported": {
        "jwt": {
          "proof_signing_alg_values_supported": ["ES256"]
        }
      },
      "display": [
        {
          "name": "Person Identification Data",
          "locale": "en-US"
        }
      ]
    }
  }
}
```

Each credential configuration contains several categories of information.

### Credential Format

The `format` field specifies the data format of the issued credential. Common formats include:

| Format | Standard | Description |
|--------|----------|-------------|
| `mso_mdoc` | ISO 18013-5 | Mobile document format using CBOR encoding and Mobile Security Object (MSO) for integrity. Used for mobile driving licenses and other ISO-standardized documents. |
| `jwt_vc_json` | W3C VC Data Model | Verifiable Credential encoded as a JSON Web Token. Uses JSON-LD or plain JSON for the credential payload. |
| `vc+sd-jwt` | SD-JWT VC | Verifiable Credential using Selective Disclosure JWT. Allows the holder to disclose only specific claims during presentation. |

The format determines:
- How the wallet constructs the credential request
- What proof types are applicable
- How the issued credential is stored and later presented

### Scope

The `scope` field links the credential configuration to an OAuth 2.0 scope value. When using the Authorization Code Grant, the wallet includes this scope in the authorization request to signal which credential type it intends to request.

### Cryptographic Requirements

Two fields describe the cryptographic capabilities required for issuance:

- **`cryptographic_binding_methods_supported`** -- How the credential is bound to the holder's key. Values include `cose_key` (for mso_mdoc), `jwk` (for JWT-based formats), and `did:key` or other DID methods.
- **`proof_types_supported`** -- What proof types the issuer accepts in the credential request. For each proof type (e.g., `jwt`), the supported signing algorithms are listed.

Together, these fields tell the wallet which key type to generate, how to format the proof of possession, and which signing algorithm to use.

---

## Display Metadata

Each credential configuration includes a `display` array that provides human-readable information for rendering the credential in the wallet's UI.

### Display Properties

| Property | Description |
|----------|-------------|
| `name` | The human-readable name of the credential type (e.g., "Mobile Driving License"). |
| `locale` | The language/region for this display entry (e.g., "en-US", "de-DE"). Multiple display entries can provide localized names and descriptions. |
| `logo` | An object containing `uri` (URL to the logo image) and `alt_text`. |
| `description` | A longer description of the credential. |
| `background_color` | A CSS-style hex color for rendering the credential card background. |
| `text_color` | A CSS-style hex color for text on the credential card. |

### Localization

The `display` array supports multiple entries with different `locale` values, allowing the wallet to select the most appropriate display metadata for the user's language preference:

```json
"display": [
  {
    "name": "Personalausweis",
    "locale": "de-DE",
    "description": "Digitaler Personalausweis der Bundesrepublik Deutschland"
  },
  {
    "name": "National ID Card",
    "locale": "en-US",
    "description": "Digital national identity card of Germany"
  }
]
```

The wallet matches the user's device locale against available display entries and falls back to the first entry (or an entry without a locale) if no match is found.

---

## Proof Types

The `proof_types_supported` field within each credential configuration specifies what kinds of proof the issuer expects in the credential request. A proof demonstrates that the wallet controls the private key corresponding to the public key that will be bound to the credential.

### JWT Proof

The most common proof type across the implementations studied:

```json
"proof_types_supported": {
  "jwt": {
    "proof_signing_alg_values_supported": ["ES256", "ES384"]
  }
}
```

The wallet constructs a JWT proof containing:
- The nonce (`c_nonce`) received from the issuer
- The wallet's public key (in the JWT header as `jwk` or `kid`)
- A signature using one of the supported algorithms

### CWT Proof

Used with mso_mdoc format credentials:

```json
"proof_types_supported": {
  "cwt": {
    "proof_signing_alg_values_supported": ["ES256"]
  }
}
```

CWT (CBOR Web Token) proofs use COSE signing instead of JWS, aligning with the CBOR-based mso_mdoc format.

The wallet reads the `proof_types_supported` field to determine how to construct its proof of possession for the credential request. See [07 -- Proof Construction](../07-proof-construction/) for full details.

---

## Authorization Server Metadata

The issuer metadata may reference one or more authorization servers:

```json
{
  "credential_issuer": "https://issuer.example.com",
  "authorization_servers": [
    "https://auth.example.com"
  ],
  "credential_configurations_supported": { ... }
}
```

### Why Separate Authorization Servers?

The OID4VCI specification allows the credential issuer and the authorization server to be separate entities. This supports deployment models where:

- A government identity provider handles authentication (the authorization server) while a separate department issues credentials (the credential issuer).
- A centralized OAuth 2.0 provider is shared across multiple issuers.
- The authorization server implements specialized authentication mechanisms (e.g., eID card readers, biometric verification) that the issuer does not manage directly.

### Resolving Authorization Server Metadata

When `authorization_servers` is present, the wallet must also fetch the authorization server's own metadata from:

```
{authorization_server}/.well-known/oauth-authorization-server
```

This metadata provides:
- The `authorization_endpoint` URL
- The `token_endpoint` URL
- Supported `grant_types`
- PKCE support (`code_challenge_methods_supported`)
- PAR support (`pushed_authorization_request_endpoint`)
- DPoP support

If `authorization_servers` is omitted, the wallet assumes the credential issuer itself acts as the authorization server and constructs the authorization server metadata URL from the `credential_issuer` identifier.

---

## Metadata in the Issuance Flow

Issuer metadata is consumed at multiple points during the issuance flow:

1. **Offer resolution** (Section 02) -- The wallet validates that the `credential_configuration_ids` in the offer match entries in the issuer's metadata.
2. **Authorization** (Section 04) -- The wallet uses `authorization_servers` and scope information to construct the authorization request.
3. **Key generation** (Section 06) -- The wallet reads `cryptographic_binding_methods_supported` to determine what key type to generate.
4. **Proof construction** (Section 07) -- The wallet reads `proof_types_supported` to determine the proof format and signing algorithm.
5. **Credential request** (Section 08) -- The wallet uses the `format` and configuration ID to construct the credential request payload.
6. **UI rendering** -- Display metadata (name, logo, colors) is used throughout the issuance flow to present credential information to the user.

This makes issuer metadata a foundational data structure that influences nearly every subsequent step in the protocol.
