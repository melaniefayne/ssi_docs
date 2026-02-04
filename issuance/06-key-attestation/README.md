# 06 -- Key Attestation

This section covers key attestation and wallet attestation -- the mechanisms by which a wallet proves to an issuer that its cryptographic keys are stored in secure hardware and that the wallet application itself is genuine. Key attestation is a critical trust signal in credential issuance: without it, an issuer has no way to verify that the holder's private key cannot be extracted, copied, or used outside the authorized wallet.

---

## Documents

| # | Document | Scope |
|---|----------|-------|
| 1 | [Conceptual Overview](./conceptual-overview.md) | Why key attestation matters, the distinction between wallet attestation and key attestation, the trust chain from issuer to wallet provider to wallet to keys, and the WSCD (Wallet Secure Cryptographic Device) concept. |
| 2 | [Protocol & Standards](./protocol-and-standards.md) | JWK key representation (RFC 7517), key attestation in OID4VCI proofs, wallet instance attestation, Android Keystore attestation, iOS Secure Enclave and App Attest, and remote secure elements. |
| 3 | [Implementation Details](./implementation-details.md) | How each wallet implementation (EUDI Android, EUDI iOS, Procivis ONE, Affinidi) implements key attestation -- attestation endpoints, JWK serialization, hardware security levels, and remote secure element integration. |
| 4 | [Comparative Analysis](./comparative-analysis.md) | Side-by-side comparison of attestation mechanisms, security tiers, hardware requirements, trust chain models, and trade-offs across all implementations. |

## Key Themes

- **Hardware trust anchors** -- Android Keystore (TEE/StrongBox) and iOS Secure Enclave provide hardware-backed key storage. The attestation mechanisms differ significantly between platforms, but the goal is the same: prove that a key was generated inside secure hardware and cannot be extracted.
- **Wallet vs. key attestation** -- Wallet attestation proves the app is genuine; key attestation proves specific keys are hardware-backed. Both are needed for full trust, and the EUDI implementations make this distinction explicit with separate attestation endpoints.
- **Security tiers** -- Procivis ONE defines three distinct key storage tiers (internal, secure element, remote secure element), giving issuers flexibility in what security level they require. This contrasts with EUDI's binary hardware-backed model and Affinidi's configuration-driven approach.
- **Remote secure elements** -- Procivis ONE's integration with Ubiqu RSE introduces a third model beyond local hardware: keys stored in a remote HSM, accessed via PIN/biometric-protected SDK calls. This enables hardware-grade security on devices without local secure elements.

---

## Prerequisites

Before reading this section, you should be familiar with:

- The authorization and token exchange flow, including `c_nonce` issuance (see [04 -- Authorization & Token](../04-authorization-and-token/))
- Nonce lifecycle and how nonces bind proofs to sessions (see [05 -- Nonce & Replay Protection](../05-nonce-and-replay-protection/))

After this section, continue to [07 -- Proof Construction](../07-proof-construction/) to understand how key material and nonces are combined into the proof-of-possession submitted with the credential request.
