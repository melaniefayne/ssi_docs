# Section 11: Cryptography for Verification

This section covers the cryptographic mechanisms specific to the verification/presentation flow. For foundational cryptography (algorithms, credential formats), see the [Issuance Cryptography section](../../issuance/11-cryptography/).

---

## Contents

| File | Topic |
|------|-------|
| [verification-operations.md](./verification-operations.md) | Cryptographic operations during verification |
| [selective-disclosure-crypto.md](./selective-disclosure-crypto.md) | Cryptography of selective disclosure |
| [key-binding-proofs.md](./key-binding-proofs.md) | Holder key binding and proof mechanisms |
| [session-security.md](./session-security.md) | Session encryption, nonce handling, replay prevention |

---

## Key Concepts

- **Holder Binding Proof**: Cryptographic proof that holder controls credential
- **Selective Disclosure**: Revealing only specific claims
- **Session Key Derivation**: Secure communication for proximity
- **Nonce Binding**: Preventing replay attacks

---

## Verification vs Issuance Cryptography

| Aspect | Issuance | Verification |
|--------|----------|--------------|
| **Who signs** | Issuer signs VC | Holder signs VP |
| **Key binding** | Issuer binds holder key | Holder proves key possession |
| **Disclosure** | Full credential | Selected claims only |
| **Challenge** | c_nonce from issuer | nonce from verifier |

---

## Relationship to Other Sections

- **[../../issuance/11-cryptography/](../../issuance/11-cryptography/)**: Foundational algorithms and formats
- **[07 Holder Binding](../07-holder-binding/)**: Proof-of-possession concepts
- **[09 Verifier Validation](../09-verifier-validation/)**: How proofs are verified
