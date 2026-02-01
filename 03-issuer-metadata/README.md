# 03 -- Issuer Metadata Discovery

This section covers how wallets discover an issuer's capabilities, supported credential types, display information, and cryptographic requirements. Issuer metadata is the bridge between receiving a credential offer and understanding what the issuer can provide and how to interact with it.

---

## Documents

| # | Document | Scope |
|---|----------|-------|
| 1 | [Conceptual Overview](./conceptual-overview.md) | What issuer metadata is, the `/.well-known/openid-credential-issuer` endpoint, credential configurations, display metadata, proof types, and authorization server metadata. |
| 2 | [Protocol & Standards](./protocol-and-standards.md) | The OID4VCI Issuer Metadata specification: endpoint URL construction, response structure, credential configuration format, supported credential formats, and display properties. References specific specification sections. |
| 3 | [Implementation Details](./implementation-details.md) | How each wallet implementation (EUDI Android, EUDI iOS, Procivis ONE, Affinidi) retrieves, parses, caches, and uses issuer metadata. Code paths, format handling, and metadata mapping logic. |
| 4 | [Comparative Analysis](./comparative-analysis.md) | Side-by-side comparison of metadata access patterns, multi-issuer support, caching strategies, credential format diversity, and abstraction levels across all implementations. |

## Key Themes

- **Discovery as a protocol step** -- Issuer metadata retrieval is not optional. It is a required step between receiving a credential offer and initiating the token exchange. Without metadata, the wallet cannot construct valid credential requests.
- **Credential format diversity** -- The `credential_configurations_supported` object reveals what formats an issuer supports (mso_mdoc, jwt_vc_json, vc+sd-jwt). This determines the cryptographic operations the wallet must perform and the storage format for issued credentials.
- **Abstraction depth** -- Implementations differ dramatically in how much of the metadata layer is exposed to application code. EUDI wallets interact with metadata directly; Procivis ONE abstracts it behind the One Core SDK; Affinidi replaces dynamic discovery with configuration-driven setup.
- **Multi-issuer coordination** -- Wallets that support multiple issuers must manage metadata from each, with implications for caching, staleness, and user experience when browsing available credentials.

---

## Prerequisites

Before reading this section, you should be familiar with:

- How credential offers work and what they contain (see [02 -- Credential Offer](../02-credential-offer/))
- The high-level architecture of each wallet implementation (see [01 -- Architecture](../01-architecture/))

After this section, continue to [04 -- Authorization & Token](../04-authorization-and-token/) to understand how the wallet uses information from the issuer's metadata to obtain an access token.
