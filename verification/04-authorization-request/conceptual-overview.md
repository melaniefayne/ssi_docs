# Authorization Request — Conceptual Overview

The **Authorization Request** is the formal protocol message from verifier to wallet that initiates a credential presentation. Built on OAuth 2.0 authorization requests, OID4VP extends them with credential-specific parameters.

---

## Request Purpose

The authorization request accomplishes several things simultaneously:

```
┌─────────────────────────────────────────────────────────────────┐
│                    Authorization Request                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. IDENTIFICATION      Who is asking?                           │
│     └── client_id, client_metadata                               │
│                                                                  │
│  2. SPECIFICATION       What do they want?                       │
│     └── presentation_definition (or dcql_query)                  │
│                                                                  │
│  3. INSTRUCTION         How should I respond?                    │
│     └── response_type, response_mode, response_uri               │
│                                                                  │
│  4. SECURITY            How is this protected?                   │
│     └── nonce, state, signed JWT                                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Request Lifecycle

```
    Verifier                              Wallet                              User
       │                                    │                                   │
       │  1. Create request                 │                                   │
       │     - Generate nonce               │                                   │
       │     - Build presentation_definition│                                   │
       │     - Sign request (optional)      │                                   │
       │                                    │                                   │
       │  2. Deliver request                │                                   │
       │ ──────────────────────────────────►│                                   │
       │  (QR, deep link, redirect)         │                                   │
       │                                    │                                   │
       │                                    │  3. Parse request                 │
       │                                    │     - Validate structure          │
       │                                    │     - Validate signature          │
       │                                    │     - Extract requirements        │
       │                                    │                                   │
       │                                    │  4. Display to user              │
       │                                    │ ─────────────────────────────────►│
       │                                    │                                   │
```

---

## Key Parameters

### Response Type (`response_type`)

Specifies what token(s) the wallet should return:

| Value | Description |
|-------|-------------|
| `vp_token` | Verifiable Presentation token only |
| `vp_token id_token` | VP token plus SIOP v2 ID token |

### Response Mode (`response_mode`)

Specifies how the response should be delivered:

| Mode | Description | Use Case |
|------|-------------|----------|
| `fragment` | Response in URL fragment | Same-device, browser-based |
| `direct_post` | HTTP POST to response_uri | Cross-device, server-to-server |
| `direct_post.jwt` | Encrypted/signed POST | High security |

### Response URI (`response_uri`)

Where the VP token should be sent:

```
response_uri: https://verifier.example.com/callback
```

The wallet will POST the response here (for `direct_post` mode).

### Nonce

A unique, single-use challenge that binds the presentation to this specific request:

```
nonce: "n-0S6_WzA2Mj"
```

The nonce appears in the VP token's proof, preventing replay attacks.

### State

Client-side session correlation:

```
state: "af0ifjsldkj"
```

Echoed back in the response to correlate with session context.

---

## Request Formats

### Simple (Query Parameters)

All parameters in the URL:

```
openid4vp://authorize?
  response_type=vp_token&
  client_id=https://verifier.example.com&
  response_uri=https://verifier.example.com/callback&
  response_mode=direct_post&
  presentation_definition=...&
  nonce=n-0S6_WzA2Mj&
  state=af0ifjsldkj
```

### By Reference (Request URI)

Full request fetched from server:

```
openid4vp://authorize?
  client_id=https://verifier.example.com&
  request_uri=https://verifier.example.com/requests/abc123
```

### Signed Request (JAR)

Request as a signed JWT (JWT-Secured Authorization Request):

```
openid4vp://authorize?
  client_id=https://verifier.example.com&
  request=eyJhbGciOiJFUzI1NiIsInR5cCI6Im9hdXRoLWF1dGh6LXJlcStqd3QifQ...
```

Or combined with request_uri:

```
openid4vp://authorize?
  client_id=https://verifier.example.com&
  request_uri=https://verifier.example.com/requests/abc123
```

Where the URI returns a signed JWT.

---

## Signed Request Structure

When the request is a JWT:

```
Header:
{
  "alg": "ES256",
  "typ": "oauth-authz-req+jwt",
  "kid": "verifier-key-1"
}

Payload:
{
  "iss": "https://verifier.example.com",
  "aud": "https://self-issued.me/v2",
  "response_type": "vp_token",
  "client_id": "https://verifier.example.com",
  "client_id_scheme": "x509_san_dns",
  "response_uri": "https://verifier.example.com/callback",
  "response_mode": "direct_post",
  "nonce": "n-0S6_WzA2Mj",
  "state": "af0ifjsldkj",
  "presentation_definition": { ... },
  "iat": 1683000000,
  "exp": 1683000300
}

Signature:
[ES256 signature]
```

### Why Sign Requests?

| Benefit | Description |
|---------|-------------|
| **Integrity** | Request cannot be tampered with |
| **Authentication** | Verifier identity is proven |
| **Non-repudiation** | Verifier cannot deny making request |
| **Expiration** | Time-limited validity (`iat`/`exp`) |

---

## Validation Steps

When wallet receives a request:

```
1. Parse URI/QR code
   └── Extract scheme (openid4vp://) and parameters

2. If request_uri present:
   └── Fetch full request from URI
   └── Validate content-type

3. If request is JWT:
   └── Decode JWT
   └── Validate signature
   └── Check iat/exp timing
   └── Verify client_id matches JWT claims

4. Validate required parameters:
   └── response_type present
   └── nonce present (required for VP)
   └── presentation_definition or dcql_query present

5. Validate client identity (based on client_id_scheme):
   └── x509_san_dns → validate cert chain
   └── did → resolve DID document
   └── redirect_uri → verify matches response_uri

6. Parse presentation_definition
   └── Extract input_descriptors
   └── Identify required credentials
```

---

## Error Handling

If validation fails, the wallet may:

| Situation | Wallet Behavior |
|-----------|-----------------|
| Invalid structure | Display error, abort |
| Untrusted verifier | Display warning, allow proceed or abort |
| Expired request | Display error, abort |
| Invalid signature | Display error, abort |
| No matching credentials | Display error, allow partial or abort |

---

## Same-Device vs Cross-Device

### Same-Device Flow

Verifier and wallet on same device:

```
Browser ──deep link──► Wallet ──POST──► Verifier Backend
                                             │
                              ◄──redirect────┘
```

### Cross-Device Flow

Verifier on desktop, wallet on mobile:

```
Desktop Browser                Mobile Wallet              Verifier Backend
      │                             │                           │
      │ shows QR                    │                           │
      │◄────────────────────────────│ scans                     │
      │                             │                           │
      │                             │────POST vp_token─────────►│
      │                             │                           │
      │◄────────────────────────────────────────────────────────│
      │        polling or websocket notification                │
```

The `response_mode` parameter drives which flow is used.

---

## Relationship to Credential Requirements

The authorization request contains or references the **presentation definition** which specifies exactly what credentials and claims are needed. This is covered in detail in [Section 05](../05-presentation-definition/).

The request wrapper (this section) handles:
- Delivery and security
- Response routing
- Session management

The presentation definition (Section 05) handles:
- What credentials are required
- What claims within those credentials
- Matching constraints
