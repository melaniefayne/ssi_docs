# Presentation Response — Protocol and Standards

This document details the OID4VP authorization response structure and delivery mechanisms.

---

## Authorization Response

### Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `vp_token` | Yes | The Verifiable Presentation(s) |
| `presentation_submission` | Yes | Maps VPs to request |
| `state` | If in request | Echoed state parameter |

### Response Delivery

#### direct_post

```http
POST /callback HTTP/1.1
Host: verifier.example.com
Content-Type: application/x-www-form-urlencoded

vp_token=eyJhbGciOiJFUzI1NiIsInR5cCI6IkpXVCJ9...&
presentation_submission=%7B%22id%22%3A%22a30e3b91...%22%7D&
state=af0ifjsldkj
```

#### direct_post.jwt

```http
POST /callback HTTP/1.1
Host: verifier.example.com
Content-Type: application/x-www-form-urlencoded

response=eyJhbGciOiJFQ0RILUVTK0EyNTZLVyIsImVuYyI6IkEyNTZHQ00ifQ...
```

The `response` is a JWE containing the authorization response.

---

## VP Token Formats

### Single VP

```
vp_token=eyJ...  (JWT VP or SD-JWT+KB)
```

### Multiple VPs

```json
{
  "vp_token": [
    "eyJ...",
    "eyJ..."
  ]
}
```

Or URL-encoded array.

---

## Presentation Submission

### Structure

```json
{
  "id": "a30e3b91-fb77-4d22-95fa-871689c322e2",
  "definition_id": "32f54163-7166-48f1-93d8-ff217bdb0653",
  "descriptor_map": [
    {
      "id": "banking_input",
      "format": "jwt_vp_json",
      "path": "$",
      "path_nested": {
        "format": "jwt_vc_json",
        "path": "$.vp.verifiableCredential[0]"
      }
    }
  ]
}
```

### Descriptor Map

| Field | Description |
|-------|-------------|
| `id` | Matches input descriptor ID |
| `format` | VP/VC format identifier |
| `path` | JSONPath to VP in response |
| `path_nested` | Path to VC within VP |

### Format Identifiers

| Format | Identifier |
|--------|------------|
| JWT VP | `jwt_vp_json` |
| JWT VC | `jwt_vc_json` |
| SD-JWT VC | `vc+sd-jwt` |
| mDoc | `mso_mdoc` |
| JSON-LD VP | `ldp_vp` |
| JSON-LD VC | `ldp_vc` |

---

## Verifier Response

After processing the VP Token:

### Success

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "redirect_uri": "https://verifier.example.com/success?session=xyz"
}
```

### Error

```http
HTTP/1.1 400 Bad Request
Content-Type: application/json

{
  "error": "invalid_request",
  "error_description": "Signature verification failed"
}
```

---

## Error Response

When wallet cannot complete:

```
error=access_denied&
error_description=User+declined&
state=af0ifjsldkj
```

### Error Codes

| Code | Description |
|------|-------------|
| `invalid_request` | Malformed request |
| `unauthorized_client` | Client not allowed |
| `access_denied` | User declined |
| `unsupported_response_type` | Response type not supported |
| `invalid_scope` | Cannot fulfill scope |
| `server_error` | Internal error |

---

## mDoc Response

For ISO 18013-5 credentials:

```cbor
DeviceResponse = {
  "version": "1.0",
  "documents": [Document],
  "status": uint
}

Document = {
  "docType": tstr,
  "issuerSigned": IssuerSigned,
  "deviceSigned": DeviceSigned,
  ? "errors": Errors
}
```

---

## Standards Reference

| Standard | Section | Topic |
|----------|---------|-------|
| OID4VP | 6 | Authorization Response |
| OID4VP | 6.2 | VP Token |
| OID4VP | 6.3 | Presentation Submission |
| DIF PEX | 7 | Presentation Submission |
| ISO 18013-5 | 8.3.2.1 | DeviceResponse |
