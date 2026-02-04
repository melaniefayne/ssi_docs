# Presentation Request — Conceptual Overview

A **Presentation Request** is the mechanism by which a Verifier asks a Holder to present one or more credentials. In OID4VP, this is formalized as an Authorization Request that extends OAuth 2.0 with verifiable credential-specific parameters.

---

## The Request's Role

The presentation request serves multiple purposes simultaneously:

1. **Identifies the Verifier** — The Holder knows who is asking for credentials
2. **Specifies Requirements** — What credentials and claims are needed
3. **Establishes Freshness** — A nonce binds the response to this specific request
4. **Defines Response Path** — How and where to send the presentation

---

## Request Structure (Conceptual)

```
┌─────────────────────────────────────────────────────────────┐
│  Presentation Request                                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Verifier Identity                                           │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ client_id: "https://verifier.example.com"              │ │
│  │ client_metadata: { name, logo, ... }                   │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Credential Requirements                                     │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ presentation_definition: {                             │ │
│  │   input_descriptors: [                                 │ │
│  │     { id, constraints, ... }                           │ │
│  │   ]                                                    │ │
│  │ }                                                      │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Response Instructions                                       │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ response_type: "vp_token"                              │ │
│  │ response_mode: "direct_post"                           │ │
│  │ response_uri: "https://verifier.example.com/callback"  │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  Security Binding                                            │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ nonce: "n-0S6_WzA2Mj"                                  │ │
│  │ state: "af0ifjsldkj"                                   │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Delivery Mechanisms

### QR Code

The most common mechanism for cross-device verification:

```
                    ┌─────────────────┐
                    │  █████████████  │
                    │  █           █  │
                    │  █  QR Code  █  │
                    │  █           █  │
                    │  █████████████  │
                    └─────────────────┘
                            │
                            │ User scans
                            ▼
                    ┌─────────────────┐
                    │  Wallet App     │
                    │  ┌───────────┐  │
                    │  │ Request   │  │
                    │  │ Received  │  │
                    │  └───────────┘  │
                    └─────────────────┘
```

**Characteristics:**
- Works across devices (verifier on desktop, wallet on mobile)
- No network connection needed to receive request
- Limited by QR code size (~2KB practical limit)
- Can use `request_uri` for larger requests

### Deep Link

For same-device flows where verifier and wallet are on the same device:

```
User clicks link:
openid4vp://authorize?client_id=...&request_uri=...
                │
                │ OS routes to wallet
                ▼
┌─────────────────────────────────────────────┐
│  Wallet receives intent/URL                  │
│  Parses OID4VP parameters                    │
│  Fetches full request if request_uri present │
└─────────────────────────────────────────────┘
```

**Characteristics:**
- Seamless UX on mobile
- Requires wallet to be registered for URI scheme
- Same device means shared network context
- Can handle larger requests via `request_uri`

### NFC (Proximity)

For in-person verification scenarios:

```
┌───────────────┐         ┌───────────────┐
│   Terminal    │   NFC   │    Mobile     │
│   ┌───────┐   │◄───────►│   ┌───────┐   │
│   │Reader │   │   Tap   │   │Wallet │   │
│   └───────┘   │         │   └───────┘   │
└───────────────┘         └───────────────┘
```

**Characteristics:**
- Physical presence required
- Very small payload (NDEF URI record)
- Often combined with BLE for response
- ISO 18013-5 device engagement

### OAuth Redirect

For web applications following standard OAuth patterns:

```
Browser                    Verifier                    Wallet
   │                          │                           │
   │  1. User action          │                           │
   │────────────────────────► │                           │
   │                          │                           │
   │  2. Redirect             │                           │
   │◄──────────────────────── │                           │
   │  Location: openid4vp://  │                           │
   │                          │                           │
   │  3. Open wallet          │                           │
   │─────────────────────────────────────────────────────►│
   │                          │                           │
```

---

## Request URI vs Inline Request

### Inline Request

All parameters encoded in the URI:

```
openid4vp://authorize?
  response_type=vp_token&
  client_id=https://verifier.example.com&
  presentation_definition=...&
  nonce=n-0S6_WzA2Mj
```

**Advantages:**
- Self-contained
- No additional network request
- Works offline (for initial request)

**Limitations:**
- URI size limits
- Complex requests may not fit

### Request URI (By Reference)

URI points to the full request:

```
openid4vp://authorize?
  client_id=https://verifier.example.com&
  request_uri=https://verifier.example.com/requests/abc123
```

The wallet fetches the full request:

```http
GET /requests/abc123 HTTP/1.1
Host: verifier.example.com

HTTP/1.1 200 OK
Content-Type: application/oauth-authz-req+jwt

eyJhbGciOiJFUzI1NiIsInR5cCI6Im9hdXRoLWF1dGh6LXJlcStqd3QiLCJraWQiOiIuLi4ifQ...
```

**Advantages:**
- Unlimited request size
- Request can be signed (JWT)
- Single-use/expiring requests possible
- Request integrity protected

**Limitations:**
- Requires network connectivity
- Additional round trip

---

## Request Signing

For high-assurance scenarios, the request can be signed as a JWT:

```
Header:
{
  "alg": "ES256",
  "typ": "oauth-authz-req+jwt",
  "kid": "verifier-key-1"
}

Payload:
{
  "client_id": "https://verifier.example.com",
  "response_type": "vp_token",
  "presentation_definition": { ... },
  "nonce": "n-0S6_WzA2Mj",
  "iat": 1683000000,
  "exp": 1683000300
}

Signature:
[ES256 signature over header.payload]
```

**Benefits of signing:**
- Request integrity verification
- Verifier authentication
- Replay window limitation (via `iat`/`exp`)
- Certificate chain validation possible

---

## Client ID Schemes

The `client_id` parameter identifies the Verifier. OID4VP supports multiple schemes:

| Scheme | Format | Trust Basis |
|--------|--------|-------------|
| `redirect_uri` | The redirect URI itself | Implicit (matches response destination) |
| `x509_san_dns` | DNS name from X.509 cert | PKI certificate chain |
| `x509_san_hash` | Hash of X.509 certificate | Pre-shared certificate |
| `did` | Decentralized Identifier | DID Document resolution |
| `verifier_attestation` | JWT attestation | Attestation issuer trust |

### Example: x509_san_dns

```
client_id: x509_san_dns:verifier.example.com
```

The wallet:
1. Extracts the request's signing certificate
2. Validates the certificate chain
3. Checks that SAN includes `verifier.example.com`
4. Displays verified identity to user

---

## Nonce and Replay Prevention

The `nonce` parameter is critical for security:

```
Verifier generates:
  nonce = random(256 bits)
  store(session_id → nonce)

Wallet includes in VP:
  proof.challenge = nonce

Verifier validates:
  assert VP.proof.challenge == stored_nonce
  delete(stored_nonce)  // prevent replay
```

**Properties:**
- Must be cryptographically random
- Must be unpredictable
- Must be single-use
- Binds presentation to specific request

---

## State Parameter

The `state` parameter links request and response:

```
Request:
  state = "af0ifjsldkj"

Response:
  state = "af0ifjsldkj"  // echoed back

Verifier:
  lookup(state → session context)
```

**Purpose:**
- Session correlation
- CSRF protection
- Maintain context across redirect

---

## What Happens After Request Delivery

1. **Wallet receives request**
2. **Validates request structure**
3. **Resolves client metadata** (if not inline)
4. **Validates client identity** (if signed)
5. **Parses presentation definition**
6. **Matches against stored credentials**
7. **Displays request to user**
8. **Awaits user consent**

The subsequent steps are covered in:
- [05 Presentation Definition](../05-presentation-definition/) — Credential requirements
- [06 Credential Selection](../06-credential-selection/) — Matching and consent
- [08 Presentation Response](../08-presentation-response/) — VP construction and delivery
