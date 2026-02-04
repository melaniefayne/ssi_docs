# Presentation Response — Conceptual Overview

The presentation response is the message the wallet sends back to the verifier containing the Verifiable Presentation (VP Token) and submission metadata.

---

## Response Components

```
┌─────────────────────────────────────────────────────────────────┐
│                    Presentation Response                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  VP Token                                                   ││
│  │  ├── Holder proof (binds presentation to request)          ││
│  │  └── Verifiable Credentials (with selective disclosure)    ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  Presentation Submission                                    ││
│  │  └── Maps VP contents to request's input descriptors        ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  State                                                      ││
│  │  └── Echoed from request for session correlation            ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## VP Token Structure

### JWT VP

```
eyJhbGciOiJFUzI1NiIsInR5cCI6IkpXVCJ9.
eyJpc3MiOiJkaWQ6a2V5Ono2TWsuLi4iLCJhdWQiOiJodHRwczovL3ZlcmlmaWVyLmV4YW1wbGUuY29tIiwibm9uY2UiOiJuLTBTNl9XekEyTWoiLCJ2cCI6eyJAY29udGV4dCI6WyJodHRwczovL3d3dy53My5vcmcvMjAxOC9jcmVkZW50aWFscy92MSJdLCJ0eXBlIjpbIlZlcmlmaWFibGVQcmVzZW50YXRpb24iXSwidmVyaWZpYWJsZUNyZWRlbnRpYWwiOlsiZXlKLi4uIl19fQ.
[signature]
```

### SD-JWT with Key Binding

```
[issuer-jwt]~[disclosure1]~[disclosure2]~[kb-jwt]

Where:
- issuer-jwt: Original credential from issuer
- disclosures: Only the disclosed claims
- kb-jwt: Key binding proof from holder
```

### mDoc DeviceResponse

```cbor
{
  "version": "1.0",
  "documents": [{
    "docType": "org.iso.18013.5.1.mDL",
    "issuerSigned": {
      "nameSpaces": {
        "org.iso.18013.5.1": [...]  // Selected elements only
      },
      "issuerAuth": [...]  // COSE_Sign1 with MSO
    },
    "deviceSigned": {
      "nameSpaces": {},
      "deviceAuth": [...]  // COSE_Sign1 device proof
    }
  }],
  "status": 0
}
```

---

## Presentation Submission

Maps the VP contents to the original request:

```json
{
  "id": "submission-uuid",
  "definition_id": "request-definition-id",
  "descriptor_map": [
    {
      "id": "age_verification",
      "format": "vc+sd-jwt",
      "path": "$",
      "path_nested": {
        "format": "vc+sd-jwt",
        "path": "$.vp.verifiableCredential[0]"
      }
    }
  ]
}
```

### Descriptor Map Entry

| Field | Purpose |
|-------|---------|
| `id` | References input descriptor from request |
| `format` | Credential format used |
| `path` | JSONPath to credential in response |
| `path_nested` | For nested structures (VP containing VCs) |

---

## Response Delivery

### direct_post Mode

```http
POST /callback HTTP/1.1
Host: verifier.example.com
Content-Type: application/x-www-form-urlencoded

vp_token=eyJ...&
presentation_submission=%7B%22id%22%3A...%7D&
state=af0ifjsldkj
```

### Verifier Response

After receiving the VP:

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "redirect_uri": "https://verifier.example.com/success?session=xyz"
}
```

The wallet then navigates to the `redirect_uri`.

---

## Response Flow

```
Wallet                                         Verifier
  │                                               │
  │  Prepare response:                            │
  │  - Construct VP Token                         │
  │  - Build presentation_submission              │
  │  - Include state from request                 │
  │                                               │
  │  POST vp_token, presentation_submission       │
  │──────────────────────────────────────────────►│
  │                                               │
  │                                   Verify:     │
  │                                   - Signatures│
  │                                   - Nonce     │
  │                                   - Claims    │
  │                                               │
  │               200 OK + redirect_uri           │
  │◄──────────────────────────────────────────────│
  │                                               │
  │  Navigate to redirect_uri                     │
  │                                               │
```

---

## Error Responses

When presentation cannot be completed:

```http
POST /callback HTTP/1.1
Host: verifier.example.com
Content-Type: application/x-www-form-urlencoded

error=access_denied&
error_description=User+declined+to+share+credentials&
state=af0ifjsldkj
```

### Error Codes

| Code | Meaning |
|------|---------|
| `access_denied` | User declined |
| `invalid_request` | Malformed request |
| `invalid_scope` | Cannot satisfy request |
| `vp_formats_not_supported` | Format mismatch |
