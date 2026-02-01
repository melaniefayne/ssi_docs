# Key Attestation -- Conceptual Overview

When an issuer creates a credential and binds it to a holder's public key, the issuer is making an implicit assumption: that the corresponding private key is stored securely and cannot be extracted, copied, or misused. Key attestation is the mechanism that makes this assumption explicit and verifiable. It allows the wallet to prove -- cryptographically -- that a key was generated inside secure hardware and that the wallet application itself is genuine.

This document explains the concepts. Platform-specific mechanisms (Android Keystore attestation, iOS Secure Enclave) and OID4VCI protocol integration are covered in [Protocol & Standards](./protocol-and-standards.md).

---

## Why Key Attestation Matters

Without key attestation, the following scenario is possible:

1. A wallet generates a key pair in software (not hardware).
2. The wallet claims the key is hardware-backed.
3. The issuer issues a credential bound to that key.
4. The wallet (or malware on the device) exports the private key.
5. The credential can now be used on any device, by any party, without the holder's knowledge.

This undermines the entire holder binding model. The credential becomes a transferable bearer token rather than a possession-bound artifact. Verifiers who check holder binding are given false assurance.

Key attestation prevents this by providing the issuer with a cryptographic proof chain:

- The key was generated inside a specific piece of secure hardware.
- The key has non-exportable properties (the hardware will not allow extraction).
- The attestation chain traces back to a root of trust (hardware manufacturer or platform vendor).

---

## Wallet Attestation vs. Key Attestation

There are two distinct but related attestation types in the OID4VCI ecosystem. They answer different questions.

### Wallet Attestation

**Question answered:** "Is this wallet application genuine and unmodified?"

Wallet attestation (also called wallet instance attestation) proves that the software making the request is a legitimate, unmodified version of an authorized wallet application. This protects against:

- Modified or repackaged wallet apps that might extract keys or credentials.
- Emulators or debugging environments that bypass platform security.
- Unauthorized third-party applications masquerading as the wallet.

The wallet proves its authenticity to a **wallet provider** (the entity that published the wallet), which in turn issues a wallet attestation JWT. The issuer trusts the wallet provider, so it accepts the wallet attestation as proof that the client is genuine.

### Key Attestation

**Question answered:** "Is this specific cryptographic key stored in secure hardware?"

Key attestation proves that a particular key pair was generated inside a hardware security module (HSM, TEE, Secure Enclave, StrongBox) and has properties that prevent extraction. Unlike wallet attestation (which is about the app), key attestation is about the specific keys that will be used for holder binding.

Key attestation is typically performed per-key or per-key-set. Each time the wallet generates new keys for a credential, it can provide attestation for those specific keys.

### The Relationship

Both attestations are complementary:

| Attestation Type | Scope | Attested By | Presented To |
|-----------------|-------|-------------|--------------|
| Wallet attestation | The wallet app instance | Wallet provider | Issuer |
| Key attestation | Specific key pairs | Platform hardware | Issuer (via wallet) |

An issuer that requires both can be confident that: (a) the requesting application is a genuine wallet, and (b) the keys that will hold the credential are non-extractable.

---

## The Trust Chain

Key attestation operates within a chain of trust that extends from the hardware manufacturer to the credential issuer:

```
Hardware Manufacturer (Google, Apple, etc.)
         |
         | certifies hardware security module
         v
Platform Vendor (Android, iOS)
         |
         | provides attestation APIs
         v
Wallet Provider (EU Commission, Procivis, Affinidi)
         |
         | attests wallet instance
         v
Wallet Application
         |
         | attests specific keys
         v
Credential Issuer
         |
         | trusts attestation chain, issues credential
         v
Credential (bound to attested key)
```

### Trust Decisions at Each Level

- **Hardware manufacturer to platform:** The platform trusts that the hardware security module correctly enforces key non-exportability and provides truthful attestation. This is verified via certificate chains rooted in the manufacturer's root CA.

- **Platform to wallet provider:** The wallet provider registers with the platform (Google Play, Apple App Store) and uses platform-provided attestation APIs (Play Integrity, App Attest) to prove their app's authenticity.

- **Wallet provider to wallet instance:** The wallet provider operates an attestation endpoint. The wallet app authenticates with this endpoint and receives a signed JWT attesting that this specific instance is genuine.

- **Wallet to issuer:** The wallet presents both its wallet attestation and key attestation to the issuer. The issuer validates the attestation chain and, if satisfied, proceeds with credential issuance.

---

## WSCD: Wallet Secure Cryptographic Device

The European Digital Identity framework introduces the concept of a **Wallet Secure Cryptographic Device (WSCD)** -- the secure component responsible for:

- Generating and storing the holder's private keys.
- Performing cryptographic operations (signing) without exposing key material.
- Providing attestation that keys are hardware-protected.

The WSCD is a logical concept that maps to different physical implementations:

| WSCD Implementation | Description | Example |
|---------------------|-------------|---------|
| Platform TEE | Trusted Execution Environment on the device SoC | ARM TrustZone on Android |
| Dedicated SE | Embedded Secure Element chip | StrongBox on Pixel, Secure Enclave on iPhone |
| External SE | Removable or external secure element | Smart cards, USB tokens |
| Remote SE | Cloud-hosted hardware security module | Ubiqu RSE (used by Procivis) |

The EUDI Architecture Reference Framework requires that wallet implementations use a WSCD for holder key storage. The specific WSCD type may vary by member state and deployment context, but the requirement for hardware-backed key storage is a regulatory mandate, not merely a best practice.

---

## What Attestation Does NOT Prove

Key attestation has important limitations:

- **It does not prove the user's identity.** Attestation proves properties of keys and software, not of people. User authentication is handled by the authorization phase (Section 04).

- **It does not prevent device compromise.** If the device OS is fully compromised (rooted/jailbroken), attestation may be bypassable. Platform integrity checks (SafetyNet/Play Integrity, DeviceCheck) mitigate but do not eliminate this risk.

- **It does not guarantee key usage policy.** Attestation proves a key exists in hardware; it does not prove that the wallet will require biometric or PIN authentication before using the key. Usage policies are enforced by the wallet application and the platform's key access controls.

- **It does not create a revocation mechanism.** If a device is lost or compromised after issuance, key attestation provides no way to revoke the credential. Revocation must be handled through separate mechanisms (status lists, accumulators).

---

## The Role of Key Attestation in Issuance

Key attestation integrates into the OID4VCI flow at the proof construction step:

1. The wallet generates a key pair in secure hardware.
2. The wallet obtains key attestation from the platform and/or wallet provider.
3. The wallet constructs a proof-of-possession that includes (or references) the key attestation.
4. The issuer validates the attestation as part of proof validation.
5. If attestation is satisfactory, the issuer binds the credential to the attested public key.

The specific mechanism for including attestation in the proof varies by implementation -- some embed it in the proof JWT, some send it as a separate parameter, and some rely on the wallet provider to vouch for keys through a backend-to-backend channel. These differences are detailed in [Implementation Details](./implementation-details.md).

---

**Next**: [Protocol & Standards](./protocol-and-standards.md) -- JWK representation, platform attestation APIs, and the OID4VCI attestation protocol.
