# 09 -- Issuer Response

This section covers what happens after the credential request is sent: the issuer's response. The response may deliver a credential immediately, defer issuance for later retrieval, return a batch of credentials, or signal an error. How the wallet handles each of these outcomes -- and the UX paths it presents to the user -- is a critical part of the issuance flow that varies significantly across implementations.

---

## Documents

| # | Document | Scope |
|---|----------|-------|
| 1 | [Conceptual Overview](./conceptual-overview.md) | Immediate credential delivery, deferred issuance, batch responses, c_nonce rotation, notification endpoints, and error responses. |
| 2 | [Protocol & Standards](./protocol-and-standards.md) | OID4VCI response specification: credential response structure, deferred credential endpoint, notification endpoint, and error codes. |
| 3 | [Implementation Details](./implementation-details.md) | How each wallet implementation (EUDI Android, EUDI iOS, Procivis ONE, Affinidi) handles issuer responses -- callback systems, state management, deferred retrieval, dynamic issuance, and UX paths. |
| 4 | [Comparative Analysis](./comparative-analysis.md) | Side-by-side comparison of deferred issuance support, partial success handling, dynamic issuance, redirect mechanisms, notification support, and error handling. |

## Key Themes

- **Multiple outcome paths** -- The credential request can result in immediate delivery, deferred issuance, partial success, or failure. Implementations must handle all paths gracefully.
- **Deferred issuance** -- When the issuer cannot issue immediately, the wallet receives a `transaction_id` and must poll for the credential later. This requires persistent state management and a distinct UX path.
- **Dynamic issuance** -- The EUDI iOS implementation introduces a unique flow where the issuer requires an OID4VP presentation before completing issuance, blending issuance and presentation protocols.
- **Error recovery** -- Error handling varies from simple failure screens (Procivis) to partial success states (EUDI) that allow some credentials to be issued even when others fail.

---

## Prerequisites

Before reading this section, you should be familiar with:

- The credential request structure and how it is sent to the issuer (see [08 -- Credential Request](../08-credential-request/))
- Proof construction, since proof errors are a common failure mode in issuer responses (see [07 -- Proof Construction](../07-proof-construction/))

After this section, continue to [10 -- Secure Storage](../10-secure-storage/) to understand how issued credentials are stored on the device.
