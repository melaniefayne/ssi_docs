# Credential Offer -- Protocol & Standards

This document specifies the OID4VCI Credential Offer protocol in detail, covering URI formats, grant types, transaction code parameters, and the offer resolution process. All references are to the **OpenID for Verifiable Credential Issuance (OID4VCI)** specification (Draft 13 and v1.0 Final).

---

## Credential Offer URI Format

The Credential Offer is delivered to the wallet as a URI using the `openid-credential-offer` custom scheme. The URI contains the offer data in one of two forms.

### Inline Offer

The complete Credential Offer JSON object is URL-encoded and included directly as the `credential_offer` query parameter:

```
openid-credential-offer://?credential_offer=%7B%22credential_issuer%22%3A%22https%3A%2F%2Fissuer.example.com%22%2C%22credential_configuration_ids%22%3A%5B%22org.iso.18013.5.1.mDL%22%5D%2C%22grants%22%3A%7B%22urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Apre-authorized_code%22%3A%7B%22pre-authorized_code%22%3A%22SplxlOBeZQQYbYS6WxSbIA%22%7D%7D%7D
```

Decoded, this is:

```
openid-credential-offer://?credential_offer={"credential_issuer":"https://issuer.example.com","credential_configuration_ids":["org.iso.18013.5.1.mDL"],"grants":{"urn:ietf:params:oauth:grant-type:pre-authorized_code":{"pre-authorized_code":"SplxlOBeZQQYbYS6WxSbIA"}}}
```

**OID4VCI Reference:** Section 4.1, "Credential Offer" (v1.0 Final).

### By-Reference Offer

A URL pointing to the Credential Offer JSON object is included as the `credential_offer_uri` query parameter:

```
openid-credential-offer://?credential_offer_uri=https%3A%2F%2Fissuer.example.com%2Foffers%2Fabc123
```

The wallet resolves this URI with an HTTP GET request to retrieve the full Credential Offer object. The response must be `application/json` with the same structure as the inline offer.

**OID4VCI Reference:** Section 4.1.1, "Credential Offer URI" (v1.0 Final). The specification notes that `credential_offer_uri` is preferred when the offer object is large or when the issuer wants to enforce single-use semantics.

### HAIP URI Scheme

Some implementations also support the `haip-vci://` URI scheme, defined by the High Assurance Interoperability Profile (HAIP). This scheme signals that the offer conforms to HAIP requirements (specific cryptographic algorithms, key attestation, etc.) and is functionally equivalent to `openid-credential-offer://` for offer delivery purposes.

---

## Grant Types

The `grants` object within the Credential Offer specifies which OAuth 2.0 grant types the wallet may use to obtain an access token. OID4VCI defines two grant types for credential issuance.

### Pre-Authorized Code Grant

**Grant type identifier:** `urn:ietf:params:oauth:grant-type:pre-authorized_code`

This grant type indicates that the issuer has already authorized the issuance and is providing a code that the wallet can exchange directly at the token endpoint, without visiting the authorization endpoint.

**OID4VCI Reference:** Section 4.1.1, "Pre-Authorized Code Grant" (v1.0 Final).

#### Offer Structure

```json
{
  "credential_issuer": "https://issuer.example.com",
  "credential_configuration_ids": ["eu.europa.ec.eudi.pid.1"],
  "grants": {
    "urn:ietf:params:oauth:grant-type:pre-authorized_code": {
      "pre-authorized_code": "SplxlOBeZQQYbYS6WxSbIA",
      "tx_code": {
        "input_mode": "numeric",
        "length": 6,
        "description": "Enter the 6-digit code from your email"
      }
    }
  }
}
```

#### Fields

| Field | Required | Description |
|-------|----------|-------------|
| `pre-authorized_code` | Yes | A short-lived, single-use code that the wallet sends to the token endpoint. |
| `tx_code` | No | If present, the wallet must collect a transaction code from the user before exchanging the pre-authorized code. See [TxCode Specification](#txcode-specification) below. |

#### Token Exchange

The wallet sends the pre-authorized code to the token endpoint:

```http
POST /token HTTP/1.1
Host: issuer.example.com
Content-Type: application/x-www-form-urlencoded

grant_type=urn:ietf:params:oauth:grant-type:pre-authorized_code
&pre-authorized_code=SplxlOBeZQQYbYS6WxSbIA
&tx_code=123456
```

The `tx_code` parameter is included only if the offer specified a `tx_code` requirement.

### Authorization Code Grant

**Grant type identifier:** `authorization_code`

This grant type indicates that the wallet must follow the standard OAuth 2.0 Authorization Code flow, redirecting the user to the issuer's authorization endpoint for authentication and consent.

**OID4VCI Reference:** Section 4.1.2, "Authorization Code Grant" (v1.0 Final).

#### Offer Structure

```json
{
  "credential_issuer": "https://issuer.example.com",
  "credential_configuration_ids": ["org.iso.18013.5.1.mDL"],
  "grants": {
    "authorization_code": {
      "issuer_state": "eyJhbGciOiJSU0EtT..."
    }
  }
}
```

#### Fields

| Field | Required | Description |
|-------|----------|-------------|
| `issuer_state` | No | An opaque string that the wallet includes in the authorization request. Binds the authorization session to this specific offer. |
| `authorization_server` | No | The identifier of the authorization server to use, if the issuer supports multiple. Must match one of the entries in `authorization_servers` from the issuer's metadata. |

#### Authorization Flow

1. The wallet constructs an authorization request using PKCE (RFC 7636) and optionally PAR (RFC 9126).
2. The user authenticates at the authorization endpoint.
3. The authorization server redirects back to the wallet with an authorization code.
4. The wallet exchanges the authorization code for an access token at the token endpoint.

### Dual Grant Offers

An offer may include both grant types:

```json
{
  "grants": {
    "urn:ietf:params:oauth:grant-type:pre-authorized_code": {
      "pre-authorized_code": "SplxlOBeZQQYbYS6WxSbIA"
    },
    "authorization_code": {
      "issuer_state": "eyJhbGciOiJSU0EtT..."
    }
  }
}
```

The wallet chooses which grant type to use. The specification does not mandate a preference, but implementations typically prefer the Pre-Authorized Code Grant when no user authentication is needed at the authorization server.

---

## TxCode Specification

The `tx_code` object within the Pre-Authorized Code Grant provides parameters for a transaction code that the user must enter to complete the token exchange. This serves as an out-of-band second factor, binding a specific person to the offer.

**OID4VCI Reference:** Section 4.1.1.1, "Transaction Code" (v1.0 Final).

### Structure

```json
{
  "tx_code": {
    "input_mode": "numeric",
    "length": 6,
    "description": "Enter the 6-digit code sent to your registered email address"
  }
}
```

### Fields

| Field | Required | Default | Description |
|-------|----------|---------|-------------|
| `input_mode` | No | `numeric` | The type of characters the transaction code consists of. Defined values: `numeric` (digits only), `text` (any characters). |
| `length` | No | - | The expected length of the transaction code. If omitted, the wallet should present a variable-length input field. |
| `description` | No | - | A human-readable description providing context about how the transaction code was delivered and what the user should enter. |

### Validation Rules

Per the specification:

- If `input_mode` is `numeric`, the wallet should constrain input to digits (0-9) and may present a numeric keyboard.
- If `input_mode` is `text`, the wallet should accept any character input.
- If `length` is specified, the wallet should enforce the exact length and may auto-submit when the required number of characters is entered.
- The `description` field should be displayed to the user but has no programmatic effect.

The transaction code is transmitted to the token endpoint as the `tx_code` parameter alongside the `pre-authorized_code`.

---

## Offer Resolution Process

When a wallet receives a Credential Offer URI, it follows a defined resolution process to extract the offer data and prepare for issuance.

### Step-by-Step Resolution

```
1. Receive URI
   openid-credential-offer://?credential_offer=...
   OR
   openid-credential-offer://?credential_offer_uri=...

2. Parse URI scheme and query parameters
   - Validate scheme is openid-credential-offer:// (or haip-vci://)
   - Extract credential_offer or credential_offer_uri parameter

3. Obtain Credential Offer object
   - If credential_offer: URL-decode and parse JSON
   - If credential_offer_uri: HTTP GET to resolve URI, parse JSON response

4. Validate Credential Offer object
   - credential_issuer: Must be a valid URL
   - credential_configuration_ids: Must be a non-empty array of strings
   - grants: If present, must contain at least one recognized grant type

5. Resolve issuer metadata
   - Construct metadata URL: {credential_issuer}/.well-known/openid-credential-issuer
   - HTTP GET to retrieve issuer metadata
   - Validate that all credential_configuration_ids in the offer exist
     in the issuer's credential_configurations_supported

6. Present offer to user
   - Display credential types being offered (using display metadata from issuer)
   - If tx_code is required, present input field
   - Await user acceptance or rejection

7. Begin token exchange
   - If Pre-Authorized Code Grant: proceed to token endpoint
   - If Authorization Code Grant: proceed to authorization endpoint
```

### Error Conditions

| Condition | Expected Behavior |
|-----------|-------------------|
| Unknown URI scheme | Wallet cannot handle the URI; OS may show an error or offer to search for a compatible app. |
| Invalid JSON in `credential_offer` | Wallet rejects the offer and displays an error. |
| `credential_offer_uri` returns non-200 | Wallet displays a resolution error. The offer may have expired or been consumed. |
| `credential_configuration_ids` not found in issuer metadata | Wallet displays that the offered credential type is not recognized. |
| `credential_issuer` metadata endpoint unreachable | Wallet displays a connectivity or issuer availability error. |

---

## Specification References

| Topic | OID4VCI Section | Notes |
|-------|-----------------|-------|
| Credential Offer object | Section 4.1 | Core structure definition |
| Credential Offer URI | Section 4.1.1 | URI format and parameters |
| Pre-Authorized Code Grant | Section 4.1.1 | Grant type definition, `pre-authorized_code` and `tx_code` |
| Transaction Code | Section 4.1.1.1 | `tx_code` object specification |
| Authorization Code Grant | Section 4.1.2 | `issuer_state`, `authorization_server` |
| Credential Offer Endpoint | Section 4.2 | Issuer-side endpoint for offer creation |
| Token Endpoint | Section 6 | Token exchange using pre-authorized code |

### Related Standards

| Standard | Relevance |
|----------|-----------|
| RFC 7636 (PKCE) | Required for Authorization Code Grant |
| RFC 9126 (PAR) | Optional pushed authorization for Authorization Code Grant |
| RFC 6749 (OAuth 2.0) | Foundation for both grant types |
| HAIP | High Assurance Interoperability Profile; defines `haip-vci://` URI scheme |
