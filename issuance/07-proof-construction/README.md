# 07 -- Proof Construction

This section covers the construction of cryptographic proofs that accompany credential requests in the OID4VCI issuance flow. A proof demonstrates that the wallet holder controls the private key that will be bound to the issued credential. Without a valid proof, the issuer cannot cryptographically tie the credential to a specific holder, and the credential would be susceptible to theft and misuse.

---

## Documents

| # | Document | Scope |
|---|----------|-------|
| 1 | [Conceptual Overview](./conceptual-overview.md) | What proofs are, why proof of possession matters, how key binding works, and the role of proofs in preventing credential theft. |
| 2 | [Protocol & Standards](./protocol-and-standards.md) | JWT proof type specification (header, payload, signing), DPoP proof structure, and Key Binding JWT format in SD-JWT credentials. |
| 3 | [Implementation Details](./implementation-details.md) | How each wallet implementation (EUDI Android, EUDI iOS, Procivis ONE, Affinidi) constructs proofs -- SDK encapsulation, biometric-gated signing, PIN-gated signing, and key access control. |
| 4 | [Comparative Analysis](./comparative-analysis.md) | Side-by-side comparison of proof signing mechanisms, supported algorithms, SDK abstraction levels, and user authentication requirements during signing. |

## Key Themes

- **Proof of possession** -- The proof cryptographically demonstrates that the entity requesting a credential controls the private key that will be embedded in (or bound to) the credential. This is the foundational security property that prevents credential theft.
- **Key binding** -- The issued credential is cryptographically tied to the holder's key pair. Only the holder who possesses the corresponding private key can later present the credential in a verifiable manner.
- **User authentication at signing time** -- Implementations gate access to the signing key behind user authentication (biometrics, PIN, or device credential). This ensures that even if a device is compromised, proof construction requires active user participation.
- **SDK abstraction** -- The depth of SDK encapsulation varies significantly. Some implementations construct proofs entirely within the SDK, while others expose the signing step to the application layer for user authentication integration.

---

## Prerequisites

Before reading this section, you should be familiar with:

- The nonce mechanism and how `c_nonce` is issued in the token response (see [05 -- Nonce & Replay Protection](../05-nonce-and-replay-protection/))
- Key attestation and how wallet keys are established (see [06 -- Key Attestation](../06-key-attestation/))

After this section, continue to [08 -- Credential Request](../08-credential-request/) to understand how the constructed proof is included in the credential request sent to the issuer.
