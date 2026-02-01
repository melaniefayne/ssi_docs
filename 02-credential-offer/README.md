# 02 — Credential Offer

This section covers the very first step in the OID4VCI issuance flow: the **Credential Offer**. A credential offer is the mechanism by which an issuer signals to a wallet that one or more credentials are available for issuance, along with the information needed to begin the protocol exchange.

---

## Documents

| # | Document | Scope |
|---|----------|-------|
| 1 | [Conceptual Overview](./conceptual-overview.md) | What credential offers are, how they fit into the issuance flow, the OID4VCI Credential Offer object structure, delivery mechanisms, and the distinction between issuer-initiated and wallet-initiated flows. |
| 2 | [Protocol & Standards](./protocol-and-standards.md) | The OID4VCI Credential Offer specification in detail: URI formats, grant types (Pre-Authorized Code, Authorization Code), TxCode parameters, and the offer resolution process. References specific specification sections. |
| 3 | [Implementation Details](./implementation-details.md) | How each wallet implementation (EUDI Android, EUDI iOS, Procivis ONE, Affinidi) resolves, validates, and processes credential offers. Code paths, URI scheme handling, transport mechanisms, and TxCode validation logic. |
| 4 | [Comparative Analysis](./comparative-analysis.md) | Side-by-side comparison of offer delivery mechanisms, transport diversity, TxCode handling, regulatory constraints, and wallet-initiated vs issuer-initiated support across all implementations. |

## Key Themes

- **Delivery diversity** -- Credential offers can arrive via QR codes, deep links, universal links, NFC, BLE, or MQTT. The choice of delivery mechanism determines the user experience and constrains the deployment context.
- **Grant type selection** -- The Pre-Authorized Code Grant and Authorization Code Grant serve different trust models. Pre-authorized flows assume the issuer has already authenticated the holder; authorization code flows delegate authentication to an OAuth 2.0 authorization server.
- **TxCode as a second factor** -- Transaction codes provide an out-of-band verification step, binding a specific user to a specific offer. Implementations differ in what input modes they accept and how they validate length constraints.
- **Regulatory constraints** -- The EUDI wallet enforces a PID (Person Identification Data) requirement mandated by EU regulation, which has no equivalent in the other implementations.
- **Transport abstraction** -- Procivis ONE's multi-transport architecture (HTTP, BLE, MQTT) contrasts sharply with the HTTP-only approaches of EUDI and Affinidi, reflecting fundamentally different assumptions about connectivity.

---

## Prerequisites

Before reading this section, you should be familiar with:

- The SSI trust triangle and the role of the issuer (see [00 -- Introduction](../00-introduction/))
- The high-level architecture of each wallet implementation (see [01 -- Architecture](../01-architecture/))

After this section, continue to [03 -- Issuer Metadata Discovery](../03-issuer-metadata/) to understand how the wallet retrieves the issuer's capabilities after receiving an offer.
