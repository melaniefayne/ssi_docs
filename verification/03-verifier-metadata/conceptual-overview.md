# Verifier Metadata — Conceptual Overview

Verifier metadata answers the holder's fundamental question: **"Who is asking for my credentials, and should I trust them?"**

---

## The Trust Problem

When a wallet receives a presentation request, the holder needs to make an informed decision:

```
┌─────────────────────────────────────────────────────────────────┐
│  Wallet displays:                                                │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  🏢  Example Bank                                          │ │
│  │      wants to verify your identity                         │ │
│  │                                                            │ │
│  │  Requested information:                                    │ │
│  │  • Full name                                               │ │
│  │  • Date of birth                                           │ │
│  │  • Address                                                 │ │
│  │                                                            │ │
│  │  ✓ Verified organization                                   │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  How does the wallet know this is really "Example Bank"?         │
│  How does it know the verifier can be trusted?                   │
└─────────────────────────────────────────────────────────────────┘
```

Verifier metadata and trust mechanisms answer these questions.

---

## What Verifier Metadata Contains

### Display Information

Human-readable information shown to the holder:

| Field | Purpose | Example |
|-------|---------|---------|
| `client_name` | Organization name | "Example Bank" |
| `logo_uri` | Organization logo | "https://example.com/logo.png" |
| `client_uri` | Website | "https://example.com" |
| `policy_uri` | Privacy policy | "https://example.com/privacy" |
| `tos_uri` | Terms of service | "https://example.com/terms" |

### Technical Information

Protocol requirements for the presentation:

| Field | Purpose | Example |
|-------|---------|---------|
| `vp_formats` | Supported VP formats | `{"mso_mdoc": {}, "vc+sd-jwt": {}}` |
| `response_types_supported` | Supported response types | `["vp_token"]` |
| `response_modes_supported` | Supported response modes | `["direct_post"]` |

---

## Trust Establishment Methods

### 1. X.509 Certificates

The verifier's identity is bound to a PKI certificate chain:

```
┌─────────────────────────────────────────────────────────────────┐
│  Certificate Chain                                               │
│                                                                  │
│  Root CA (trusted by wallet)                                     │
│      │                                                           │
│      ▼                                                           │
│  Intermediate CA                                                 │
│      │                                                           │
│      ▼                                                           │
│  Verifier Certificate                                            │
│      └── Subject: CN=Example Bank                                │
│      └── SAN: DNS:verify.example.com                             │
│      └── Public Key: [used to sign requests]                     │
└─────────────────────────────────────────────────────────────────┘
```

**How it works:**
1. Verifier signs the presentation request with their private key
2. Request includes or references the certificate chain
3. Wallet validates the chain against trusted roots
4. Wallet extracts identity from certificate subject/SAN
5. UI shows verified identity to holder

**Strengths:**
- Established PKI infrastructure
- Revocable certificates
- Familiar trust model

**Limitations:**
- Requires certificate management
- Cost of certificates
- Centralized trust (CAs)

### 2. DID-Based Identity

The verifier is identified by a Decentralized Identifier:

```
client_id: did:web:verify.example.com
                    │
                    ▼
Wallet resolves:   GET https://verify.example.com/.well-known/did.json
                    │
                    ▼
DID Document:      {
                     "id": "did:web:verify.example.com",
                     "verificationMethod": [{
                       "id": "#key-1",
                       "publicKeyJwk": { ... }
                     }]
                   }
```

**How it works:**
1. Verifier's `client_id` is a DID
2. Wallet resolves DID to DID Document
3. DID Document contains public keys
4. Wallet validates request signature against DID key
5. Trust may come from registry lookup or manual verification

### 3. Verifier Attestation

A trusted third party vouches for the verifier:

```
┌─────────────────────────────────────────────────────────────────┐
│  Attestation Flow                                                │
│                                                                  │
│  Trust Registry ────attestation────► Verifier                    │
│       │                                   │                      │
│       │ trusts                            │ presents             │
│       ▼                                   ▼                      │
│     Wallet ◄─────────validates──────── Request                   │
└─────────────────────────────────────────────────────────────────┘
```

**How it works:**
1. Verifier obtains attestation JWT from trusted registry
2. Request includes `client_id_scheme: verifier_attestation`
3. Request includes attestation JWT
4. Wallet validates attestation issuer against trust list
5. Wallet extracts verifier identity from attestation

### 4. Redirect URI (Implicit)

The verifier is identified solely by the redirect URI:

```
client_id: https://verify.example.com/callback
response_uri: https://verify.example.com/callback
```

The assumption: only the legitimate verifier can receive responses at that URI.

**Strengths:**
- Simple, no certificates needed
- Works for low-assurance scenarios

**Limitations:**
- No cryptographic verification
- DNS/TLS as sole trust anchor
- Susceptible to DNS hijacking

---

## Metadata Discovery

### Inline Metadata

Metadata included directly in the request:

```json
{
  "client_id": "https://verifier.example.com",
  "client_metadata": {
    "client_name": "Example Bank",
    "logo_uri": "https://verifier.example.com/logo.png",
    "vp_formats": {
      "mso_mdoc": {},
      "vc+sd-jwt": {}
    }
  }
}
```

### Metadata by Reference

Metadata fetched from a URI:

```json
{
  "client_id": "https://verifier.example.com",
  "client_metadata_uri": "https://verifier.example.com/.well-known/openid-client"
}
```

The wallet fetches:

```http
GET /.well-known/openid-client HTTP/1.1
Host: verifier.example.com

HTTP/1.1 200 OK
Content-Type: application/json

{
  "client_name": "Example Bank",
  "logo_uri": "https://verifier.example.com/logo.png",
  ...
}
```

---

## Reader Authentication (ISO 18013-5)

For mDL/mDoc credentials, the standard defines Reader Authentication:

```
┌─────────────────────────────────────────────────────────────────┐
│  Reader Authentication Structure                                 │
│                                                                  │
│  SessionTranscript                                               │
│      │                                                           │
│      ▼                                                           │
│  ReaderAuth = COSE_Sign1(                                        │
│    payload: ReaderAuthentication = [                             │
│      "ReaderAuthentication",                                     │
│      SessionTranscript,                                          │
│      ItemsRequestBytes                                           │
│    ],                                                            │
│    signature: signed with reader certificate private key         │
│  )                                                               │
└─────────────────────────────────────────────────────────────────┘
```

The wallet validates:
1. Certificate chain validity
2. Certificate purpose (Reader Authentication)
3. Signature over session transcript
4. Certificate is not revoked

---

## Trust Indicators in UI

Wallets typically display trust status to users:

| Indicator | Meaning |
|-----------|---------|
| ✓ Verified | Certificate chain valid, issuer trusted |
| ⚠ Unknown | No certificate, or issuer not in trust list |
| ✗ Untrusted | Certificate invalid, expired, or revoked |

Example UI states:

```
Trusted:
┌────────────────────────────────┐
│ 🏢  Example Bank               │
│ ✓ Verified organization        │
└────────────────────────────────┘

Unknown:
┌────────────────────────────────┐
│ 🏢  verify.example.com         │
│ ⚠ Organization not verified   │
└────────────────────────────────┘
```

---

## Privacy Considerations

### Metadata Fetching

When wallet fetches metadata from verifier's server:
- Verifier learns wallet IP address
- Verifier learns timing of presentation consideration
- Consider: should metadata be cached? Pre-fetched?

### Display Information

Verifier-provided metadata could be misleading:
- Logo could impersonate another organization
- Name could be deceptive
- Trust indicators become critical

---

## Relationship to Issuance

The holder's wallet received credentials from **Issuers**. Now verifiers want those credentials. The trust models parallel each other:

| Aspect | Issuance | Verification |
|--------|----------|--------------|
| Trust anchor | Issuer certificates | Reader certificates |
| Discovery | Issuer metadata (`/.well-known/`) | Client metadata |
| Validation | Credential signature | Request signature |
| UI indicator | Issuer identity | Verifier identity |

Credentials bound to holder keys at issuance will have those bindings verified during presentation — the chain of trust continues.
