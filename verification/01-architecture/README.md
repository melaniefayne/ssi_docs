# Architecture Overview

This section documents the high-level architecture of the verification/presentation flow in each implementation. Understanding the module structure, key classes, and data flow patterns is essential before diving into protocol specifics.

## Contents

1. [EUDI Wallet Architecture](./eudi-architecture.md) — Android and iOS verification architecture
2. [Procivis ONE Architecture](./procivis-architecture.md) — React Native cross-platform architecture
3. [Affinidi Architecture](./affinidi-architecture.md) — Cloud-based Iota Framework architecture
4. [Comparative Analysis](./comparative-analysis.md) — Side-by-side architectural comparison

---

## Common Architectural Patterns

Despite different technology stacks, all implementations share common architectural patterns for verification:

### 1. Session Coordination

All implementations maintain session state throughout the verification flow:

| Implementation | Session Coordinator | Purpose |
|----------------|---------------------|---------|
| EUDI Android | `WalletCorePresentationController` | Orchestrates request reception, credential selection, response sending |
| EUDI iOS | `RemoteSessionCoordinator`, `ProximitySessionCoordinator` | Protocol-specific session management |
| Procivis | Proof state machine | Tracks proof lifecycle (PENDING → REQUESTED → ACCEPTED) |
| Affinidi | Iota Session (correlationId, transactionId) | Links request initiation to response retrieval |

### 2. Layered Module Structure

All implementations separate concerns into distinct layers:

```
┌─────────────────────────────────────────┐
│  UI Layer (Screens / ViewModels)        │
├─────────────────────────────────────────┤
│  Business Logic (Interactors)           │
├─────────────────────────────────────────┤
│  Core Logic (Controllers / Coordinators)│
├─────────────────────────────────────────┤
│  SDK / Protocol Layer                   │
├─────────────────────────────────────────┤
│  Platform Services (Storage, Crypto)    │
└─────────────────────────────────────────┘
```

### 3. State Machine for Presentation

Verification flows through well-defined states:

```
IDLE
  │
  ▼ (receive request)
REQUEST_RECEIVED
  │
  ▼ (user views request)
CREDENTIAL_SELECTION
  │
  ▼ (user selects and consents)
AUTHENTICATION_REQUIRED
  │
  ▼ (biometric/PIN)
RESPONSE_SENDING
  │
  ▼ (transmission)
RESPONSE_SENT / ERROR
```

### 4. Transport Abstraction

All implementations abstract the transport mechanism:

- **Remote/Online** — HTTPS-based, works across devices
- **Proximity** — BLE/NFC, requires physical presence

The same credential selection and presentation logic operates regardless of transport.

---

## Key Differences Preview

| Aspect | EUDI | Procivis | Affinidi |
|--------|------|----------|----------|
| **Platform** | Native (Kotlin/Swift) | React Native | Cloud + Web SDK |
| **Protocol handling** | Wallet SDK | Core library | Iota Framework |
| **Signing location** | Device (Keystore/Enclave) | RSE (Remote Secure Element) | Vault (cloud) |
| **Transport modes** | HTTP, BLE, NFC | HTTP, BLE, NFC | HTTP (WebSocket, Redirect) |
| **PEX version** | DCQL (OID4VP v1.0) | PEX v1 + v2 | PEX |

---

## Architecture Deep Dives

Select your implementation to continue:

- [EUDI Wallet Architecture](./eudi-architecture.md) — Detailed module breakdown for Android and iOS
- [Procivis ONE Architecture](./procivis-architecture.md) — React Native component structure
- [Affinidi Architecture](./affinidi-architecture.md) — Cloud-based verification architecture
- [Comparative Analysis](./comparative-analysis.md) — Decision guidance based on architectural trade-offs
