# Section 06: Credential Selection

This section covers how wallets match stored credentials against verifier requirements and enable users to choose which credentials and claims to share.

---

## Contents

| File | Topic |
|------|-------|
| [conceptual-overview.md](./conceptual-overview.md) | Credential matching, user consent, selective disclosure UI |
| [protocol-and-standards.md](./protocol-and-standards.md) | Constraint matching algorithms, path expressions |
| [implementation-details.md](./implementation-details.md) | How EUDI, Procivis, and Affinidi implement selection |
| [comparative-analysis.md](./comparative-analysis.md) | Comparison of selection approaches |

---

## Key Concepts

- **Credential Matching**: Finding credentials that satisfy verifier requirements
- **Constraint Evaluation**: Checking if credential claims meet specified constraints
- **User Selection**: Enabling holder choice when multiple credentials qualify
- **Claim Selection**: Choosing which specific claims to disclose
- **Selective Disclosure**: Revealing only required claims, not entire credential

---

## Relationship to Other Sections

- **[05 Presentation Definition](../05-presentation-definition/)**: Defines the requirements credentials must satisfy
- **[07 Holder Binding](../07-holder-binding/)**: Key binding proofs after selection
- **[08 Presentation Response](../08-presentation-response/)**: Selected credentials are assembled into VP
- **[10 Secure Storage](../../docs/10-secure-storage/)**: Credentials are retrieved from secure storage
