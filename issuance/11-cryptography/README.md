# Section 11: Cryptography

This is the comprehensive cryptography reference for the SSI documentation. It covers every algorithm, credential format, protocol-level cryptographic mechanism, and security analysis relevant to the EUDI, Procivis, and Affinidi implementations.

## Contents

| File | Topic |
|------|-------|
| [algorithms.md](./algorithms.md) | Every cryptographic algorithm used across implementations: ES256, EdDSA, secp256k1, CRYSTALS-DILITHIUM, BBS+, SHA-256 |
| [credential-formats.md](./credential-formats.md) | Credential format deep-dive: mDoc (ISO 18013-5), SD-JWT, W3C VC with JSON-LD, JWT VC |
| [protocols.md](./protocols.md) | Protocol-level cryptography: OID4VCI token exchange, PAR, DPoP, PKCE, JWK, JWS, COSE |
| [trust-and-attack-prevention.md](./trust-and-attack-prevention.md) | Security analysis: trust establishment, replay attacks, token theft, substitution, impersonation, MITM |

## How to Read This Section

This section is organized from primitives to applications:

1. **Algorithms** -- the lowest-level cryptographic operations (signing, hashing)
2. **Credential Formats** -- how algorithms are applied to structure verifiable credentials
3. **Protocols** -- how credentials are exchanged securely over the wire
4. **Trust and Attack Prevention** -- how the complete system resists real-world threats

Each layer builds on the previous. The algorithms section explains what ES256 does; the credential formats section explains how ES256 is used to sign an SD-JWT; the protocols section explains how the signed SD-JWT is transmitted via OID4VCI with DPoP protection; and the security analysis section explains how this combination prevents specific attacks.

## Key Principles

- **Defense in depth**: No single cryptographic mechanism provides complete security. The system layers multiple protections.
- **Proof of possession**: Every critical exchange requires the holder to prove they control a private key, not just that they possess a credential.
- **Binding**: Credentials are bound to keys, tokens are bound to senders, and requests are bound to sessions. Unbinding any of these creates an attack surface.
- **Minimal disclosure**: Credential formats are designed to reveal only the claims the verifier needs, not the entire credential.
