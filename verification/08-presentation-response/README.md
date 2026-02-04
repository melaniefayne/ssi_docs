# Presentation Response

The Presentation Response is the final step of the verification flow where the Holder transmits the Verifiable Presentation (VP Token) to the Verifier. This section covers response construction, transmission modes, and delivery mechanisms.

## Contents

1. [Conceptual Overview](./conceptual-overview.md) — VP Token structure and construction
2. [Protocol & Standards](./protocol-and-standards.md) — OID4VP response specification
3. [Comparative Analysis](./comparative-analysis.md) — Implementation comparison

---

## Key Concepts

The presentation response consists of:

1. **VP Token** — The Verifiable Presentation containing credentials and holder proof
2. **Presentation Submission** — Mapping between requested and provided credentials
3. **State** — Echoed from request for session correlation

---

## Response Modes

| Mode | Mechanism | Use Case |
|------|-----------|----------|
| `direct_post` | HTTP POST to verifier | Cross-device, production |
| `direct_post.jwt` | Encrypted POST | Confidential responses |
| `fragment` | URL fragment redirect | Same-device browser flows |

---

## VP Token Structure

The VP Token contains the actual presentation:

```
┌─────────────────────────────────────────────────────────┐
│  VP Token                                                │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  ┌───────────────────────────────────────────────────┐  │
│  │  Verifiable Credential(s)                          │  │
│  │  - Issuer-signed claims                            │  │
│  │  - Selective disclosure (if applicable)            │  │
│  └───────────────────────────────────────────────────┘  │
│                                                          │
│  ┌───────────────────────────────────────────────────┐  │
│  │  Holder Binding Proof                              │  │
│  │  - Nonce from verifier                             │  │
│  │  - Audience (verifier ID)                          │  │
│  │  - Holder signature                                │  │
│  └───────────────────────────────────────────────────┘  │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

---

## Response Flow

```
Wallet                              Verifier
   │                                   │
   │  1. Construct VP Token            │
   │     - Include selected credentials│
   │     - Apply selective disclosure  │
   │     - Create holder binding proof │
   │                                   │
   │  2. Create Presentation Submission│
   │     - Map descriptors to credentials│
   │                                   │
   │  3. Transmit Response             │
   │  ─────────────────────────────────►
   │     POST /callback                │
   │     vp_token=...                  │
   │     presentation_submission=...   │
   │     state=...                     │
   │                                   │
   │                                   │  4. Validate response
   │                                   │  5. Extract claims
   │                                   │
```

---

## Format-Specific Responses

| Format | VP Token Structure |
|--------|-------------------|
| **SD-JWT-VC** | Issuer JWT + disclosures + kb-jwt |
| **MSO-MDOC** | CBOR DeviceResponse |
| **W3C VC** | JSON VP with proof |

---

## Implementation Sections

- [Conceptual Overview](./conceptual-overview.md) — Response construction details
- [Protocol & Standards](./protocol-and-standards.md) — OID4VP response format
- [Comparative Analysis](./comparative-analysis.md) — How implementations send responses
