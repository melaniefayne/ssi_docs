# Holder Binding

Holder binding is the cryptographic mechanism that proves the presenter of a credential is its legitimate owner. Without holder binding, credentials could be freely copied and presented by anyone who obtains them.

## Contents

1. [Conceptual Overview](./conceptual-overview.md) — What holder binding is and why it matters
2. [Protocol & Standards](./protocol-and-standards.md) — Cryptographic mechanisms and formats
3. [Comparative Analysis](./comparative-analysis.md) — How each implementation handles binding

---

## The Core Problem

A Verifiable Credential is a digital document. Like any digital file, it can be copied. Without holder binding:

```
Legitimate Holder                    Attacker
┌─────────────────┐                 ┌─────────────────┐
│  Has credential │                 │  Copies credential │
│  VC signed by   │ ──── Copy ───► │  Presents to      │
│  Issuer         │                 │  Verifier         │
└─────────────────┘                 └─────────────────┘
        │                                   │
        ▼                                   ▼
   Legitimate                          Fraudulent
   Presentation                        Presentation
```

Holder binding prevents this by requiring proof of key possession.

---

## Types of Holder Binding

### 1. Cryptographic Holder Binding

The credential contains a public key. The holder must prove control of the corresponding private key:

```
At Issuance:
┌────────────────────────────────────┐
│  Credential                         │
│  ┌────────────────────────────────┐│
│  │ claims: { name: "Alice", ... } ││
│  │ holder_key: "did:key:z6Mk..."  ││◄── Holder's public key
│  │ issuer_signature: "..."        ││
│  └────────────────────────────────┘│
└────────────────────────────────────┘

At Presentation:
┌────────────────────────────────────┐
│  Verifiable Presentation           │
│  ┌────────────────────────────────┐│
│  │ credential: { ... }            ││
│  │ proof: {                       ││
│  │   challenge: "verifier-nonce"  ││◄── Proves key control
│  │   signature: "..."             ││    at presentation time
│  │ }                              ││
│  └────────────────────────────────┘│
└────────────────────────────────────┘
```

### 2. Biometric Holder Binding

The credential contains biometric data (e.g., photo). The verifier compares against the presenter:

```
┌────────────────────────────────────┐
│  Credential                         │
│  ┌────────────────────────────────┐│
│  │ claims: { ... }                ││
│  │ portrait: [base64 image]       ││◄── Biometric template
│  └────────────────────────────────┘│
└────────────────────────────────────┘
        │
        ▼
   Verifier compares
   portrait to presenter
```

### 3. Claims-Based Holder Binding

The credential contains identifying claims that can be verified through other means:

```
┌────────────────────────────────────┐
│  Credential                         │
│  ┌────────────────────────────────┐│
│  │ name: "Alice Smith"            ││
│  │ birthdate: "1990-01-15"        ││◄── Can be verified
│  └────────────────────────────────┘│    against other ID
└────────────────────────────────────┘
```

---

## Holder Binding in Different Formats

| Format | Binding Mechanism |
|--------|-------------------|
| **SD-JWT-VC** | Key Binding JWT (kb-jwt) |
| **MSO-MDOC** | Device Authentication |
| **W3C VC (JSON-LD)** | VP Proof |
| **W3C VC (JWT)** | VP JWT Signature |

---

## The Nonce Challenge

The Verifier's nonce prevents replay attacks:

```
1. Verifier → Wallet: "Present credential, prove with nonce=abc123"

2. Wallet signs: Sign(nonce="abc123" + audience="verifier") with holder_key

3. Wallet → Verifier: Credential + Proof(signature over nonce)

4. Verifier validates:
   - Signature is valid
   - Nonce matches the one sent
   - Audience is correct
   - Credential's holder_key matches proof's signing key
```

---

## Why This Matters

Without cryptographic holder binding:
- Credentials can be shared or sold
- Stolen credentials can be used by attackers
- Identity theft is trivial

With holder binding:
- Only the legitimate holder can present
- Credentials are non-transferable
- Hardware-backed keys provide strong assurance

---

## Implementation Sections

- [Conceptual Overview](./conceptual-overview.md) — Deep dive into binding mechanisms
- [Protocol & Standards](./protocol-and-standards.md) — Format-specific specifications
- [Comparative Analysis](./comparative-analysis.md) — Implementation comparison
