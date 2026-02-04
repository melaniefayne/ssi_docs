# Presentation Definition

The Presentation Definition specifies what credentials and claims the Verifier requires from the Holder. It is the query language that bridges the Verifier's requirements with the Holder's stored credentials.

## Contents

1. [Conceptual Overview](./conceptual-overview.md) — What presentation definitions are and how they work
2. [Protocol & Standards](./protocol-and-standards.md) — DIF PEX and DCQL specifications
3. [Comparative Analysis](./comparative-analysis.md) — How each implementation handles definitions

---

## Key Concepts

A Presentation Definition answers the question: "What credentials does this Verifier need?"

It specifies:
- **Credential types** — What kinds of credentials are acceptable
- **Required claims** — Which specific claims must be present
- **Optional claims** — Which claims may be included if available
- **Constraints** — Filters on claim values
- **Format requirements** — Which credential formats are acceptable

---

## Query Languages

Two primary query languages are used across implementations:

### DIF Presentation Exchange (PEX)

The original query language from the Decentralized Identity Foundation:

```json
{
  "id": "example-request",
  "input_descriptors": [
    {
      "id": "identity-credential",
      "constraints": {
        "fields": [
          {
            "path": ["$.credentialSubject.given_name"],
            "filter": { "type": "string" }
          }
        ]
      }
    }
  ]
}
```

**Supported by:** All implementations (Procivis supports v1 and v2)

### Digital Credentials Query Language (DCQL)

A newer, simpler query format adopted in OID4VP v1.0:

```json
{
  "credentials": [
    {
      "id": "pid",
      "format": "dc+sd-jwt",
      "claims": [
        { "path": ["given_name"] },
        { "path": ["family_name"] }
      ]
    }
  ]
}
```

**Supported by:** EUDI (via OID4VP v1.0)

---

## The Matching Problem

The Presentation Definition creates a matching problem:

```
Verifier's Requirements     ←→     Holder's Credentials
        │                                    │
        ▼                                    ▼
┌─────────────────┐              ┌─────────────────┐
│ Input Descriptor│              │ Credential #1   │
│ - type: mDL     │   Match?     │ - type: mDL     │
│ - claims:       │◄────────────►│ - claims:       │
│   - given_name  │              │   - given_name  │
│   - family_name │              │   - family_name │
│   - birthdate   │              │   - birthdate   │
└─────────────────┘              │   - address     │
                                 └─────────────────┘
                                          │
                                 ┌─────────────────┐
                                 │ Credential #2   │
                                 │ - type: Diploma │
                                 │ - claims:       │
                                 │   - degree      │
                                 │   - university  │
                                 └─────────────────┘
```

The wallet must:
1. Find credentials matching each input descriptor
2. Verify all required fields are present
3. Apply any value constraints
4. Determine which credentials to present

---

## Selective Disclosure

Presentation Definitions enable selective disclosure:

```json
{
  "constraints": {
    "limit_disclosure": "required",
    "fields": [
      { "path": ["$.credentialSubject.given_name"] },
      { "path": ["$.credentialSubject.family_name"] }
    ]
  }
}
```

When `limit_disclosure` is `required`:
- Only specified fields may be revealed
- Wallet must support selective disclosure
- Other credential fields remain hidden

---

## Implementation Sections

- [Conceptual Overview](./conceptual-overview.md) — Query structure and semantics
- [Protocol & Standards](./protocol-and-standards.md) — PEX and DCQL specifications
- [Comparative Analysis](./comparative-analysis.md) — Implementation differences
