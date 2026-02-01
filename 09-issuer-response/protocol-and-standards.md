# Issuer Response -- Protocol & Standards

This document specifies the technical structure of OID4VCI credential responses, deferred credential endpoint behavior, notification endpoint mechanics, and error response formats.

---

## Credential Response

The issuer's response to a valid credential request is an HTTP 200 response with a JSON body.

### Immediate Credential Response

```json
{
  "credential": "<format-specific-credential>",
  "c_nonce": "fGFF7UkhLa",
  "c_nonce_expires_in": 86400,
  "notification_id": "3fwe98fj"
}
```

**Fields:**

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `credential` | Yes (unless deferred) | String or Object | The issued credential in the requested format. The type depends on the format: a string for `mso_mdoc` (base64url-encoded CBOR) and `vc+sd-jwt` (compact serialization), or a JSON object for `ldp_vc`. |
| `c_nonce` | No | String | A new nonce value that replaces the previous `c_nonce`. The wallet must use this nonce in any subsequent credential request proofs within the same session. |
| `c_nonce_expires_in` | No | Number | The lifetime of the new `c_nonce` in seconds. If omitted, the nonce lifetime is determined by issuer policy. |
| `notification_id` | No | String | An identifier that the wallet should include when sending a notification to the issuer's notification endpoint. Only present when the issuer supports the notification endpoint. |

### Format-Specific Credential Values

**mso_mdoc:**

The `credential` field contains a base64url-encoded CBOR-encoded `IssuerSigned` structure as defined in ISO 18013-5:

```json
{
  "credential": "o2d2ZXJzaW9uYzEuMGlkb2N1bWVudHOB..."
}
```

**SD-JWT VC:**

The `credential` field contains the SD-JWT in compact serialization:

```json
{
  "credential": "eyJhbGciOiJFUzI1NiIsInR5cCI6InZjK3NkLWp3dCJ9.eyJpc3Mi..."
}
```

**JSON-LD VC:**

The `credential` field contains a JSON object:

```json
{
  "credential": {
    "@context": ["https://www.w3.org/2018/credentials/v1"],
    "type": ["VerifiableCredential", "UniversityDegreeCredential"],
    "issuer": "did:web:issuer.example.com",
    "credentialSubject": { ... },
    "proof": { ... }
  }
}
```

---

## Deferred Credential Response

When the issuer cannot issue the credential immediately, it returns a deferred response instead.

### Initial Response (Deferred)

```json
{
  "transaction_id": "8xLOxBtZp8",
  "c_nonce": "fGFF7UkhLa",
  "c_nonce_expires_in": 86400
}
```

**Fields:**

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `transaction_id` | Yes | String | A unique identifier for the deferred issuance transaction. The wallet uses this to retrieve the credential later. |
| `c_nonce` | No | String | A nonce for potential subsequent requests. |
| `c_nonce_expires_in` | No | Number | Lifetime of the nonce in seconds. |

The presence of `transaction_id` without `credential` signals deferred issuance. The wallet must persist the `transaction_id` for later retrieval.

### Deferred Credential Endpoint

The wallet retrieves deferred credentials by sending a POST request to the deferred credential endpoint (declared in issuer metadata as `deferred_credential_endpoint`).

**Request:**

```
POST /deferred-credential HTTP/1.1
Host: issuer.example.com
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "transaction_id": "8xLOxBtZp8"
}
```

The request is authenticated with the same access token used for the original credential request. If the access token has expired, the wallet must obtain a new one (using a refresh token if available).

**Successful Response:**

```json
{
  "credential": "<issued-credential>"
}
```

When the credential is ready, the issuer returns it in the same format as an immediate credential response.

**Pending Response:**

```
HTTP/1.1 400 Bad Request
Content-Type: application/json

{
  "error": "issuance_pending",
  "interval": 5
}
```

| Field | Type | Description |
|-------|------|-------------|
| `error` | String | `issuance_pending` indicates the credential is still being processed. |
| `interval` | Number | Minimum number of seconds the wallet should wait before polling again. If omitted, the wallet uses a default polling interval. |

The wallet should respect the `interval` value and not poll more frequently. Implementations typically use exponential backoff or the issuer-specified interval, whichever is larger.

**Denied Response:**

```json
{
  "error": "credential_request_denied",
  "error_description": "Credential issuance was denied after review."
}
```

This terminal response indicates the credential will not be issued. The wallet should remove the deferred record and inform the user.

---

## Batch Credential Response

When the wallet sends a credential request with multiple proofs (the `proofs` field), the issuer returns a batch response.

### Response Structure

```json
{
  "credentials": [
    {
      "credential": "<credential-1>"
    },
    {
      "credential": "<credential-2>"
    },
    {
      "credential": "<credential-3>"
    }
  ],
  "c_nonce": "rotated-nonce",
  "c_nonce_expires_in": 86400
}
```

**Fields:**

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `credentials` | Yes | Array | An array of credential objects, one per proof in the request. Each object contains a `credential` field with the issued credential. |
| `c_nonce` | No | String | Rotated nonce for subsequent requests. |
| `c_nonce_expires_in` | No | Number | Lifetime of the rotated nonce. |

### Index Correspondence

The `credentials` array elements correspond to the `proofs.jwt` array elements by index:

```
proofs.jwt[0]  -->  credentials[0].credential  (bound to key from proof 0)
proofs.jwt[1]  -->  credentials[1].credential  (bound to key from proof 1)
proofs.jwt[2]  -->  credentials[2].credential  (bound to key from proof 2)
```

Each credential is bound to the public key from the corresponding proof JWT.

---

## Notification Endpoint

The notification endpoint allows the wallet to inform the issuer about credential lifecycle events.

### Endpoint Discovery

The notification endpoint URL is declared in the issuer's metadata:

```json
{
  "credential_issuer": "https://issuer.example.com",
  "notification_endpoint": "https://issuer.example.com/notification",
  ...
}
```

### Request

```
POST /notification HTTP/1.1
Host: issuer.example.com
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "notification_id": "3fwe98fj",
  "event": "credential_accepted"
}
```

**Fields:**

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `notification_id` | Yes | String | The identifier provided by the issuer in the credential response. |
| `event` | Yes | String | The lifecycle event being reported. |

### Event Types

| Event | Description |
|-------|-------------|
| `credential_accepted` | The wallet successfully processed, validated, and stored the credential. The issuance is complete from the wallet's perspective. |
| `credential_deleted` | The user or wallet deleted the credential. This may trigger issuer-side cleanup or revocation. |
| `credential_failure` | The wallet failed to process the credential (e.g., validation failure, storage error). The issuer may revoke the credential or offer re-issuance. |

### Response

The issuer responds with `204 No Content` on success. Error responses follow standard HTTP error conventions.

### Timing

The specification does not mandate when the wallet must send notifications. Implementations may:

- Send `credential_accepted` immediately after storing the credential.
- Send `credential_deleted` when the user explicitly deletes the credential.
- Send `credential_failure` immediately upon encountering a processing error.
- Batch notifications and send them periodically.

---

## Error Responses

Error responses follow a standard structure defined by the OID4VCI specification.

### Error Response Format

```json
{
  "error": "<error-code>",
  "error_description": "<human-readable-description>",
  "c_nonce": "<optional-new-nonce>",
  "c_nonce_expires_in": 300
}
```

The `c_nonce` and `c_nonce_expires_in` fields may be included in error responses, particularly for `invalid_proof` errors where the nonce was the cause of failure. This allows the wallet to retry with a fresh nonce without starting over.

### Error Codes

| Error Code | HTTP Status | Description | Recovery Path |
|------------|-------------|-------------|---------------|
| `invalid_proof` | 400 | The proof of possession failed validation. Possible causes: signature invalid, nonce mismatch, nonce expired, audience mismatch, timestamp out of window. | If a new `c_nonce` is provided, reconstruct the proof with the new nonce and retry. Otherwise, restart from token exchange. |
| `invalid_token` | 401 | The access token is invalid, expired, or revoked. | Obtain a new access token via token exchange (using refresh token if available). |
| `unsupported_credential_type` | 400 | The issuer does not support the requested credential type. The `credential_configuration_id` or format + type combination does not match any supported configuration. | Check issuer metadata for supported types and retry with a valid type. |
| `unsupported_credential_format` | 400 | The requested format is not supported by the issuer. | Check issuer metadata for supported formats and retry with a valid format. |
| `invalid_credential_request` | 400 | The request body is malformed. Required fields are missing, field types are wrong, or the structure does not conform to the specification. | Fix the request body and retry. |
| `credential_request_denied` | 403 | The issuer denied the request based on policy (e.g., the user is not eligible, the authorization context does not permit this credential). | No automated recovery. The user may need to contact the issuer. |
| `issuance_pending` | 400 | (Deferred endpoint only) The credential is still being processed. | Wait for the `interval` duration and poll again. |
| `invalid_transaction_id` | 400 | (Deferred endpoint only) The `transaction_id` is invalid or expired. | The deferred issuance has failed. The user may need to restart the issuance flow. |

---

## c_nonce Rotation in Responses

The `c_nonce` lifecycle spans both token responses and credential responses. Understanding the rotation rules is essential for correct protocol implementation.

### Rotation Rules

1. **Initial nonce** -- The first `c_nonce` is provided in the token response.
2. **Consumed in proof** -- The wallet includes the `c_nonce` as the `nonce` claim in the JWT proof.
3. **Rotated in credential response** -- The credential response may include a new `c_nonce` that replaces the previous one.
4. **Rotated in error response** -- An `invalid_proof` error response may include a new `c_nonce`, providing a recovery path.
5. **Expiration** -- The `c_nonce_expires_in` field defines the lifetime. The wallet must use the nonce before it expires.

### Implications

- The wallet must always use the **most recently received** `c_nonce`. If a credential response includes a new nonce, the previous nonce is invalidated.
- If multiple credential requests are sent in sequence (e.g., for different credential types in the same offer), each response's `c_nonce` must be used for the next request.
- If the nonce expires between requests, the wallet must obtain a new one (typically by making a request that triggers a nonce refresh).

---

## Summary

The OID4VCI credential response specification defines a flexible framework that handles immediate delivery, deferred issuance, batch responses, and error recovery. The `c_nonce` rotation mechanism maintains session freshness across multiple requests. The notification endpoint provides post-issuance lifecycle visibility. Error responses include recovery paths (particularly the `invalid_proof` error with nonce refresh) that allow the wallet to retry without restarting the entire flow.
