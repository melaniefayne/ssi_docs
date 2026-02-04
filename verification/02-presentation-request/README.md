# Presentation Request

The Presentation Request is the entry point for the OID4VP verification flow. It is the mechanism by which a Verifier initiates a request for credentials from a Holder's wallet.

## Contents

1. [Conceptual Overview](./conceptual-overview.md) — What presentation requests are and how they work
2. [Protocol & Standards](./protocol-and-standards.md) — OID4VP Authorization Request specification
3. [Comparative Analysis](./comparative-analysis.md) — How each implementation handles requests

---

## Key Concepts

A Presentation Request (also called Authorization Request in OID4VP) is the Verifier's way of asking a Holder to present specific credentials. The request contains:

- **Verifier identity** — Who is requesting the credentials
- **Credential requirements** — What credentials and claims are needed (Presentation Definition)
- **Response instructions** — Where and how to send the presentation
- **Security binding** — Nonce to prevent replay attacks

---

## Relationship to Issuance

| Issuance | Verification |
|----------|-------------|
| Credential Offer | Presentation Request |
| Issuer initiates | Verifier initiates |
| `openid-credential-offer://` | `openid4vp://` |
| Wallet receives credential | Wallet sends presentation |

---

## Delivery Mechanisms

Presentation requests reach the wallet through various channels:

| Mechanism | Description | Use Case |
|-----------|-------------|----------|
| **QR Code** | Request URI encoded as QR | Cross-device, in-person |
| **Deep Link** | Request URI as clickable link | Same-device, web/app |
| **NFC** | Request URI via NFC tap | Proximity, physical terminals |
| **Redirect** | OAuth-style redirect | Web applications |

---

## Request Flow Preview

```
Verifier                           Wallet
   │                                  │
   │  1. Construct request            │
   │     - presentation_definition    │
   │     - client_id                  │
   │     - nonce                      │
   │     - response_mode              │
   │                                  │
   │  2. Deliver request              │
   │  ─────────────────────────────►  │
   │     (QR / deep link / redirect)  │
   │                                  │
   │                                  │  3. Parse & validate
   │                                  │  4. Match credentials
   │                                  │  5. Get user consent
   │                                  │  6. Construct VP
   │                                  │
   │  7. Receive response             │
   │  ◄─────────────────────────────  │
   │     (vp_token)                   │
   │                                  │
```

---

## Implementation-Specific Docs

- [EUDI Implementation](./eudi-implementation.md)
- [Procivis Implementation](./procivis-implementation.md)
- [Affinidi Implementation](./affinidi-implementation.md)
