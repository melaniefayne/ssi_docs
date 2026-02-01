# Implementations Overview

This documentation analyzes credential issuance across three distinct SSI wallet implementations. Each represents a fundamentally different approach to solving the same problem: securely delivering a Verifiable Credential from an issuer to a holder.

This document profiles each implementation and provides a comparison table. No architecture or code details here — those are covered starting in [01 — Architecture Overview](../01-architecture/).

---

## EUDI Wallet (EU Digital Identity)

The European Digital Identity Wallet is a reference implementation developed under the EU Digital Identity framework (eIDAS 2.0). It is regulation-driven: the architecture, supported formats, and security requirements are shaped by European law and the Architecture and Reference Framework (ARF).

### Technical Profile

| Attribute | Android | iOS |
|-----------|---------|-----|
| **Language** | Kotlin | Swift |
| **Core SDK** | `eudi-lib-android-wallet-core` v0.23.0 | EudiWalletKit v0.19.4 |
| **Platform** | Native Android | Native iOS |
| **OID4VCI Version** | v1.0 (Final) | v1.0 (Final) |
| **Credential Formats** | mDoc (ISO 18013-5), SD-JWT-VC | mDoc (ISO 18013-5), SD-JWT-VC |
| **Key Storage** | Android Keystore (hardware-backed) | iOS Secure Enclave |
| **Authentication** | Biometric (device-level) | Biometric (device-level) |
| **Authorization** | OAuth 2.0 with PAR and DPoP | OAuth 2.0 with PAR and DPoP |

### Key Characteristics

- **Regulatory mandate**: The EUDI Wallet is not just a technical project. It exists to satisfy the requirements of eIDAS 2.0, which mandates that EU member states provide digital identity wallets to their citizens. This regulatory context drives design decisions — for example, the mandatory support for the EU Person Identification Data (PID) credential.

- **Native platform implementations**: Android and iOS are developed as separate native codebases, not a cross-platform wrapper. Each leverages platform-specific security primitives directly.

- **Hardware-backed keys**: Both platforms require hardware-backed key storage. On Android, this means Android Keystore with StrongBox or TEE backing. On iOS, this means the Secure Enclave. Software-only key storage is not an option.

- **PAR and DPoP support**: The EUDI implementation supports Pushed Authorization Requests (RFC 9126) and Demonstration of Proof-of-Possession (RFC 9449), adding transport-level security to the OAuth 2.0 authorization flow.

- **Format focus**: The EUDI Wallet supports two credential formats — mDoc and SD-JWT-VC — reflecting the EU's focus on government-issued identity documents and selective disclosure.

- **Open source**: Both codebases are publicly available and actively developed. Source code analysis in this documentation references specific files and versions.

---

## Procivis ONE Wallet

Procivis ONE is a cross-platform wallet built with React Native, backed by a Rust-based core library. It is designed for flexibility: support for multiple protocol versions, multiple credential formats, multiple transport mechanisms, and multiple key storage backends.

### Technical Profile

| Attribute | Value |
|-----------|-------|
| **Language** | TypeScript (React Native), Rust (core) |
| **Core SDK** | `@procivis/react-native-one-core` v1.81885.0 |
| **Platform** | Cross-platform (Android and iOS via React Native) |
| **OID4VCI Versions** | Draft 13, Final 1, HAIP, Swiyu |
| **Credential Formats** | W3C VC (JSON-LD), SD-JWT, JSON-LD with BBS+, ISO mDL (mDoc) |
| **Key Storage** | Secure Element, Android Keystore, Ubiqu Remote Secure Element (RSE) |
| **Transport Protocols** | HTTP, BLE (Bluetooth Low Energy), MQTT |
| **Post-Quantum Support** | CRYSTALS-DILITHIUM |

### Key Characteristics

- **Multi-protocol support**: Procivis ONE does not commit to a single OID4VCI version. It supports Draft 13, Final 1, the HAIP (High Assurance Interoperability Profile), and the Swiss Swiyu profile. This makes it adaptable to different ecosystems and regulatory regimes.

- **Cross-platform via React Native**: The UI layer is TypeScript/React Native, but the cryptographic and protocol logic lives in a Rust core library exposed via native bindings. This gives cross-platform reach without sacrificing performance or security in the core.

- **Broad format support**: Procivis supports the widest range of credential formats of the three implementations: W3C Verifiable Credentials (JSON-LD), SD-JWT, JSON-LD with BBS+ signatures (enabling unlinkable selective disclosure), and ISO mDL. This breadth is a deliberate design choice for ecosystem interoperability.

- **Multiple key storage backends**: Beyond on-device hardware key storage (Secure Element, Android Keystore), Procivis supports remote key storage via Ubiqu Remote Secure Element (RSE). This allows key material to be managed in a cloud HSM while remaining under the holder's control — a model relevant for enterprise deployments.

- **Multiple transport mechanisms**: Credential exchange can occur over HTTP (the standard OID4VCI transport), BLE (for proximity-based issuance), or MQTT (for asynchronous or IoT-adjacent scenarios). This is unusual; most implementations support only HTTP.

- **Post-quantum cryptography**: Procivis includes support for CRYSTALS-DILITHIUM, a lattice-based digital signature scheme selected by NIST for post-quantum standardization. While not yet required by any production trust framework, this forward-looking support addresses the long-term threat of quantum computing to current signature schemes.

- **Open source**: The core library and wallet application are publicly available. Source code analysis in this documentation references specific files and versions.

---

## Affinidi

Affinidi provides a cloud-based credential issuance service paired with a holder-side application (Affinidi Vault). Unlike the EUDI and Procivis implementations, Affinidi is a managed service: the issuance infrastructure runs in Affinidi's cloud, and integrators interact with it through a multi-language Trust Development Kit (TDK).

### Technical Profile

| Attribute | Value |
|-----------|-------|
| **Architecture** | Cloud-based issuance service + Affinidi Vault (holder) |
| **TDK Languages** | TypeScript, Python, Java, Kotlin, .NET, and others |
| **OID4VCI Version** | OID4VCI with pre-authorized code flow |
| **Credential Format** | W3C VC Data Model with JSON-LD |
| **Signature Algorithm** | EcdsaSecp256k1Signature2019 |
| **DID Method** | `did:key` |
| **Claim Modes** | TX_CODE (user-provided transaction code), FIXED_HOLDER (pre-determined holder DID) |
| **Revocation** | Supported (credential status management) |

### Key Characteristics

- **Cloud-native issuance**: The issuer-side logic (credential construction, signing, delivery) runs as a managed cloud service. Integrators do not deploy or manage issuance infrastructure themselves. This is a fundamentally different operational model from EUDI or Procivis, where the wallet developer controls the full stack.

- **Multi-language TDK**: Affinidi provides SDKs in TypeScript, Python, Java, Kotlin, .NET, and additional languages. This lowers the integration barrier for backend teams that do not work in Kotlin or Swift.

- **Pre-authorized code flow**: Affinidi's documented issuance flow centers on the pre-authorized code flow variant of OID4VCI. The issuer creates a credential offer with an embedded pre-authorized code, optionally requiring the holder to enter a transaction code (TX_CODE) for additional verification.

- **W3C VC with JSON-LD**: Affinidi uses the W3C Verifiable Credentials Data Model serialized as JSON-LD, with `EcdsaSecp256k1Signature2019` as the proof mechanism. This is a specific choice: it prioritizes semantic interoperability (via JSON-LD contexts) over the selective disclosure properties of SD-JWT or the hardware optimization of mDoc.

- **`did:key` method**: Affinidi uses `did:key` as its DID method, which encodes the public key directly in the DID. This is simple and self-contained — no DID resolution infrastructure is needed — but limits key rotation and multi-key scenarios.

- **Claim modes**: Affinidi supports two modes for associating a credential with a holder. In TX_CODE mode, the holder proves their identity by entering a transaction code received out-of-band. In FIXED_HOLDER mode, the issuer specifies the holder's DID at offer creation time, and the credential is bound to that DID.

- **Credential revocation**: Affinidi supports credential revocation through a status management mechanism, allowing issuers to invalidate credentials after issuance.

- **Analysis basis**: Unlike EUDI and Procivis, the Affinidi analysis in this documentation is based on public documentation, API references, and published guides — not source code inspection. This is noted throughout the documentation where it affects the depth of analysis.

---

## Comparison Table

| Dimension | EUDI Wallet | Procivis ONE | Affinidi |
|-----------|-------------|--------------|----------|
| **Architecture** | Native mobile (Android + iOS) | Cross-platform (React Native + Rust core) | Cloud service + holder vault |
| **Languages** | Kotlin, Swift | TypeScript, Rust | Multi-language TDK (TS, Python, Java, etc.) |
| **Core SDK** | `eudi-lib-android-wallet-core` v0.23.0 / EudiWalletKit v0.19.4 | `@procivis/react-native-one-core` v1.81885.0 | Affinidi Credential Issuance Service |
| **OID4VCI Version** | v1.0 (Final) | Draft 13, Final 1, HAIP, Swiyu | Pre-authorized code flow |
| **Credential Formats** | mDoc, SD-JWT-VC | W3C VC, SD-JWT, JSON-LD + BBS+, mDL | W3C VC (JSON-LD) |
| **Signature Algorithms** | Platform-dependent (ES256, ES384) | ES256, EdDSA, BBS+, CRYSTALS-DILITHIUM | EcdsaSecp256k1Signature2019 |
| **DID Methods** | Multiple (issuer-dependent) | Multiple (configurable) | `did:key` |
| **Key Storage** | Android Keystore / Secure Enclave (hardware-only) | Secure Element, Android Keystore, Ubiqu RSE | Cloud-managed (Affinidi Vault) |
| **Transport** | HTTP | HTTP, BLE, MQTT | HTTP |
| **Authorization** | OAuth 2.0 with PAR, DPoP | OAuth 2.0, pre-authorized code | Pre-authorized code with TX_CODE |
| **Selective Disclosure** | SD-JWT-VC, mDoc | SD-JWT, BBS+ | Not natively supported |
| **Post-Quantum Crypto** | No | Yes (CRYSTALS-DILITHIUM) | No |
| **Revocation Support** | Status list (issuer-dependent) | Configurable | Yes (credential status management) |
| **Regulatory Driver** | eIDAS 2.0 (EU mandate) | Standards-flexible (multi-jurisdiction) | Platform/ecosystem |
| **Source Code Available** | Yes (open source) | Yes (open source) | No (public docs and API only) |
| **Analysis Basis** | Source code + documentation | Source code + documentation | Public documentation only |

---

## Why These Three?

These implementations were selected because they represent three distinct architectural philosophies:

1. **EUDI** represents the regulatory-driven, platform-native approach. It answers the question: "What does issuance look like when government mandates dictate the requirements?"

2. **Procivis ONE** represents the standards-flexible, cross-platform approach. It answers the question: "What does issuance look like when maximum interoperability and format support are the priority?"

3. **Affinidi** represents the cloud-managed, developer-accessible approach. It answers the question: "What does issuance look like when the integrator does not want to manage cryptographic infrastructure?"

By comparing all three, this documentation surfaces the trade-offs that any team building an SSI issuance flow will face — regardless of which approach they ultimately choose.

---

**Next**: [Reading Guide](./reading-guide.md) — How to navigate this documentation.
