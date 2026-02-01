# Architectural Trade-Offs

This page provides a deep discussion of the fundamental architectural decisions that differentiate the three SSI implementations. Each decision involves genuine trade-offs with no universally correct answer -- the right choice depends on the deployment context.

---

## Native vs. Cross-Platform

### The Decision

Should the wallet be implemented as separate native applications for each platform (EUDI approach), or as a shared cross-platform core with thin native wrappers (Procivis approach)?

### EUDI: Separate Native Implementations

EUDI maintains independent Android (Kotlin) and iOS (Swift) implementations. Each platform's wallet is a first-class native application that directly uses platform APIs.

**Advantages:**
- **Direct hardware access.** The Android implementation calls `android.security.keystore` directly. The iOS implementation calls `Security.framework` directly. No abstraction layer introduces potential gaps or leaky abstractions.
- **Platform-idiomatic code.** Each implementation follows the conventions, patterns, and best practices of its platform. Android uses Room, Kotlin coroutines, and Jetpack Compose. iOS uses SwiftData, async/await, and SwiftUI.
- **Maximum optimization.** Platform-specific performance characteristics can be exploited. Memory management, threading models, and UI rendering are native.
- **Independent evolution.** Each platform can adopt new OS features immediately without waiting for cross-platform support. When Apple introduces a new Secure Enclave capability or Google updates the Keystore API, the native implementation can integrate it directly.

**Disadvantages:**
- **Double the development effort.** Every feature, bug fix, and protocol update must be implemented twice.
- **Behavioral divergence.** Two independent implementations of the same protocol specification may interpret edge cases differently, leading to subtle behavioral differences.
- **Double the testing burden.** Each implementation requires its own test suite, its own CI/CD pipeline, and its own interoperability testing.
- **Harder to maintain parity.** Over time, one platform may lag behind the other in feature completeness.

### Procivis: Cross-Platform Core (Rust)

Procivis implements the credential management logic, protocol handling, and cryptographic operations in a shared Rust core (One Core SDK). Platform-specific functionality (key storage, UI) is provided by thin native wrappers.

**Advantages:**
- **Single implementation of core logic.** Protocol parsing, credential format handling, state machine logic, and most cryptographic operations are implemented once. A bug fix in the Rust core applies to both platforms.
- **Guaranteed behavioral parity.** Both platforms execute the same code for core operations, eliminating behavioral divergence.
- **Reduced total development effort.** While the native wrappers still require platform expertise, the bulk of the logic is shared.
- **Rust's safety properties.** Memory safety without garbage collection, thread safety guarantees, and strong type system reduce entire categories of bugs.

**Disadvantages:**
- **Abstraction overhead.** The key storage interface must abstract across fundamentally different platform APIs. Android Keystore and iOS Secure Enclave have different capabilities, error modes, and configuration options. The abstraction must handle the union of both without exposing platform-specific details.
- **FFI boundary complexity.** Calling Rust from Kotlin (via JNI) and Swift (via C FFI) introduces marshaling overhead and debugging complexity. Stack traces across the FFI boundary are harder to interpret.
- **Rust expertise required.** The core team must be proficient in Rust, which has a steeper learning curve than Kotlin or Swift. Debugging the Rust core requires different tools and mental models than debugging the native wrappers.
- **Platform update lag.** When a platform introduces a new capability, the cross-platform abstraction must be updated to support it, which may take longer than a direct native integration.

### Affinidi: Cloud Service (No Native Wallet)

Affinidi sidesteps the native/cross-platform decision entirely by providing a cloud service with multi-language SDKs. There is no wallet application to build.

**Advantages:**
- **No platform-specific code at all.** The entire credential management system runs server-side.
- **Language flexibility.** Integrators choose the SDK language that fits their existing stack.
- **No app store distribution.** No need to manage mobile app releases, reviews, or updates.

**Disadvantages:**
- **No wallet UX.** The holder experience depends on third-party wallets or custom web interfaces.
- **No device hardware access.** Cannot leverage TEE, Secure Enclave, or other device security features.
- **Network dependency.** Every operation requires connectivity to the Affinidi cloud.

### Verdict

The choice depends on the team and the product:
- If you have separate Android and iOS teams and need maximum platform optimization, choose native (EUDI-style).
- If you have a full-stack team with Rust capability and need behavioral parity across platforms, choose cross-platform (Procivis-style).
- If you are building a backend service and not a wallet, choose cloud (Affinidi-style).

---

## Thick Client vs. Cloud Service

### The Decision

Should the wallet perform all credential operations locally on the device (thick client), or delegate operations to a cloud backend (cloud service)?

### Thick Client (EUDI, Procivis Local)

The wallet stores credentials, manages keys, executes cryptographic operations, and handles protocol flows entirely on the device. No server-side component is required for core operations.

**Advantages:**
- **Offline capability.** The wallet functions without network connectivity. Credentials can be presented via BLE or NFC.
- **Data minimization.** Credential data never leaves the device (except during presentation to a verifier). There is no cloud provider that has access to the holder's credentials.
- **Hardware security.** Cryptographic keys are in device hardware. The security model does not depend on the security of a remote server.
- **User control.** The holder has full sovereignty over their credentials. No third party can revoke access to the wallet (though issuers can revoke credentials).

**Disadvantages:**
- **Device-bound credentials.** If the device is lost, all credentials are lost. Re-issuance is required.
- **Update complexity.** Protocol updates, bug fixes, and security patches require app updates distributed through app stores. Users who do not update are vulnerable.
- **Device diversity.** The wallet must work correctly across thousands of device models with varying hardware capabilities, OS versions, and manufacturer customizations.

### Cloud Service (Affinidi, Procivis RSE)

The wallet delegates some or all operations to a cloud backend. Keys may be managed remotely, and credentials may be stored in a cloud vault.

**Advantages:**
- **Multi-device access.** Credentials are accessible from any authenticated device.
- **Survivability.** Device loss does not mean credential loss.
- **Centralized updates.** Protocol changes and bug fixes are deployed server-side without requiring user action.
- **Reduced device requirements.** The device does not need specialized hardware (TEE, Secure Enclave) for core security.

**Disadvantages:**
- **Network dependency.** Every credential operation requires network connectivity.
- **Cloud trust.** The cloud provider has access to (or manages) credential data and key material. This is a significant trust assumption.
- **Regulatory implications.** Some regulations (eIDAS 2.0) require that keys be under the holder's exclusive control, which is harder to demonstrate with cloud-managed keys.
- **Availability risk.** If the cloud service is down, no credential operations are possible.

### Hybrid Approaches

Procivis's RSE mode is a hybrid: the wallet application runs locally (thick client for UX and protocol flow), but key operations are delegated to a remote HSM (cloud for key management). This provides organizational key control while maintaining a native wallet experience.

---

## Single-Standard vs. Multi-Standard

### The Decision

Should the wallet implement a single standard deeply (EUDI approach), or support multiple standards broadly (Procivis approach)?

### Single-Standard (EUDI)

EUDI focuses on the OID4VCI/OID4VP ecosystem with mDoc and SD-JWT formats. It implements these standards deeply and correctly, with full support for DPoP, PAR, PKCE, wallet attestation, credential policies, and batch issuance.

**Advantages:**
- **Depth of implementation.** Every edge case, error condition, and optional feature of the supported standards is handled.
- **Reduced attack surface.** Fewer code paths mean fewer potential vulnerabilities.
- **Clear compliance target.** The certification scope is well-defined.
- **Interoperability confidence.** Deep testing against a single standard produces high confidence in interoperability with conformant counterparts.

**Disadvantages:**
- **Limited ecosystem reach.** Cannot interact with issuers or verifiers using unsupported standards.
- **Migration risk.** If the standard evolves in an incompatible direction, significant rework may be needed.
- **Vendor/issuer lock-in.** Issuers and verifiers must also implement the same standard.

### Multi-Standard (Procivis)

Procivis supports OID4VCI (multiple draft versions), four credential formats, five cryptographic algorithms, and five transport protocols.

**Advantages:**
- **Maximum interoperability.** Can interact with the widest range of issuers and verifiers.
- **Future-proofing.** If one standard becomes dominant, the wallet already supports it.
- **Flexible deployment.** A single wallet can be configured for different standards in different markets.
- **Negotiation capability.** The wallet can negotiate the format and protocol that both parties support.

**Disadvantages:**
- **Combinatorial complexity.** Four formats times five algorithms times five transports creates a large matrix of combinations to test. Not all combinations may be well-tested.
- **Shallow implementation risk.** Supporting many standards may mean none are implemented to the same depth as a focused implementation.
- **Larger codebase.** More code means more maintenance, more potential bugs, and a larger binary size.
- **Configuration complexity.** Operators must understand the trade-offs of each standard to configure the wallet correctly.

---

## Hardware-Only vs. Flexible Key Storage

### The Decision

Should the wallet require hardware-backed key storage (EUDI approach), or support a range of key storage backends including software and cloud options (Procivis, Affinidi approaches)?

### Hardware-Only (EUDI)

EUDI requires keys to be stored in device hardware (TEE/Strongbox on Android, Secure Enclave on iOS). Software key storage exists only for development and testing.

**Advantages:**
- **Strongest security guarantee.** Keys are physically non-extractable. No software vulnerability can leak key material.
- **Clear attestation model.** Hardware attestation chains provide cryptographic proof of key properties.
- **Regulatory alignment.** Meets the strictest interpretation of WSCD requirements.

**Disadvantages:**
- **Device dependency.** Not all devices support the required hardware. Older or cheaper devices may lack Strongbox or have limited TEE capabilities.
- **No multi-device.** Hardware-bound keys are, by definition, bound to one device.
- **No recovery.** If the device is lost or destroyed, keys are irrecoverable.

### Flexible Key Storage (Procivis, Affinidi)

Procivis supports hardware (SECURE_ELEMENT), remote HSM (UBIQU_RSE), and software key storage. Affinidi uses cloud key management exclusively.

**Advantages:**
- **Broader device compatibility.** Devices without hardware security can still use the wallet (at reduced security).
- **Organizational control.** RSE allows enterprises to manage keys centrally.
- **Multi-device access.** Cloud or remote keys can be accessed from multiple devices.
- **Recovery options.** Remote and cloud keys survive device loss.

**Disadvantages:**
- **Weaker security floor.** If software key storage is used (even as a fallback), the security guarantee is only as strong as the weakest option.
- **Configuration risk.** Operators must understand the implications of each key storage option. Choosing software storage for production deployments undermines the security model.
- **Attestation gaps.** Software and cloud keys cannot produce hardware attestation chains. Issuers that require hardware attestation will reject these keys.

---

## Platform-Specific Optimization vs. Portability

### The Decision

Should the implementation exploit every platform-specific feature for optimal performance and security, or prioritize portability across platforms?

### Platform-Specific Optimization (EUDI)

EUDI uses Room Database on Android and SwiftData on iOS. It uses Jetpack Compose on Android and SwiftUI on iOS. Each platform's implementation leverages the best available tools.

**Benefits:** Maximum performance, smallest binary, most natural UX, direct access to latest platform features.

**Cost:** Two completely separate implementations to build, test, and maintain.

### Portability (Procivis)

Procivis uses Rust for the core logic and thin native wrappers for platform interaction. The data model, protocol logic, and cryptographic operations are portable.

**Benefits:** Reduced duplication, behavioral consistency, single implementation to audit.

**Cost:** Abstraction overhead, FFI complexity, potential limitations in exploiting platform-specific features, need for Rust expertise.

---

## SDK Boundary: Thin Wrapper vs. Thick Abstraction

### The Decision

Where should the boundary be drawn between the SDK and the application? Should the SDK be a thin wrapper around platform APIs (minimal abstraction), or a thick abstraction that encapsulates entire subsystems?

### Thin Wrapper (EUDI)

EUDI's libraries (EudiWalletKit, EudiOpenId4Vci) are focused libraries that handle specific concerns. The wallet application orchestrates them, making decisions about UI flow, state management, and user interaction.

**Characteristics:**
- The application has fine-grained control over every step
- The SDK exposes protocol primitives, not high-level workflows
- The developer must understand the underlying protocols to use the SDK correctly
- Customization is straightforward (override any step)

### Thick Abstraction (Procivis One Core)

Procivis One Core encapsulates credential management, key storage, protocol handling, and format processing in a single SDK. The application interacts with high-level operations ("issue credential", "present credential") rather than protocol primitives.

**Characteristics:**
- The application has less visibility into individual protocol steps
- The SDK handles protocol details internally
- The developer can use the SDK without deep protocol knowledge
- Customization requires understanding the SDK's extension points

### Affinidi: API Boundary

Affinidi's boundary is even thicker: the entire system is behind an API. The integrator calls REST endpoints or SDK methods that trigger server-side operations.

**Characteristics:**
- Maximum encapsulation: the integrator sees only inputs and outputs
- Minimum control: the integrator cannot customize protocol-level behavior
- Simplest integration: a few API calls to issue a credential
- Least flexibility: the API defines the possible operations

### Boundary Comparison

```
Thin Wrapper (EUDI):
  App --> EudiOpenId4Vci.resolveOffer()
  App --> EudiOpenId4Vci.authorize()       // App controls each step
  App --> EudiOpenId4Vci.requestCredential()
  App --> Store credential in Room DB

Thick Abstraction (Procivis):
  App --> OneCore.issueCredential(offer)   // SDK handles all steps
  App <-- Credential stored internally

API (Affinidi):
  Backend --> AffinidiSDK.issueCredential(claims)  // Cloud handles everything
  Backend <-- Credential ID
```

The thinner the wrapper, the more control (and responsibility) the developer has. The thicker the abstraction, the simpler the integration but the less flexibility for customization.

---

## Summary of Trade-Off Positions

| Dimension | EUDI | Procivis | Affinidi |
|-----------|------|----------|----------|
| Native vs. Cross-Platform | Native (2 apps) | Cross-platform core (Rust) | Cloud service |
| Thick Client vs. Cloud | Thick client | Thick client + optional cloud (RSE) | Cloud service |
| Single vs. Multi Standard | Single standard (deep) | Multi-standard (broad) | Single standard (simple) |
| Hardware vs. Flexible Keys | Hardware-only | Flexible (hardware + RSE + software) | Cloud-only |
| Platform Optimization | Maximum | Moderate (portable core) | N/A |
| SDK Boundary | Thin wrapper | Thick abstraction | API |

Each position is a deliberate architectural choice that serves a specific set of requirements. There is no universally superior position -- only the one that best fits the deployment context.
