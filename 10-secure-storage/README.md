# Section 10: Secure Storage

This section covers how SSI wallets protect credential data and cryptographic keys at rest. Secure storage is a foundational requirement: credentials are bearer instruments, and any compromise of stored keys or credential data undermines the entire trust model.

## Contents

| File | Topic |
|------|-------|
| [conceptual-overview.md](./conceptual-overview.md) | Why secure storage matters: bearer tokens, hardware backing, encryption at rest, credential lifecycle and policies |
| [android-storage.md](./android-storage.md) | Android credential storage: Keystore, TEE, Strongbox, and implementation details for EUDI, Procivis, and Affinidi |
| [ios-storage.md](./ios-storage.md) | iOS credential storage: Secure Enclave, Keychain Services, and implementation details for EUDI, Procivis, and Affinidi |
| [comparative-analysis.md](./comparative-analysis.md) | Cross-platform storage comparison table, trust boundary analysis, and backup/restore considerations |

## Key Themes

- **Hardware-backed key storage** is the gold standard. Both Android (TEE/Strongbox) and iOS (Secure Enclave) provide hardware isolation for cryptographic keys.
- **Credential policies** govern how credentials are consumed: one-time use credentials are discarded after presentation, while rotate-on-use credentials trigger re-issuance.
- **Defense in depth** requires layering hardware isolation, encryption at rest, biometric gating, and access control policies.
- **Implementation divergence** is significant: EUDI uses platform-native storage directly, Procivis abstracts through a cross-platform SDK, and Affinidi delegates to cloud-based vault infrastructure.
