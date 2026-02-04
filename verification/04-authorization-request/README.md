# Section 04: Authorization Request

This section covers the OID4VP Authorization Request — the protocol message that initiates a presentation flow. The authorization request specifies what the verifier needs, how they should receive it, and provides security bindings.

---

## Contents

| File | Topic |
|------|-------|
| [conceptual-overview.md](./conceptual-overview.md) | What an authorization request is and its components |
| [protocol-and-standards.md](./protocol-and-standards.md) | OID4VP request structure, parameters, and signed requests |
| [implementation-details.md](./implementation-details.md) | How EUDI, Procivis, and Affinidi parse and validate requests |
| [comparative-analysis.md](./comparative-analysis.md) | Comparison of request handling approaches |

---

## Key Concepts

- **Response Type**: What the wallet should return (`vp_token`)
- **Response Mode**: How the response should be delivered (`direct_post`, `fragment`)
- **Presentation Definition**: What credentials/claims are required (covered in depth in Section 05)
- **Nonce**: Security binding to prevent replay attacks
- **State**: Session correlation parameter

---

## Relationship to Other Sections

- **[02 Presentation Request](../02-presentation-request/)**: Transport and delivery of the authorization request
- **[03 Verifier Metadata](../03-verifier-metadata/)**: Client metadata often accompanies the request
- **[05 Presentation Definition](../05-presentation-definition/)**: The credential requirements within the request
- **[08 Presentation Response](../08-presentation-response/)**: The response structure depends on request parameters
