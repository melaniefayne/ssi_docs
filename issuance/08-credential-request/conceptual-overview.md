# Credential Request -- Conceptual Overview

The **Credential Request** is the protocol message that the wallet sends to the issuer to request the issuance of one or more credentials. It is sent as an HTTP POST to the issuer's credential endpoint, authenticated with an access token and accompanied by a proof of possession. The request specifies what credential the wallet wants, in what format, and with what key binding.

---

## The Credential Endpoint

The credential endpoint is the issuer's HTTP endpoint that receives credential requests and returns credentials. Its URL is declared in the issuer's metadata as `credential_endpoint`.

```
POST /credential HTTP/1.1
Host: issuer.example.com
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "credential_configuration_id": "eu.europa.ec.eudi.pid.1",
  "proof": {
    "proof_type": "jwt",
    "jwt": "eyJhbGciOiJFUzI1NiIs..."
  }
}
```

The request is authenticated via the `Authorization` header, which carries the access token obtained during the token exchange phase. If DPoP is used, a `DPoP` header is also included.

---

## Request Contents

The credential request body contains three categories of information:

### 1. Credential Identification

The wallet must tell the issuer which credential to issue. This is done through one of two mechanisms:

- **By configuration identifier** -- The wallet includes a `credential_configuration_id` (v1.0 Final) or `credential_identifier` (Draft 13) that references an entry in the issuer's `credential_configurations_supported` metadata. This is the preferred approach when the wallet has resolved the credential offer or metadata.
- **By format and type** -- The wallet includes the `format` field along with format-specific parameters (`doctype` for mDoc, `vct` for SD-JWT VC) that identify the credential. This approach is used when the wallet does not have a pre-assigned identifier.

### 2. Proof of Possession

The `proof` field contains the cryptographic proof that the wallet controls the private key to be bound to the credential. For JWT proofs:

```json
{
  "proof": {
    "proof_type": "jwt",
    "jwt": "<signed-jwt>"
  }
}
```

For batch issuance (multiple credential instances), the `proofs` (plural) field is used with an array of JWTs:

```json
{
  "proofs": {
    "jwt": ["<jwt-1>", "<jwt-2>", "<jwt-3>"]
  }
}
```

See [07 -- Proof Construction](../07-proof-construction/) for details on proof structure and signing.

### 3. Format-Specific Parameters

Depending on the credential format, additional parameters may be included:

| Format | Parameter | Example |
|--------|-----------|---------|
| `mso_mdoc` | `doctype` | `"org.iso.18013.5.1.mDL"` |
| `vc+sd-jwt` | `vct` | `"VerifiablePortableDocumentA1"` |
| `jwt_vc_json` | `credential_definition` | `{ "type": ["VerifiableCredential", "UniversityDegreeCredential"] }` |
| `ldp_vc` | `credential_definition` | `{ "type": ["VerifiableCredential", "PermanentResidentCard"] }` |

---

## Issuer Validation

When the issuer receives a credential request, it performs a sequence of validations before issuing the credential:

### 1. Access Token Validation

The issuer verifies the access token:

- **Token validity** -- The token has not expired and has not been revoked.
- **Token scope** -- The token authorizes issuance of the requested credential type.
- **Sender constraint** -- If DPoP is used, the issuer verifies the DPoP proof and confirms the token is bound to the presenter's key.

### 2. Proof Validation

The issuer verifies the proof of possession:

- **Signature** -- The JWT signature is valid against the included public key.
- **Nonce** -- The `nonce` claim matches the `c_nonce` issued for this session.
- **Audience** -- The `aud` claim matches the issuer's identifier.
- **Freshness** -- The `iat` claim is within an acceptable time window.
- **Key attestation** -- If required (HAIP profile), the issuer verifies the key attestation to confirm the key is hardware-backed.

### 3. Format and Type Validation

The issuer confirms it supports the requested credential format and type:

- The `credential_configuration_id` (or format + type) matches a supported credential configuration.
- The requested claims or attributes are within the issuer's capability.

### 4. Authorization Context

The issuer confirms the credential request is authorized in the context of the original grant:

- For pre-authorized code flows, the credential type must match what was offered.
- For authorization code flows, the credential type must be within the authorized scope.

---

## Response Types

After successful validation, the issuer responds with one of several outcomes.

### Immediate Credential Delivery (Happy Path)

The issuer returns the credential directly in the response:

```json
{
  "credential": "<issued-credential>",
  "c_nonce": "fGFF7UkhLa",
  "c_nonce_expires_in": 86400
}
```

The `credential` field contains the issued credential in the requested format (mDoc CBOR, SD-JWT string, JSON-LD VC object). The response may include a new `c_nonce` for subsequent requests.

### Deferred Issuance

If the issuer cannot issue the credential immediately (e.g., it requires additional verification or manual approval), it returns a `transaction_id` instead:

```json
{
  "transaction_id": "8xLOxBtZp8",
  "c_nonce": "fGFF7UkhLa",
  "c_nonce_expires_in": 86400
}
```

The wallet stores the `transaction_id` and polls the deferred credential endpoint later to retrieve the credential. See [09 -- Issuer Response](../09-issuer-response/) for details on deferred issuance handling.

### Batch Credential Response

When multiple credential instances are requested (via `proofs` with multiple JWTs), the issuer returns an array of credentials:

```json
{
  "credentials": [
    { "credential": "<credential-1>" },
    { "credential": "<credential-2>" },
    { "credential": "<credential-3>" }
  ],
  "c_nonce": "fGFF7UkhLa",
  "c_nonce_expires_in": 86400
}
```

Each credential in the array is bound to the corresponding key from the `proofs` array (by index).

---

## Batch Issuance

Batch issuance allows the wallet to request multiple instances of the same credential type in a single request. This is particularly important for **one-time-use credential policies**, where each credential instance is intended for a single presentation and is then discarded.

### Why Batch Issuance

Without batch issuance, a one-time-use credential policy would require a full issuance round-trip for every presentation. Batch issuance pre-provisions multiple instances so the wallet has a pool of credentials available for presentation without repeated issuance requests.

### How It Works

1. The wallet generates multiple key pairs (one per credential instance).
2. The wallet constructs a proof JWT for each key pair.
3. The wallet sends a single credential request with all proofs in the `proofs` array.
4. The issuer validates each proof and issues one credential per proof, each bound to a different key.
5. The wallet stores all credential instances.

### Credential Policies

Implementations typically define credential policies that determine batch behavior:

- **Rotate use** (default) -- A single credential is issued and reused across presentations. Batch size: 1.
- **One-time use** -- Multiple credential instances are issued, each used once. Batch size: configurable (e.g., 10 for PID credentials).

---

## Error Responses

The issuer returns standard error responses for invalid requests:

| Error Code | Meaning |
|------------|---------|
| `invalid_proof` | The proof of possession is invalid (bad signature, wrong nonce, expired). |
| `invalid_token` | The access token is invalid, expired, or revoked. |
| `unsupported_credential_type` | The issuer does not support the requested credential type or format. |
| `unsupported_credential_format` | The requested format is not supported. |
| `invalid_credential_request` | The request body is malformed or missing required fields. |

---

## What Happens Next

After the issuer processes the credential request, it returns a response. The handling of this response -- whether immediate, deferred, or partial -- is covered in [09 -- Issuer Response](../09-issuer-response/).
