# Section 09: Verifier Validation

This section covers the verifier-side validation of Verifiable Presentations — the checks performed after receiving a VP Token to determine if the presentation is valid and trustworthy.

---

## Contents

| File | Topic |
|------|-------|
| [conceptual-overview.md](./conceptual-overview.md) | What verifiers check and why |
| [protocol-and-standards.md](./protocol-and-standards.md) | Validation algorithms and requirements |
| [implementation-details.md](./implementation-details.md) | How verification is implemented |
| [comparative-analysis.md](./comparative-analysis.md) | Comparison of validation approaches |

---

## Key Concepts

- **Signature Verification**: Cryptographic proof validation
- **DID Resolution**: Resolving holder/issuer identifiers to keys
- **Schema Validation**: Checking credential structure
- **Revocation Status**: Checking if credentials are still valid
- **Trust Framework**: Verifying issuer is trusted

---

## Validation Checklist

| Check | What It Validates |
|-------|-------------------|
| VP Signature | Holder controls the credential |
| VC Signature | Issuer signed the credential |
| Nonce | Presentation is fresh, not replayed |
| Issuer Trust | Issuer is in trust framework |
| Revocation | Credential not revoked |
| Expiration | Credential not expired |
| Schema | Credential structure is valid |
| Constraints | Claims meet presentation definition |

---

## Relationship to Other Sections

- **[08 Presentation Response](../08-presentation-response/)**: What the verifier receives
- **[03 Verifier Metadata](../03-verifier-metadata/)**: Trust configuration for validation
- **[11 Cryptography](../11-cryptography/)**: Cryptographic primitives used
