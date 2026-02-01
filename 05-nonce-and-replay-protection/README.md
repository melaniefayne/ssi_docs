# 05 -- Nonce & Replay Protection

This section covers the nonce mechanisms and replay prevention strategies used in OID4VCI credential issuance. Nonces are short-lived, single-use values that bind protocol messages to specific sessions, preventing an attacker from capturing and replaying a valid credential request to obtain duplicate credentials. Transaction codes (TxCodes) provide an additional layer of user verification that is complementary to cryptographic nonces.

---

## Documents

| # | Document | Scope |
|---|----------|-------|
| 1 | [Conceptual Overview](./conceptual-overview.md) | What `c_nonce` is, how it prevents replay attacks, the nonce lifecycle across token and credential responses, and how TxCodes provide additional user verification. |
| 2 | [Protocol & Standards](./protocol-and-standards.md) | Technical details of `c_nonce` in the OID4VCI specification, nonce rotation, JWT proof nonce claims, replay attack mechanics, and TxCode specification parameters. |
| 3 | [Implementation Details](./implementation-details.md) | How each wallet implementation (EUDI Android, EUDI iOS, Procivis ONE, Affinidi) handles nonce management internally and validates TxCode input from users. Code paths, validation logic, and error handling. |
| 4 | [Comparative Analysis](./comparative-analysis.md) | Side-by-side comparison of TxCode input modes, validation strictness, nonce management transparency, and error handling across all implementations. |

## Key Themes

- **Session binding** -- The `c_nonce` ties every proof-of-possession to a specific issuance session. Without it, an attacker could reuse a captured proof to request credentials on behalf of the legitimate holder.
- **Nonce lifecycle** -- Nonces are issued in the token response, consumed in the credential request proof, and may be rotated in the credential response. Implementations must track this lifecycle correctly.
- **TxCode as human verification** -- Transaction codes serve a different purpose than cryptographic nonces. They confirm that the person interacting with the wallet is the intended recipient of the credential offer, providing an out-of-band binding that cryptographic mechanisms alone cannot achieve.
- **Validation strictness** -- Implementations vary significantly in how strictly they validate TxCode input (numeric-only vs. text, exact length enforcement, debounce behavior), which affects both security and user experience.

---

## Prerequisites

Before reading this section, you should be familiar with:

- The authorization and token exchange flow, including the token response structure (see [04 -- Authorization & Token](../04-authorization-and-token/))
- The credential offer structure, including TxCode parameters in the grant object (see [02 -- Credential Offer](../02-credential-offer/))

After this section, continue to [06 -- Key Attestation](../06-key-attestation/) to understand how wallet and key attestation establish trust in the holder's cryptographic material.
