# Section 10: Transport Modes

This section covers the different mechanisms for transporting presentation requests and responses between verifiers, wallets, and holders.

---

## Contents

| File | Topic |
|------|-------|
| [conceptual-overview.md](./conceptual-overview.md) | Transport modes and their characteristics |
| [protocol-and-standards.md](./protocol-and-standards.md) | OID4VP response modes, ISO 18013-5 transport |
| [implementation-details.md](./implementation-details.md) | How EUDI, Procivis, and Affinidi implement transport |
| [comparative-analysis.md](./comparative-analysis.md) | Comparison of transport approaches |

---

## Key Concepts

- **Same-Device vs Cross-Device**: Verifier and wallet on same or different devices
- **Remote vs Proximity**: Network-based vs physical presence (BLE/NFC)
- **Response Modes**: How VP is delivered (redirect, direct_post, etc.)
- **Device Engagement**: ISO 18013-5 proximity handshake

---

## Transport Modes Overview

| Mode | Channel | Use Case |
|------|---------|----------|
| **Same-device redirect** | App-to-app | Mobile web login |
| **Cross-device QR** | Scan + HTTPS | Desktop verification with mobile wallet |
| **BLE proximity** | Bluetooth LE | In-person verification |
| **NFC tap** | Near Field | Contactless verification |

---

## Relationship to Other Sections

- **[02 Presentation Request](../02-presentation-request/)**: How requests are delivered
- **[08 Presentation Response](../08-presentation-response/)**: How responses are constructed
- **[04 Authorization Request](../04-authorization-request/)**: response_mode parameter
