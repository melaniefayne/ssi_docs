# Authorization Request — Protocol and Standards

This document details the OID4VP authorization request structure as defined in the specification, including all parameters, encoding requirements, and validation rules.

---

## OID4VP Authorization Request Parameters

### Required Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `response_type` | string | Must be `vp_token` or `vp_token id_token` |
| `client_id` | string | Verifier identifier |
| `nonce` | string | Unique challenge for replay protection |

### Credential Query Parameters (one required)

| Parameter | Type | Description |
|-----------|------|-------------|
| `presentation_definition` | object | DIF Presentation Exchange query |
| `presentation_definition_uri` | string | URI to fetch presentation definition |
| `dcql_query` | object | Digital Credentials Query Language query |

### Response Delivery Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `response_mode` | string | `fragment` | How to deliver response |
| `response_uri` | string | - | Destination for `direct_post` responses |
| `redirect_uri` | string | - | Destination for `fragment` responses |

### Client Identification Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `client_id_scheme` | string | How client_id should be interpreted |
| `client_metadata` | object | Inline client metadata |
| `client_metadata_uri` | string | URI to fetch client metadata |

### Security Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `state` | string | Client state, echoed in response |
| `request` | string | Signed request object (JAR) |
| `request_uri` | string | URI to fetch request object |

---

## Response Types

### `vp_token`

Standard response containing only the Verifiable Presentation:

```json
{
  "response_type": "vp_token"
}
```

### `vp_token id_token`

Combined with Self-Issued OpenID Provider v2:

```json
{
  "response_type": "vp_token id_token"
}
```

Returns both a VP token and a self-issued ID token containing holder claims.

---

## Response Modes

### `fragment`

Response in URL fragment (default for same-device):

```
https://verifier.example.com/callback#
  vp_token=eyJ...&
  presentation_submission={...}&
  state=af0ifjsldkj
```

### `direct_post`

HTTP POST to response_uri (recommended for cross-device):

```http
POST /callback HTTP/1.1
Host: verifier.example.com
Content-Type: application/x-www-form-urlencoded

vp_token=eyJ...&
presentation_submission=%7B...%7D&
state=af0ifjsldkj
```

### `direct_post.jwt`

Encrypted and/or signed POST response:

```http
POST /callback HTTP/1.1
Host: verifier.example.com
Content-Type: application/x-www-form-urlencoded

response=eyJ...
```

Where `response` is a JWT containing `vp_token` and `presentation_submission`.

---

## Request URI Fetching

When `request_uri` is present:

```http
GET /requests/abc123 HTTP/1.1
Host: verifier.example.com
Accept: application/oauth-authz-req+jwt, application/jwt

HTTP/1.1 200 OK
Content-Type: application/oauth-authz-req+jwt

eyJhbGciOiJFUzI1NiIsInR5cCI6Im9hdXRoLWF1dGh6LXJlcStqd3QifQ...
```

### Requirements

| Aspect | Requirement |
|--------|-------------|
| Protocol | HTTPS required |
| Content-Type | `application/oauth-authz-req+jwt` for signed, `application/json` for unsigned |
| Caching | Should be single-use or short-lived |
| Timing | Wallet should fetch promptly |

---

## Signed Request Object (JAR)

### JWT Header

```json
{
  "alg": "ES256",
  "typ": "oauth-authz-req+jwt",
  "kid": "verifier-key-1",
  "x5c": ["MIIB...", "MIIC..."]
}
```

| Field | Description |
|-------|-------------|
| `alg` | Signing algorithm |
| `typ` | Must be `oauth-authz-req+jwt` |
| `kid` | Key identifier for DID-based verification |
| `x5c` | Certificate chain for X.509-based verification |

### JWT Payload

```json
{
  "iss": "https://verifier.example.com",
  "aud": "https://self-issued.me/v2",
  "iat": 1683000000,
  "exp": 1683000300,
  "client_id": "https://verifier.example.com",
  "client_id_scheme": "x509_san_dns",
  "response_type": "vp_token",
  "response_mode": "direct_post",
  "response_uri": "https://verifier.example.com/callback",
  "nonce": "n-0S6_WzA2Mj",
  "state": "af0ifjsldkj",
  "presentation_definition": {
    "id": "example_presentation",
    "input_descriptors": [...]
  }
}
```

| Field | Description |
|-------|-------------|
| `iss` | Request issuer (should match client_id) |
| `aud` | Intended audience |
| `iat` | Issued at timestamp |
| `exp` | Expiration timestamp |

### Validation Rules

1. `iss` must match `client_id` in payload
2. `iat` must be in the past
3. `exp` must be in the future
4. Signature must validate against verifier's key
5. For `x509_san_dns`: certificate SAN must match `client_id`
6. For `did`: signature must verify against DID Document key

---

## Client ID Scheme Details

### `redirect_uri`

```json
{
  "client_id_scheme": "redirect_uri",
  "client_id": "https://verifier.example.com/callback"
}
```

- `client_id` must exactly match `response_uri` or `redirect_uri`
- No cryptographic verification of client identity
- Trust based solely on HTTPS

### `x509_san_dns`

```json
{
  "client_id_scheme": "x509_san_dns",
  "client_id": "verifier.example.com"
}
```

- Request must be signed JWT
- JWT header must contain `x5c` certificate chain
- Leaf certificate must have DNS SAN matching `client_id`
- Certificate chain must validate to trusted root

### `x509_san_uri`

```json
{
  "client_id_scheme": "x509_san_uri",
  "client_id": "https://verifier.example.com"
}
```

- Same as `x509_san_dns` but uses URI SAN
- Certificate must have URI SAN matching `client_id`

### `did`

```json
{
  "client_id_scheme": "did",
  "client_id": "did:web:verifier.example.com"
}
```

- Request must be signed JWT
- Wallet resolves DID to DID Document
- JWT `kid` references verification method in DID Document
- Signature validates against that verification method

### `verifier_attestation`

```json
{
  "client_id_scheme": "verifier_attestation",
  "client_id": "verifier-123"
}
```

- JWT header contains `jwt` with verifier attestation
- Attestation issued by trusted party
- Attestation `sub` matches `client_id`
- Attestation `cnf` contains verifier's public key

---

## Nonce Requirements

The `nonce` parameter:

| Requirement | Specification |
|-------------|---------------|
| Uniqueness | Must be unique per request |
| Randomness | Cryptographically random |
| Length | Sufficient entropy (recommend 256 bits) |
| Usage | Single-use, must be consumed after verification |

The nonce appears in:
1. Request: `nonce` parameter
2. VP Token: In the proof (e.g., JWT `nonce` claim or mDoc session transcript)

---

## Presentation Definition URI

When using `presentation_definition_uri`:

```json
{
  "presentation_definition_uri": "https://verifier.example.com/definitions/kyc"
}
```

Wallet fetches:

```http
GET /definitions/kyc HTTP/1.1
Host: verifier.example.com
Accept: application/json

HTTP/1.1 200 OK
Content-Type: application/json

{
  "id": "kyc_verification",
  "input_descriptors": [...]
}
```

### Caching Considerations

- Definition may be cached if static
- `Cache-Control` headers should be respected
- Consider freshness for dynamic requirements

---

## DCQL Query (Alternative to Presentation Definition)

Digital Credentials Query Language provides an alternative query format:

```json
{
  "dcql_query": {
    "credentials": [
      {
        "id": "pid",
        "format": "vc+sd-jwt",
        "claims": [
          {"path": ["given_name"]},
          {"path": ["family_name"]},
          {"path": ["birthdate"]}
        ]
      }
    ]
  }
}
```

DCQL is designed to be simpler than Presentation Exchange for common use cases.

---

## Error Responses

Wallet returns errors via the specified response mode:

```
https://verifier.example.com/callback#
  error=invalid_request&
  error_description=Missing+nonce+parameter&
  state=af0ifjsldkj
```

| Error Code | Description |
|------------|-------------|
| `invalid_request` | Malformed request |
| `unauthorized_client` | Client not authorized |
| `access_denied` | User denied consent |
| `unsupported_response_type` | Response type not supported |
| `invalid_scope` | Invalid scope parameter |
| `vp_formats_not_supported` | Requested format not supported |
| `invalid_presentation_definition_uri` | Cannot fetch definition |
| `invalid_presentation_definition_reference` | Definition not found |

---

## Standards Reference

| Standard | Section | Topic |
|----------|---------|-------|
| OID4VP | 5 | Authorization Request |
| OID4VP | 6 | Request Object |
| OID4VP | 7 | Response Modes |
| OID4VP | 9 | Client Identifier Schemes |
| RFC 9101 | - | JWT-Secured Authorization Request (JAR) |
| RFC 6749 | 4.1.1 | OAuth 2.0 Authorization Request |
