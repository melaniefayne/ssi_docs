# Presentation Request — Protocol & Standards

This document specifies the OID4VP Authorization Request in detail, covering URI formats, parameters, client identification schemes, and request resolution. All references are to the **OpenID for Verifiable Presentations (OID4VP)** specification.

---

## Authorization Request URI Format

The presentation request is delivered as an Authorization Request URI using the `openid4vp` custom scheme:

```
openid4vp://authorize?
  response_type=vp_token&
  client_id=https://verifier.example.com&
  nonce=n-0S6_WzA2Mj&
  presentation_definition=...&
  response_mode=direct_post&
  response_uri=https://verifier.example.com/callback&
  state=af0ifjsldkj
```

Alternative schemes supported by implementations:
- `eudi-openid4vp://` — EUDI-specific scheme
- `mdoc-openid4vp://` — mDoc-specific scheme
- `haip-openid4vp://` — HAIP profile scheme

---

## Core Parameters

### Required Parameters

| Parameter | Description | OID4VP Reference |
|-----------|-------------|------------------|
| `response_type` | Must be `vp_token` (or `vp_token id_token` with SIOPv2) | Section 5 |
| `client_id` | Verifier identifier, format depends on client_id_scheme | Section 5.1 |
| `nonce` | Cryptographically random value to prevent replay | Section 5 |

### Credential Request Parameters

One of these must be present:

| Parameter | Description | OID4VP Reference |
|-----------|-------------|------------------|
| `presentation_definition` | Inline JSON object with credential requirements | Section 5.4 |
| `presentation_definition_uri` | URI to fetch presentation definition | Section 5.4 |
| `dcql_query` | Digital Credentials Query Language query | Section 5.5 |
| `scope` | Pre-defined query alias (if supported) | Section 5.3 |

### Response Parameters

| Parameter | Description | OID4VP Reference |
|-----------|-------------|------------------|
| `response_mode` | How response is delivered (`direct_post`, `fragment`, etc.) | Section 6 |
| `response_uri` | Where to send response (for `direct_post`) | Section 6.2 |
| `redirect_uri` | Redirect location (for `fragment` mode) | Section 6.1 |
| `state` | Opaque value echoed in response | Section 5 |

### Optional Parameters

| Parameter | Description | OID4VP Reference |
|-----------|-------------|------------------|
| `client_metadata` | Verifier metadata if not pre-registered | Section 5.1.2 |
| `client_id_scheme` | How to interpret client_id | Section 5.1.1 |
| `request_uri` | URI to fetch full request object | Section 5.6 |
| `request_uri_method` | HTTP method for request_uri (`get` or `post`) | Section 5.6 |

---

## Request Object (Signed Request)

For signed requests, the parameters are wrapped in a JWT:

### By Value

```
openid4vp://authorize?
  client_id=https://verifier.example.com&
  request=eyJhbGciOiJFUzI1NiIsInR5cCI6Im9hdXRoLWF1dGh6LXJlcStqd3QifQ...
```

### By Reference

```
openid4vp://authorize?
  client_id=https://verifier.example.com&
  request_uri=https://verifier.example.com/requests/abc123
```

### JWT Structure

```json
// Header
{
  "alg": "ES256",
  "typ": "oauth-authz-req+jwt",
  "kid": "verifier-key-1"
}

// Payload
{
  "iss": "https://verifier.example.com",
  "aud": "https://self-issued.me/v2",
  "response_type": "vp_token",
  "client_id": "https://verifier.example.com",
  "nonce": "n-0S6_WzA2Mj",
  "state": "af0ifjsldkj",
  "presentation_definition": { ... },
  "response_mode": "direct_post",
  "response_uri": "https://verifier.example.com/callback",
  "iat": 1683000000,
  "exp": 1683000300
}
```

**Validation requirements:**
- Signature must be valid
- `iat` must be in the past
- `exp` must be in the future
- `client_id` in JWT must match URI parameter

---

## Client ID Schemes

### redirect_uri

The client_id equals the redirect/response URI:

```
client_id=https://verifier.example.com/callback&
response_uri=https://verifier.example.com/callback
```

- No request signing required
- All metadata via `client_metadata` parameter
- Lowest security, simplest deployment

### x509_san_dns

Client identified by DNS name in X.509 certificate SAN:

```
client_id=verifier.example.com
```

**Validation:**
1. Request must be signed
2. Extract signing certificate from JWT
3. Validate certificate chain against trust store
4. Check SAN contains client_id DNS name

### x509_san_hash (x509_hash)

Client identified by SHA-256 hash of certificate:

```
client_id=x509_san_hash:abc123def456...
```

**Validation:**
1. Extract signing certificate
2. Compute SHA-256 of DER-encoded certificate
3. Compare with client_id hash

### decentralized_identifier (DID)

Client identified by DID:

```
client_id=did:web:verifier.example.com
```

**Validation:**
1. Resolve DID to DID Document
2. Find verification method matching JWT `kid`
3. Verify JWT signature with public key

### verifier_attestation

Client provides attestation JWT:

```
client_id=custom-verifier-id
```

Request includes attestation in `client_metadata`:

```json
{
  "client_metadata": {
    "verifier_attestation": "eyJ0eXAiOiJ2ZXJpZmllci1hdHRlc3RhdGlvbitqd3QiLCJhbGciOiJFUzI1NiJ9..."
  }
}
```

**Validation:**
1. Validate attestation JWT issuer
2. Check attestation `sub` matches client_id
3. Validate attestation not expired

---

## Response Modes

### direct_post

Response sent via HTTP POST to `response_uri`:

```http
POST /callback HTTP/1.1
Host: verifier.example.com
Content-Type: application/x-www-form-urlencoded

vp_token=eyJ...&
presentation_submission={...}&
state=af0ifjsldkj
```

**Characteristics:**
- Works for cross-device flows
- Supports large responses
- Verifier must expose endpoint
- Most common for production use

### direct_post.jwt

Response encrypted to verifier:

```http
POST /callback HTTP/1.1
Host: verifier.example.com
Content-Type: application/x-www-form-urlencoded

response=eyJ...
```

The `response` is a JWE containing the vp_token.

**Use when:**
- Response confidentiality required
- Multi-hop or proxy scenarios

### fragment

Response in redirect URI fragment:

```
https://verifier.example.com/callback#
  vp_token=eyJ...&
  presentation_submission={...}&
  state=af0ifjsldkj
```

**Characteristics:**
- Fragment not sent to server (browser security)
- Client-side JavaScript must extract
- Limited by URL length
- Same-device flows only

---

## Presentation Definition

Specifies credential requirements. Full coverage in [05 Presentation Definition](../05-presentation-definition/).

### Inline

```json
{
  "presentation_definition": {
    "id": "example-pd",
    "input_descriptors": [
      {
        "id": "id-credential",
        "format": {
          "jwt_vc_json": { "alg": ["ES256"] }
        },
        "constraints": {
          "fields": [
            {
              "path": ["$.credentialSubject.given_name"],
              "filter": { "type": "string" }
            }
          ]
        }
      }
    ]
  }
}
```

### By Reference

```
presentation_definition_uri=https://verifier.example.com/pd/identity-verification
```

Wallet fetches:
```http
GET /pd/identity-verification HTTP/1.1
Host: verifier.example.com

HTTP/1.1 200 OK
Content-Type: application/json

{
  "id": "example-pd",
  "input_descriptors": [ ... ]
}
```

---

## DCQL Query (Alternative to Presentation Definition)

Digital Credentials Query Language provides a simpler query format:

```json
{
  "dcql_query": {
    "credentials": [
      {
        "id": "pid",
        "format": "dc+sd-jwt",
        "meta": {
          "vct_values": ["https://example.com/pid"]
        },
        "claims": [
          { "path": ["given_name"] },
          { "path": ["family_name"] }
        ]
      }
    ]
  }
}
```

**Advantages over Presentation Definition:**
- Simpler structure
- Direct claim specification
- Format-specific metadata support

---

## Request Resolution Flow

```
1. Receive URI
   openid4vp://authorize?client_id=...&request_uri=...

2. Parse query parameters
   - Extract client_id, request_uri, etc.

3. Fetch request object (if request_uri present)
   GET {request_uri}
   → Receive signed JWT

4. Validate request object
   - Verify JWT signature
   - Check iat/exp
   - Match client_id

5. Extract parameters from JWT payload
   - presentation_definition
   - nonce, state
   - response_mode, response_uri

6. Resolve client identity
   - Based on client_id_scheme
   - Validate trust chain

7. Fetch presentation_definition (if by reference)
   GET {presentation_definition_uri}

8. Ready to process
   - Match credentials
   - Display to user
```

---

## Error Responses

When the wallet cannot process a request, it returns errors via the response channel:

| Error Code | Description |
|------------|-------------|
| `invalid_request` | Request is malformed |
| `invalid_scope` | Requested scope not supported |
| `vp_formats_not_supported` | Requested credential format not supported |
| `invalid_presentation_definition_uri` | Cannot fetch presentation definition |
| `invalid_presentation_definition` | Presentation definition is invalid |

Error response (direct_post):

```http
POST /callback HTTP/1.1
Content-Type: application/x-www-form-urlencoded

error=vp_formats_not_supported&
error_description=The wallet does not support the requested credential format&
state=af0ifjsldkj
```

---

## Specification References

| Topic | OID4VP Section |
|-------|----------------|
| Authorization Request | Section 5 |
| Client Identifier | Section 5.1 |
| Client ID Schemes | Section 5.1.1 |
| Client Metadata | Section 5.1.2 |
| Presentation Definition | Section 5.4 |
| DCQL | Section 5.5 |
| Request Object | Section 5.6 |
| Response Modes | Section 6 |
| Error Responses | Section 6.3 |

### Related Standards

| Standard | Relevance |
|----------|-----------|
| RFC 6749 | OAuth 2.0 Authorization Framework |
| RFC 9101 | JWT-Secured Authorization Request (JAR) |
| DIF Presentation Exchange | Presentation Definition format |
| ISO 18013-5 | mDoc request/response for proximity |
