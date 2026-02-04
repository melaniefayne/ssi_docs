# Verifier Metadata — Protocol and Standards

This document details the protocol-level mechanisms for verifier identification and trust establishment as defined in OID4VP and ISO 18013-5.

---

## OID4VP Client Metadata

### Client Metadata Parameters

The verifier provides metadata either inline or by reference. OID4VP defines these parameters:

```json
{
  "client_name": "Example Verifier",
  "client_name#de": "Beispiel Prüfer",
  "logo_uri": "https://verifier.example.com/logo.png",
  "client_uri": "https://verifier.example.com",
  "policy_uri": "https://verifier.example.com/privacy",
  "tos_uri": "https://verifier.example.com/terms",
  "contacts": ["support@verifier.example.com"],

  "vp_formats": {
    "mso_mdoc": {},
    "vc+sd-jwt": {
      "sd-jwt_alg_values": ["ES256", "ES384"],
      "kb-jwt_alg_values": ["ES256"]
    },
    "jwt_vp_json": {
      "alg_values_supported": ["ES256"]
    }
  },

  "response_types_supported": ["vp_token"],
  "response_modes_supported": ["direct_post", "direct_post.jwt"],

  "authorization_signed_response_alg": "ES256",
  "authorization_encrypted_response_alg": "ECDH-ES",
  "authorization_encrypted_response_enc": "A256GCM"
}
```

### VP Formats

The `vp_formats` object specifies which credential formats the verifier accepts:

| Format | Description | Parameters |
|--------|-------------|------------|
| `mso_mdoc` | ISO 18013-5 mDoc | (none, implied COSE) |
| `vc+sd-jwt` | SD-JWT Verifiable Credential | `sd-jwt_alg_values`, `kb-jwt_alg_values` |
| `jwt_vp_json` | JWT-encoded VP with JSON payload | `alg_values_supported` |
| `ldp_vp` | JSON-LD VP with Data Integrity | `proof_type_values_supported` |

### Metadata Delivery

**Option 1: Inline (`client_metadata`)**

```http
GET /authorize?
  client_id=https://verifier.example.com&
  client_metadata=%7B%22client_name%22%3A%22Example%22...%7D&
  ...
```

**Option 2: By Reference (`client_metadata_uri`)**

```http
GET /authorize?
  client_id=https://verifier.example.com&
  client_metadata_uri=https://verifier.example.com/.well-known/openid-client&
  ...
```

Wallet fetches:

```http
GET /.well-known/openid-client HTTP/1.1
Host: verifier.example.com
Accept: application/json

HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: max-age=3600

{
  "client_name": "Example Verifier",
  ...
}
```

---

## Client ID Schemes

OID4VP supports multiple schemes for verifier identification via the `client_id_scheme` parameter:

### redirect_uri

```json
{
  "client_id_scheme": "redirect_uri",
  "client_id": "https://verifier.example.com/callback",
  "response_uri": "https://verifier.example.com/callback"
}
```

- `client_id` equals the `response_uri`
- No cryptographic verification
- Trust based solely on HTTPS

### x509_san_dns

```json
{
  "client_id_scheme": "x509_san_dns",
  "client_id": "verifier.example.com"
}
```

- Request signed as JWT
- JWT header contains `x5c` (certificate chain)
- Certificate SAN DNS entry must match `client_id`
- Wallet validates certificate chain

**JWT Structure:**

```
Header:
{
  "alg": "ES256",
  "typ": "oauth-authz-req+jwt",
  "x5c": [
    "MIIB...verifier-cert...",
    "MIIC...intermediate-ca...",
    "MIID...root-ca..."
  ]
}

Payload:
{
  "client_id_scheme": "x509_san_dns",
  "client_id": "verifier.example.com",
  ...
}
```

**Validation Steps:**

1. Decode `x5c` certificate chain
2. Validate chain cryptographically
3. Check root against trusted certificates
4. Extract SAN from leaf certificate
5. Verify `client_id` matches SAN DNS entry
6. Verify JWT signature with leaf certificate public key

### x509_san_uri

Similar to `x509_san_dns` but uses URI SAN:

```json
{
  "client_id_scheme": "x509_san_uri",
  "client_id": "https://verifier.example.com"
}
```

### did

```json
{
  "client_id_scheme": "did",
  "client_id": "did:web:verifier.example.com"
}
```

- Request signed as JWT
- Wallet resolves DID to DID Document
- Validates signature against DID verification method
- Trust depends on DID method and resolution

**Validation Steps:**

1. Resolve `client_id` DID to DID Document
2. Extract verification method referenced in JWT `kid`
3. Validate JWT signature
4. Trust based on DID method properties

### verifier_attestation

```json
{
  "client_id_scheme": "verifier_attestation",
  "client_id": "example-verifier-id"
}
```

- Request includes `verifier_attestation` JWT in header
- Attestation issued by trusted party
- Wallet validates attestation signature and issuer

**Attestation JWT:**

```
Header:
{
  "alg": "ES256",
  "typ": "verifier-attestation+jwt"
}

Payload:
{
  "iss": "https://trust-registry.example.com",
  "sub": "example-verifier-id",
  "iat": 1683000000,
  "exp": 1683086400,
  "cnf": {
    "jwk": { ... verifier public key ... }
  }
}
```

### pre-registered

```json
{
  "client_id_scheme": "pre-registered",
  "client_id": "client-123"
}
```

- Verifier pre-registered with wallet/ecosystem
- `client_id` matches pre-registered identifier
- Trust established during registration

---

## ISO 18013-5 Reader Authentication

For mDL proximity presentation, reader authentication is defined in ISO 18013-5.

### SessionTranscript

Binds authentication to the specific session:

```
SessionTranscript = [
  DeviceEngagementBytes,
  EReaderKeyBytes,
  Handover
]
```

### ReaderAuthentication Structure

```
ReaderAuthentication = [
  "ReaderAuthentication",
  SessionTranscript,
  ItemsRequestBytes
]
```

### COSE_Sign1 Structure

```
COSE_Sign1 = [
  protected: {
    1: -7,  // alg: ES256
    33: h'...'  // x5chain: certificate chain
  },
  unprotected: {},
  payload: ReaderAuthentication (CBOR encoded),
  signature: bytes
]
```

### Certificate Requirements

The reader certificate must:

| Requirement | ISO 18013-5 Reference |
|-------------|----------------------|
| Valid at current time | 9.1.4 |
| Chain to trusted root | 9.1.4 |
| Extended Key Usage: Reader Authentication | Annex B |
| Not revoked | 9.1.4 |

### Certificate Profile

```
Certificate:
  Subject: CN=Example Reader
  Subject Alternative Name:
    - URI: https://reader.example.com
  Extended Key Usage:
    - 1.0.18013.5.1.6 (mdoc Reader Authentication)
  Key Usage:
    - Digital Signature
```

---

## Trust Framework Integration

### EUDI Trust Framework

The EUDI ecosystem defines a trust framework with:

1. **Trust Lists**: Lists of trusted issuers and verifiers
2. **Qualified Certificates**: eIDAS qualified certificates for high assurance
3. **Registered Relying Parties**: Pre-approved verifiers

### Reader Certificate Trust

Wallets maintain trusted root certificates:

```
Trusted Roots:
├── EU Digital Identity Root CA
├── Member State CA (DE)
├── Member State CA (FR)
└── ... (per member state)
```

Verifier certificates chain to these roots.

### Dynamic Trust Updates

Trust lists may be updated:

```http
GET /trust-list/verifiers HTTP/1.1
Host: trust-registry.example.com

HTTP/1.1 200 OK
Content-Type: application/json

{
  "version": "2024-01-15",
  "trusted_verifiers": [
    {
      "client_id": "did:web:bank.example.com",
      "name": "Example Bank",
      "valid_until": "2025-01-15T00:00:00Z"
    }
  ]
}
```

---

## Response Encryption

Verifier can request encrypted responses:

```json
{
  "client_metadata": {
    "authorization_encrypted_response_alg": "ECDH-ES",
    "authorization_encrypted_response_enc": "A256GCM",
    "jwks": {
      "keys": [{
        "kty": "EC",
        "crv": "P-256",
        "x": "...",
        "y": "...",
        "use": "enc"
      }]
    }
  }
}
```

The VP Token is encrypted to the verifier's public key, ensuring only the verifier can read the presentation.

---

## Standards Reference

| Standard | Section | Topic |
|----------|---------|-------|
| OID4VP | 5.1 | Client Metadata |
| OID4VP | 5.2 | Client ID Schemes |
| OID4VP | 9 | Verifier Attestation |
| ISO 18013-5 | 9.1.4 | Reader Authentication |
| ISO 18013-5 | Annex B | Certificate Profiles |
| RFC 8725 | - | JWT Best Practices |
| RFC 7517 | - | JSON Web Key |
