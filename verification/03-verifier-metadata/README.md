# Section 03: Verifier Metadata

This section covers how wallets discover and validate verifier information during the presentation flow. Verifier metadata establishes trust and provides the holder with information about who is requesting their credentials.

---

## Contents

| File | Topic |
|------|-------|
| [conceptual-overview.md](./conceptual-overview.md) | What verifier metadata is and why it matters |
| [protocol-and-standards.md](./protocol-and-standards.md) | OID4VP client metadata, X.509 certificates, trust frameworks |
| [implementation-details.md](./implementation-details.md) | How EUDI, Procivis, and Affinidi handle verifier metadata |
| [comparative-analysis.md](./comparative-analysis.md) | Comparison of verifier trust approaches |

---

## Key Concepts

- **Client Metadata**: Information about the verifier (name, logo, policies)
- **Trust Establishment**: How the wallet determines if the verifier is legitimate
- **Reader Authentication**: X.509 certificates for verifier identity verification
- **Client ID Schemes**: Different methods for verifier identification

---

## Relationship to Other Sections

- **[02 Presentation Request](../02-presentation-request/)**: Verifier metadata is often delivered with or referenced in the request
- **[04 Authorization Request](../04-authorization-request/)**: Client ID schemes affect how metadata is validated
- **[09 Verifier Validation](../09-verifier-validation/)**: Trust decisions made here affect what validation occurs later
