# Issuer Response -- Conceptual Overview

After the wallet sends a credential request, the issuer processes it and returns a response. The response determines whether the wallet receives the credential immediately, must wait for asynchronous delivery, or encounters an error. This document covers the possible outcomes and the protocol mechanisms that govern each path.

---

## Response Outcomes

The issuer's response to a credential request falls into one of several categories.

### Immediate Credential Delivery (Happy Path)

In the simplest and most common case, the issuer validates the request, constructs the credential, and returns it immediately in the response body. The wallet receives the credential, validates its structure, and stores it.

```
Wallet                                   Issuer
  |                                        |
  |  POST /credential (proof, format)      |
  |--------------------------------------->|
  |                                        |  Validate request
  |                                        |  Construct credential
  |  200 OK { credential, c_nonce }        |
  |<---------------------------------------|
  |                                        |
  |  Validate credential                   |
  |  Store credential                      |
  |  Display success                       |
  |                                        |
```

The response includes the credential in the requested format and may include a new `c_nonce` for subsequent requests.

### Deferred Issuance

When the issuer cannot issue the credential immediately -- because it requires additional verification, manual approval, or asynchronous processing -- it returns a `transaction_id` instead of a credential. The wallet must store this identifier and poll the deferred credential endpoint later.

```
Wallet                                   Issuer
  |                                        |
  |  POST /credential (proof, format)      |
  |--------------------------------------->|
  |                                        |  Validate request
  |                                        |  Cannot issue immediately
  |  200 OK { transaction_id, c_nonce }    |
  |<---------------------------------------|
  |                                        |
  |  Store transaction_id                  |
  |  Display "In Progress" status          |
  |                                        |
  |  ... time passes ...                   |
  |                                        |
  |  POST /deferred-credential             |
  |  { transaction_id }                    |
  |--------------------------------------->|
  |                                        |  Check if ready
  |  200 OK { credential }                 |
  |<---------------------------------------|
  |                                        |
  |  Store credential                      |
  |  Update status to "Issued"             |
  |                                        |
```

**Why deferred issuance exists:**

- **Manual review** -- The issuer requires a human to review and approve the credential (e.g., a government document that requires identity verification review).
- **External data dependencies** -- The credential's claims depend on data from a third party that is not available in real time.
- **Batch processing** -- The issuer processes credential requests in batches on a schedule rather than individually.
- **Compliance checks** -- Regulatory or policy checks that take time to complete.

### Batch Credential Response

When the wallet requests multiple credential instances (via the `proofs` field with multiple JWTs), the issuer returns an array of credentials:

```json
{
  "credentials": [
    { "credential": "<credential-1>" },
    { "credential": "<credential-2>" },
    { "credential": "<credential-3>" }
  ],
  "c_nonce": "new-nonce-value",
  "c_nonce_expires_in": 86400
}
```

Each credential in the array corresponds to a proof in the request array, matched by index. Each credential instance is bound to a different key pair, enabling one-time-use credential policies where each instance is presented once and discarded.

### Partial Success

In batch or multi-credential requests, some credentials may be issued successfully while others fail. The response indicates which credentials were issued and which were not, allowing the wallet to handle each outcome independently.

This is particularly relevant when multiple credential types are requested in a single offer -- one type may be issued immediately while another is deferred or rejected.

---

## c_nonce Rotation

The issuer may include a new `c_nonce` in the credential response, even when the credential is issued successfully. This rotated nonce replaces the previous one and must be used for any subsequent credential requests in the same session.

```json
{
  "credential": "<issued-credential>",
  "c_nonce": "new-nonce-value",
  "c_nonce_expires_in": 86400
}
```

**Why rotate nonces:**

- **Replay prevention** -- A fresh nonce ensures that each credential request uses a unique proof, even if the wallet makes multiple requests within the same session.
- **Session continuity** -- The rotated nonce maintains the session binding without requiring a new token exchange.
- **Security hygiene** -- Short-lived nonces limit the window during which a captured nonce could be exploited.

The `c_nonce_expires_in` field specifies the lifetime of the new nonce in seconds. The wallet must use the new nonce before it expires or request a fresh one from the token endpoint.

---

## Notification Endpoint

The notification endpoint allows the wallet to inform the issuer about credential lifecycle events after issuance. This is a post-issuance communication channel that provides the issuer with visibility into what happens to credentials after they are delivered.

### How It Works

1. The issuer includes a `notification_id` in the credential response.
2. The wallet accepts, stores, and uses the credential.
3. At some point (immediately or later), the wallet sends a notification to the issuer's notification endpoint.

```
Wallet                                   Issuer
  |                                        |
  |  Credential received (with             |
  |    notification_id: "abc123")          |
  |                                        |
  |  ... wallet processes credential ...   |
  |                                        |
  |  POST /notification                    |
  |  { notification_id: "abc123",          |
  |    event: "credential_accepted" }      |
  |--------------------------------------->|
  |                                        |  Update credential status
  |  204 No Content                        |
  |<---------------------------------------|
  |                                        |
```

### Notification Events

| Event | Meaning |
|-------|---------|
| `credential_accepted` | The wallet successfully stored the credential. |
| `credential_deleted` | The user deleted the credential from the wallet. |
| `credential_failure` | The wallet failed to process or store the credential. |

### Purpose

Notifications serve several purposes for the issuer:

- **Issuance confirmation** -- The issuer knows the credential was successfully received and stored.
- **Audit trail** -- The issuer can log credential lifecycle events for compliance.
- **Cleanup** -- If the wallet reports failure, the issuer can revoke the credential or initiate re-issuance.
- **Analytics** -- The issuer can track credential usage patterns (acceptance rate, deletion rate).

---

## Error Responses

When the issuer cannot process the credential request, it returns an error response with an error code and optional description.

### Common Error Codes

| Error Code | HTTP Status | Meaning |
|------------|-------------|---------|
| `invalid_proof` | 400 | The proof of possession is invalid. The JWT signature verification failed, the nonce is wrong or expired, or the audience does not match. |
| `invalid_token` | 401 | The access token is invalid, expired, or revoked. The wallet must re-authenticate and obtain a new token. |
| `unsupported_credential_type` | 400 | The issuer does not support the requested credential type. |
| `unsupported_credential_format` | 400 | The issuer does not support the requested format. |
| `invalid_credential_request` | 400 | The request body is malformed or missing required fields. |
| `credential_request_denied` | 403 | The issuer has denied the credential request based on policy. |

### Error with Nonce Refresh

A common pattern is the `invalid_proof` error with a new `c_nonce`:

```json
{
  "error": "invalid_proof",
  "error_description": "Nonce expired",
  "c_nonce": "new-nonce-value",
  "c_nonce_expires_in": 300
}
```

This response tells the wallet that the proof failed because the nonce was expired, but provides a fresh nonce that the wallet can use to construct a new proof and retry the request. This is a recovery path -- the wallet does not need to restart the entire issuance flow.

---

## Deferred Credential Retrieval

When the issuer returns a `transaction_id` instead of a credential, the wallet must later poll the deferred credential endpoint.

### Deferred Credential Endpoint

```
POST /deferred-credential HTTP/1.1
Host: issuer.example.com
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "transaction_id": "8xLOxBtZp8"
}
```

### Possible Outcomes

| Outcome | Response | Wallet Action |
|---------|----------|---------------|
| Credential ready | `200 OK` with `credential` | Store the credential, update status |
| Still processing | `400` with `issuance_pending` error | Wait and retry after the `interval` period |
| Denied | `400` with `credential_request_denied` | Display failure, remove deferred record |
| Expired | `400` with `invalid_transaction_id` | Display failure, offer to restart issuance |

### Polling Behavior

The deferred response may include an `interval` field (in seconds) that specifies the minimum time the wallet should wait before polling again. The wallet must respect this interval to avoid overwhelming the issuer's endpoint. If no interval is specified, the wallet uses a reasonable default (implementation-specific).

---

## What Happens Next

After the wallet receives a credential (immediately, via deferred retrieval, or through batch response), it must securely store the credential on the device. The storage mechanisms and security properties are covered in [10 -- Secure Storage](../10-secure-storage/).
