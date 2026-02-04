# 04 -- Authorization & Token Exchange

This section covers the authorization and token exchange phase of the OID4VCI issuance flow. After the wallet has resolved a credential offer and discovered the issuer's metadata, it must obtain an access token before requesting credentials. The authorization phase establishes the wallet's right to receive the offered credentials and produces the cryptographic material (access token, nonce) needed for subsequent protocol steps.

---

## Documents

| # | Document | Scope |
|---|----------|-------|
| 1 | [Conceptual Overview](./conceptual-overview.md) | Why authorization is needed, the two grant types (Authorization Code and Pre-Authorized Code), and how the access token enables the credential request. |
| 2 | [Protocol & Standards](./protocol-and-standards.md) | OAuth 2.0 Authorization Code Flow with PKCE (RFC 7636), Pushed Authorization Requests (RFC 9126), DPoP sender-constrained tokens (RFC 9449), Pre-Authorized Code Grant mechanics, and the token endpoint response structure. |
| 3 | [Implementation Details](./implementation-details.md) | How each wallet implementation (EUDI Android, EUDI iOS, Procivis ONE, Affinidi) handles authorization -- SDK encapsulation, PAR configuration, DPoP usage, redirect URI handling, and browser-based auth flows. |
| 4 | [Comparative Analysis](./comparative-analysis.md) | Side-by-side comparison of PAR support, DPoP support, grant type preferences, browser-based authentication, and redirect URI strategies across all implementations. |

## Key Themes

- **Grant type selection** -- The choice between Authorization Code Grant and Pre-Authorized Code Grant reflects the trust model between issuer and holder. Pre-authorized flows assume prior authentication; authorization code flows delegate it to an OAuth 2.0 authorization server.
- **Security layering** -- PAR, PKCE, and DPoP are complementary security mechanisms that each address a different attack vector. Implementations vary in which layers they adopt.
- **SDK encapsulation depth** -- EUDI wallets delegate the entire authorization flow to the SDK, while Procivis exposes browser-based redirect handling at the application layer. Affinidi operates primarily in pre-authorized mode, minimizing client-side authorization complexity.
- **Sender-constrained tokens** -- DPoP binds access tokens to the client's key pair, preventing token theft and replay. Support for DPoP is a key differentiator across implementations.

---

## Prerequisites

Before reading this section, you should be familiar with:

- The credential offer structure and grant type parameters (see [02 -- Credential Offer](../02-credential-offer/))
- Issuer metadata discovery, including authorization server metadata (see [03 -- Issuer Metadata Discovery](../03-issuer-metadata/))

After this section, continue to [05 -- Nonce & Replay Protection](../05-nonce-and-replay-protection/) to understand how the `c_nonce` from the token response is used in proof construction.
