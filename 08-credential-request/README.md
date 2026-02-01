# 08 -- Credential Request

This section covers the credential request phase of the OID4VCI issuance flow. After the wallet has obtained an access token and constructed a proof of possession, it sends a credential request to the issuer's credential endpoint. The credential request is the message that tells the issuer exactly which credential to issue, in what format, and with what key binding. This section includes a detailed comparison of OID4VCI Draft 13 and v1.0 (Final), which is critical for understanding the current fragmented state of the ecosystem.

---

## Documents

| # | Document | Scope |
|---|----------|-------|
| 1 | [Conceptual Overview](./conceptual-overview.md) | What the credential request contains, how the issuer validates it, immediate vs. deferred responses, and batch issuance support. |
| 2 | [Draft Comparison](./draft-comparison.md) | Deep comparison of OID4VCI Draft 13 vs. v1.0 (Final): field naming changes, endpoint behavior differences, breaking changes, HAIP profile alignment, and decision framework for choosing a draft version. |
| 3 | [Implementation Details](./implementation-details.md) | How each wallet implementation (EUDI Android, EUDI iOS, Procivis ONE, Affinidi) constructs credential requests -- draft version support, batch issuance configuration, and credential policies. |
| 4 | [Comparative Analysis](./comparative-analysis.md) | Side-by-side comparison of draft support, batch issuance capabilities, credential format support, HAIP compliance, and a decision framework for draft selection. |

## Key Themes

- **Draft fragmentation** -- The OID4VCI specification has evolved through multiple drafts, and the ecosystem is split between implementations using Draft 13 (widely deployed before ratification) and v1.0 Final (standards-compliant, ratified September 2025). This fragmentation is the single largest interoperability challenge in the credential issuance space.
- **Credential identification** -- How the wallet tells the issuer which credential to issue has changed between drafts (`credential_identifier` in Draft 13 vs. `credential_configuration_id` in v1.0 Final). Understanding this mapping is essential for cross-version interoperability.
- **Batch issuance** -- The ability to request multiple instances of a credential in a single request is important for one-time-use credential policies. Implementations vary significantly in their batch issuance support and configuration.
- **HAIP profile** -- The High Assurance Interoperability Profile adds requirements on top of the base OID4VCI specification, including mandatory PAR, DPoP, and hardware key attestation. Alignment with HAIP is a key differentiator and a requirement for eIDAS/EUDI compliance.

---

## Prerequisites

Before reading this section, you should be familiar with:

- Proof construction, including JWT proof type and DPoP (see [07 -- Proof Construction](../07-proof-construction/))
- The token response structure, including `c_nonce` (see [05 -- Nonce & Replay Protection](../05-nonce-and-replay-protection/))
- Issuer metadata, including `credential_configurations_supported` (see [03 -- Issuer Metadata Discovery](../03-issuer-metadata/))

After this section, continue to [09 -- Issuer Response](../09-issuer-response/) to understand how the issuer responds to the credential request.
