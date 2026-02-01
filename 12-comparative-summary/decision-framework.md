# Decision Framework

This page provides guidance for choosing between the three SSI implementation approaches based on project requirements, constraints, and priorities.

---

## Choose EUDI-Style If

The EUDI Reference Wallet approach is designed for government-grade identity systems operating under regulatory frameworks.

### Primary Indicators

- **Building for the EU market.** EUDI is the reference implementation for the European Digital Identity framework. Choosing this approach aligns with the regulatory trajectory of the EU.
- **Need eIDAS 2.0 compliance.** The architecture is designed to meet eIDAS 2.0 requirements, including WSCD (Wallet Secure Cryptographic Device) certification, qualified electronic signatures, and cross-border interoperability.
- **Regulatory requirements drive the design.** When the credential system must satisfy government auditors, certification bodies, or legal frameworks, EUDI's architecture provides the clearest compliance path.
- **Government ID use cases.** PID (Person Identification Data), mDL (mobile driving license), and other government-issued credentials benefit from the hardware-backed, one-time-use credential model.
- **Need certified hardware-backed keys.** When the threat model requires proof that keys were generated in and never left certified security hardware (TEE/Strongbox/Secure Enclave), with attestation chains back to the device manufacturer.

### What You Get

- Platform-native implementations for Android and iOS, optimized for each platform's security features
- Full OID4VCI implementation with DPoP, PAR, PKCE, and wallet attestation
- mDoc and SD-JWT credential formats with selective disclosure
- Credential policies (OneTimeUse, RotateUse) designed for privacy
- Hardware key attestation integrated into the issuance flow
- Transaction logging and revocation tracking

### What You Trade Off

- **Two codebases.** Android and iOS implementations are separate, requiring parallel development and maintenance.
- **Limited credential format support.** Only mDoc and SD-JWT; no JSON-LD VC or JWT VC.
- **No offline-first transport diversity.** BLE is supported but MQTT and other IoT transports are not.
- **No post-quantum cryptography.** Only classical algorithms (ES256).
- **No multi-device credential access.** Credentials are device-bound by design.

### Ideal Project Profile

```
Project: National digital identity wallet
Jurisdiction: EU member state
Users: Citizens (millions)
Credentials: PID, mDL, health certificates
Regulatory: eIDAS 2.0, WSCD certification required
Security: Maximum (government-grade)
Timeline: 12-24 months
Team: Separate Android and iOS teams
```

---

## Choose Procivis-Style If

Procivis One provides the broadest feature coverage through a cross-platform architecture built on a Rust core.

### Primary Indicators

- **Need cross-platform from a single codebase.** The Rust-based One Core SDK provides a single implementation of credential management logic, with thin platform-specific wrappers for Android and iOS.
- **Need maximum interoperability.** Procivis supports multiple OID4VCI draft versions, all four major credential formats (mDoc, SD-JWT, JSON-LD VC, JWT VC), and multiple transport protocols (HTTPS, BLE, NFC, MQTT).
- **Enterprise deployments with remote key management.** The UBIQU RSE (Remote Secure Element) option allows organizations to manage keys centrally via HSMs while still providing a mobile wallet UX.
- **Need post-quantum readiness.** CRYSTALS-DILITHIUM 3 support provides a migration path for when quantum computers threaten classical algorithms.
- **Need BLE/MQTT for offline or IoT scenarios.** When credentials must be presented in environments without internet connectivity or to IoT devices.

### What You Get

- Single Rust core with platform-specific wrappers (reduced duplication)
- Four credential formats: mDoc, SD-JWT, JSON-LD VC, JWT VC
- Five cryptographic algorithms: ES256, EdDSA, secp256k1, DILITHIUM, BBS+
- Five transport protocols: HTTPS, BLE, NFC, MQTT, deep links
- Local hardware keys (SECURE_ELEMENT) and remote HSM keys (UBIQU_RSE)
- Multi-draft OID4VCI support for broad issuer compatibility

### What You Trade Off

- **Complexity.** Supporting everything means more configuration, more edge cases, and a larger surface area to test and maintain.
- **Rust dependency.** The core SDK is written in Rust, which requires Rust expertise for deep customization or debugging.
- **Partial open source.** The SDK is open, but the core may have proprietary components.
- **Platform-specific optimizations.** The cross-platform abstraction may not exploit every platform-specific capability as deeply as a native implementation.

### Ideal Project Profile

```
Project: Enterprise identity platform
Jurisdiction: Multi-country (EU + non-EU)
Users: Employees, partners, customers
Credentials: Employee badges, certificates, compliance attestations
Regulatory: Varies by jurisdiction, need flexibility
Security: High (hardware + RSE options)
Timeline: 6-18 months
Team: Full-stack with Rust capability
```

---

## Choose Affinidi-Style If

Affinidi provides a cloud-first, API-driven approach optimized for issuer-focused use cases and rapid development.

### Primary Indicators

- **Building a cloud-first service.** When the credential system is a backend service rather than a mobile wallet, Affinidi's API-driven model is the natural fit.
- **Need fastest time-to-market.** Affinidi's SDKs and APIs allow developers to issue credentials within hours, not weeks. The cloud infrastructure is pre-provisioned.
- **Issuer-focused (not building a wallet).** If the project is about issuing credentials to holders who use third-party wallets, Affinidi provides a complete issuance platform without requiring wallet development.
- **Need multi-language SDK support.** Affinidi provides SDKs in TypeScript, Python, Kotlin, and Swift, allowing integration into diverse backend environments.
- **Simpler issuance requirements.** When the use case does not require selective disclosure, hardware-backed keys, or offline presentation.

### What You Get

- Cloud-hosted credential issuance and management
- Multi-language SDKs (TypeScript, Python, Kotlin, Swift)
- W3C VC (JSON-LD) format with Data Integrity proofs
- Multi-device credential access via Affinidi Vault
- Multi-profile support (Edge and Cloud profiles)
- Rapid integration with existing web services

### What You Trade Off

- **No hardware-backed keys on device.** Keys are cloud-managed, not bound to device hardware. This may not satisfy WSCD requirements.
- **No selective disclosure.** JSON-LD VCs do not natively support selective disclosure (without BBS+ extension).
- **No offline presentation.** Cloud-dependent architecture requires network connectivity.
- **Limited protocol support.** No DPoP, PAR, deferred issuance, or batch issuance.
- **Vendor dependency.** The Affinidi Vault is a hosted service; credentials and keys are managed by Affinidi's infrastructure.
- **Regulatory limitations.** May not meet eIDAS 2.0, HAIP, or other government-grade security requirements.

### Ideal Project Profile

```
Project: Customer loyalty or membership credential platform
Jurisdiction: Global (no specific regulatory mandate)
Users: Consumers using existing wallets
Credentials: Membership cards, certificates, attestations
Regulatory: Minimal (no government ID requirements)
Security: Standard (cloud-grade)
Timeline: 1-4 weeks
Team: Web developers (TypeScript/Python)
```

---

## Trade-Off Dimensions

### Security vs. User Experience

| Dimension | High Security (EUDI) | Balanced (Procivis) | High UX (Affinidi) |
|-----------|---------------------|---------------------|---------------------|
| Key storage | Hardware-only | Hardware + RSE options | Cloud-managed |
| Authentication | Biometric + PIN per use | Configurable | Cloud auth |
| Credential policies | OneTimeUse (10 creds) | Configurable | N/A |
| Device binding | Strict (no multi-device) | Flexible | None (multi-device) |
| Recovery on device loss | Re-issuance required | Re-issuance or RSE | Cloud access |

Higher security creates friction: biometric prompts for every presentation, inability to access credentials from another device, and need for re-issuance if the device is lost. Lower security reduces friction but increases risk.

### Compliance vs. Flexibility

| Dimension | Compliance-First (EUDI) | Flexible (Procivis) | Speed-First (Affinidi) |
|-----------|------------------------|---------------------|------------------------|
| Standards | eIDAS 2.0, HAIP, ISO | Multi-standard | W3C VC |
| Formats | mDoc + SD-JWT | All four | JSON-LD only |
| Certification path | Clear (EU framework) | Adaptable | Unclear |
| Jurisdictional fit | EU-optimized | Multi-jurisdiction | Global/generic |

Compliance-first approaches constrain design choices but provide a clear certification path. Flexible approaches support more scenarios but require per-deployment compliance validation.

### Native Performance vs. Cross-Platform Reach

| Dimension | Native (EUDI) | Cross-Platform (Procivis) | Cloud (Affinidi) |
|-----------|--------------|---------------------------|-------------------|
| Platform optimization | Maximum per platform | Good (Rust core) | N/A (server-side) |
| Development cost | 2x (Android + iOS) | 1x (shared core) | 1x (API integration) |
| Platform-specific features | Full access | Abstracted access | No device features |
| Maintenance burden | Higher (two codebases) | Lower (one core) | Lowest (hosted) |
| UI/UX control | Full native UI | Native UI, shared logic | No wallet UI |

---

## Decision Matrix

Use this matrix to quickly identify the best-fit implementation based on your top requirements:

| If your top requirement is... | Choose |
|------------------------------|--------|
| eIDAS 2.0 compliance | EUDI |
| Government-issued identity credentials | EUDI |
| Hardware-backed key attestation | EUDI or Procivis |
| Maximum credential format support | Procivis |
| Post-quantum cryptography | Procivis |
| Offline presentation (BLE/NFC) | EUDI or Procivis |
| Remote key management (enterprise HSM) | Procivis |
| Cross-platform single codebase | Procivis |
| Fastest time to market | Affinidi |
| Cloud-first / serverless architecture | Affinidi |
| Issuer-only (no wallet development) | Affinidi |
| Multi-language SDK support | Affinidi |
| Multi-device credential access | Affinidi or Procivis (RSE) |
| W3C VC / JSON-LD ecosystem | Affinidi or Procivis |
| IoT / MQTT transport | Procivis |
| Minimum operational complexity | Affinidi |
