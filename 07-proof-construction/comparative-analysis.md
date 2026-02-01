# Proof Construction -- Comparative Analysis

This document compares how EUDI Android, EUDI iOS, Procivis ONE, and Affinidi construct cryptographic proofs during credential issuance.

---

## Proof Signing Mechanisms

The most significant divergence across implementations is how the signing key is protected and how user authentication gates access to it.

| Implementation | Signing Location | User Auth Mechanism | Auth Trigger |
|----------------|-----------------|-------------------|--------------|
| EUDI Android | Device (Android Keystore / TEE / StrongBox) | BiometricPrompt (BIOMETRIC_WEAK or DEVICE_CREDENTIAL) | `IssueEvent.DocumentRequiresUserAuth` callback |
| EUDI iOS | Device (Secure Enclave) | Face ID / Touch ID via SystemBiometryController | SDK-internal trigger during proof signing |
| Procivis ONE (RSE) | Remote Signing Element | PIN entry (RSESign screen) | `PinEventType.SHOW_PIN` with `PinFlowType.TRANSACTION` |
| Procivis ONE (non-RSE) | Device (platform Secure Element) | Platform-dependent (may not require explicit auth) | SDK-internal |
| Affinidi | Cloud (Affinidi Vault) | Vault authentication (session-based) | No per-signing auth; session-level auth only |

### Biometric-Gated Signing (EUDI)

Both EUDI implementations gate proof signing behind biometric authentication. The signing key is stored in hardware-backed secure storage (Android Keystore or iOS Secure Enclave) with an access control policy that requires user authentication before each use.

**Strengths:**
- Hardware-backed key protection ensures the private key never leaves the secure element.
- Per-signing authentication means every proof requires active user consent.
- Biometric fallback to device credential ensures the flow is not blocked by biometric enrollment issues.

**Weaknesses:**
- Device-specific keys mean credential portability requires re-issuance on a new device.
- Biometric prompt interrupts the issuance UX, adding latency.
- Relies on platform biometric quality; BIOMETRIC_WEAK (Android) accepts Class 2 biometrics, which may not meet high-assurance requirements.

### PIN-Gated Signing (Procivis ONE RSE)

When using a Remote Signing Element, Procivis ONE requires PIN entry to authorize each signing operation. The PIN authenticates the user to the remote signing service, which performs the actual cryptographic operation.

**Strengths:**
- Key material resides in a remote HSM, providing enterprise-grade key protection.
- PIN-based authentication works across all devices without biometric hardware requirements.
- Remote key management enables credential portability -- the user can sign from any device that can reach the RSE.

**Weaknesses:**
- PIN entry is less convenient than biometric authentication.
- Requires network connectivity to the RSE for every signing operation.
- PIN-based authentication is susceptible to shoulder-surfing and brute-force attacks (mitigated by RSE lockout policies).

### Cloud Signing (Affinidi)

Affinidi performs proof construction server-side within the Vault. The user authenticates to the Vault at session level, and subsequent signing operations within the session do not require additional per-operation authentication.

**Strengths:**
- No dependency on device-specific hardware security features.
- Inherent credential portability -- any device can access the Vault.
- Simplified client implementation; no need to manage biometric or PIN flows.

**Weaknesses:**
- Key material is not hardware-backed on the user's device; security depends on cloud infrastructure.
- No per-signing user authentication; a compromised session could authorize multiple proof operations.
- Does not meet hardware key attestation requirements of eIDAS / EUDI / HAIP profiles.

---

## Supported Algorithms

| Algorithm | EUDI Android | EUDI iOS | Procivis ONE | Affinidi |
|-----------|-------------|---------|-------------|---------|
| ES256 (P-256) | Yes (primary) | Yes (primary) | Yes | No |
| EdDSA (Ed25519) | No | No | Yes | No |
| CRYSTALS-DILITHIUM | No | No | Yes | No |
| EcdsaSecp256k1Signature2019 | No | No | No | Yes (primary) |

### Analysis

**EUDI Android and iOS** use ES256 exclusively. This is dictated by the hardware capabilities of the Android Keystore (TEE/StrongBox) and iOS Secure Enclave, which natively support P-256 operations. The HAIP profile mandates ES256, making this the interoperability baseline.

**Procivis ONE** supports the broadest algorithm set. Beyond ES256 for EUDI compatibility, it offers EdDSA (Ed25519) for ecosystems that prefer Edwards curves, and CRYSTALS-DILITHIUM for post-quantum readiness. This breadth reflects Procivis's positioning as a multi-ecosystem wallet that must interoperate with diverse issuers.

**Affinidi** uses EcdsaSecp256k1Signature2019, a JSON-LD signature suite based on the secp256k1 curve. This curve is standard in blockchain-adjacent identity ecosystems (DID:elem, DID:key with secp256k1) but is not supported by the EUDI/HAIP ecosystem. This creates a fundamental interoperability boundary -- Affinidi cannot produce proofs that EUDI-compliant issuers accept, and vice versa.

---

## SDK Abstraction Levels

The depth of SDK encapsulation determines how much control (and responsibility) the application layer has over proof construction.

| Implementation | Proof Construction | User Auth Handling | Key Management |
|----------------|-------------------|-------------------|----------------|
| EUDI Android | SDK-internal | App layer (BiometricPrompt via event callback) | SDK-internal (Android Keystore) |
| EUDI iOS | SDK-internal | SDK-internal (SystemBiometryController) | SDK-internal (Secure Enclave) |
| Procivis ONE | SDK-internal (native bridge) | App layer (PIN screen via event callback) | SDK-internal (RSE or platform SE) |
| Affinidi | Server-side (Vault) | Session-level (Vault auth) | Server-side (Vault) |

### EUDI Android: Split Responsibility

The EUDI Android SDK constructs the proof internally but delegates user authentication to the application layer. The `IssueEvent.DocumentRequiresUserAuth` callback hands control to the app, which must present a BiometricPrompt, collect the authentication result, and resume the SDK event. This split allows the app to customize the authentication UX (e.g., prompt text, fallback options) while keeping proof construction logic in the SDK.

### EUDI iOS: Full SDK Encapsulation

The EUDI iOS SDK handles both proof construction and biometric authentication internally through `SystemBiometryController`. The application layer does not need to present authentication prompts or manage crypto objects -- it simply initiates the issuance flow and receives the result. This is the highest level of abstraction among the implementations.

### Procivis ONE: Native Bridge with Event Callback

Procivis ONE's React Native architecture means proof construction occurs in native code (One Core SDK) accessed through a bridge. The app layer handles PIN entry when the SDK emits a `PinEventType.SHOW_PIN` event, similar to EUDI Android's approach but with PIN instead of biometric authentication. The native bridge adds a layer of indirection compared to the EUDI implementations' direct SDK integration.

### Affinidi: Server-Side Abstraction

Affinidi abstracts proof construction entirely to the server. The client application (Vault) does not perform any local cryptographic operations for proof construction. This is the simplest client-side model but provides the least control over the signing process and does not support hardware-backed key attestation.

---

## User Authentication During Signing

| Aspect | EUDI Android | EUDI iOS | Procivis ONE | Affinidi |
|--------|-------------|---------|-------------|---------|
| Auth type | Biometric + device credential | Biometric (Face ID / Touch ID) | PIN (RSE) or none (non-RSE) | Session-level |
| Per-signing auth | Yes | Yes | Yes (RSE) / No (non-RSE) | No |
| Auth controller | DeviceAuthenticationController | SystemBiometryController | PinEventType callback | Vault session |
| Fallback | Device credential (PIN/pattern/password) | Device passcode | RSE lockout | Session re-auth |
| Hardware binding | CryptoObject from Keystore | Secure Enclave access control | RSE HSM | None |

---

## Strengths and Weaknesses

| Implementation | Strengths | Weaknesses |
|----------------|-----------|------------|
| **EUDI Android** | Hardware-backed signing with per-operation biometric auth. Broad device compatibility via BIOMETRIC_WEAK fallback. Flexible app-layer authentication UX. Standards-compliant (HAIP/eIDAS). | Limited to ES256. Device-specific keys prevent portability. BiometricPrompt adds UX friction. Weak biometric class may not meet highest assurance levels. |
| **EUDI iOS** | Secure Enclave provides strong hardware isolation. Fully SDK-encapsulated proof flow. Clean integration with iOS biometric APIs. Standards-compliant (HAIP/eIDAS). | Limited to ES256. Secure Enclave keys are non-exportable (no portability). iOS-only, no cross-platform consideration. |
| **Procivis ONE** | Broadest algorithm support (ES256, EdDSA, CRYSTALS-DILITHIUM). RSE enables credential portability. Multi-ecosystem compatibility. Post-quantum readiness. | RSE signing requires network connectivity. PIN-based auth is less convenient than biometrics. React Native bridge adds architectural complexity. RSE lockout on repeated PIN failures can block issuance. |
| **Affinidi** | Simplest client implementation. Inherent portability. No hardware dependency. | No hardware-backed key protection. No per-signing user auth. secp256k1 is incompatible with EUDI/HAIP ecosystem. Does not support hardware key attestation. Cloud dependency for all signing operations. |

---

## Interoperability Implications

The algorithm and signing mechanism choices create clear ecosystem boundaries:

1. **EUDI-compatible implementations** (EUDI Android, EUDI iOS, Procivis ONE with ES256) can interoperate with EUDI-compliant issuers that require ES256 proofs and hardware key attestation.

2. **Procivis ONE** can bridge ecosystems by supporting multiple algorithms and proof formats. It can produce ES256 proofs for EUDI issuers and EdDSA proofs for other ecosystems, making it the most versatile implementation for multi-issuer scenarios.

3. **Affinidi** operates in a separate ecosystem defined by JSON-LD VCs and secp256k1. Interoperability with EUDI-compliant issuers would require adding ES256 support and JWT proof construction, which is not currently implemented.

4. **HAIP compliance** requires hardware-backed keys and ES256. Only the EUDI implementations and Procivis ONE (in non-RSE, device SE mode) can satisfy this requirement. RSE-based signing in Procivis may or may not qualify depending on the RSE's certification level.
