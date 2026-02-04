# Key Attestation -- Comparative Analysis

This document compares how the four wallet implementations handle key attestation and wallet attestation. The implementations span a wide spectrum: from explicit attestation endpoints with hardware-backed JWK exchange (EUDI) to multi-tier key storage with remote secure elements (Procivis ONE) to DID-based configuration-driven trust (Affinidi).

---

## Feature Comparison Matrix

| Feature | EUDI Android | EUDI iOS | Procivis ONE | Affinidi |
|---------|-------------|----------|--------------|----------|
| **Wallet attestation** | Yes (dedicated endpoint) | Yes (dedicated endpoint) | N/A (no public endpoint) | N/A |
| **Key attestation** | Yes (dedicated endpoint) | Yes (dedicated endpoint) | Per-tier (core engine) | DID-based signing |
| **Hardware-backed keys** | TEE / StrongBox | Secure Enclave | Platform SE (optional) | Not documented |
| **Remote secure element** | No | No | Yes (Ubiqu RSE) | No |
| **Software-only keys** | Fallback if no hardware | Not supported | Yes (INTERNAL tier) | Not documented |
| **Attestation format** | JWK --> JWT | JWK --> JWT | Tier-dependent | DID key proof |
| **Nonce binding** | Yes (explicit parameter) | Yes (explicit parameter) | Core engine handles | Session-level |
| **Trust chain** | HW --> platform --> wallet provider --> issuer | HW --> platform --> wallet provider --> issuer | Varies by tier | Affinidi platform --> issuer |

---

## Attestation Endpoint Architecture

### EUDI (Android & iOS)

Both EUDI implementations use a two-endpoint attestation model, communicating with a wallet provider backend:

```
                Wallet App                    Wallet Provider Backend
                    |                                  |
    Wallet          |   POST /wallet-instance-         |
    Attestation     |     attestation/jwk              |
                    |  ------------------------------>  |
                    |                                  |
                    |   Wallet Attestation JWT          |
                    |  <------------------------------  |
                    |                                  |
    Key             |   POST /wallet-unit-             |
    Attestation     |     attestation/jwk-set          |
                    |  ------------------------------>  |
                    |                                  |
                    |   Key Attestation JWT             |
                    |  <------------------------------  |
                    |                                  |
```

This architecture separates wallet-level trust from key-level trust, allowing the issuer to verify both independently. The wallet provider acts as a trusted intermediary that vouches for both the wallet instance and the keys it generates.

### Procivis ONE

Procivis ONE does not use external attestation endpoints. Instead, attestation is implicit in the key storage tier:

```
                Wallet App                    Core Engine
                    |                             |
                    |  Key generation request      |
                    |  (specifying security tier)  |
                    |  --------------------------> |
                    |                             |
                    |      INTERNAL: encrypted DB  |
                    |      SECURE_ELEMENT: platform |
                    |      UBIQU_RSE: remote HSM   |
                    |                             |
                    |  Key generated at tier       |
                    |  <-------------------------- |
                    |                             |
```

The trust model is different: the issuer trusts the Procivis platform's declaration of key storage tier, rather than independently verifying hardware attestation through a certificate chain.

### Affinidi

Affinidi uses DID-based key management without explicit attestation endpoints:

```
                Vault App                     Issuance Service
                    |                                |
                    |  Proof signed with DID key      |
                    |  -----------------------------> |
                    |                                |
                    |  Service validates DID and      |
                    |  proof signature                |
                    |                                |
```

The trust anchor is the Affinidi platform itself. The issuance service trusts that the Vault correctly manages keys, but there is no independently verifiable hardware attestation.

---

## Security Levels

### Hardware Security Comparison

| Level | EUDI Android | EUDI iOS | Procivis ONE | Affinidi |
|-------|-------------|----------|--------------|----------|
| **Highest** | StrongBox (dedicated SE chip) | Secure Enclave (dedicated coprocessor) | UBIQU_RSE (cloud HSM, certified) | Not documented |
| **Medium** | TEE (ARM TrustZone) | -- (no intermediate; SE or nothing) | SECURE_ELEMENT (platform SE) | Not documented |
| **Lowest** | Software fallback (if no hardware) | -- (SE required) | INTERNAL (encrypted DB) | DID key (storage unspecified) |

### Security Level Selection

| Implementation | How Level Is Determined |
|---------------|----------------------|
| EUDI Android | Automatic -- StrongBox preferred, TEE fallback, software last resort |
| EUDI iOS | Mandatory -- Secure Enclave required for all credential keys |
| Procivis ONE | Configuration-driven -- credential schema specifies required `KeyStorageSecurityBindingEnum` |
| Affinidi | Configuration-driven -- `claimMode` determines holder verification method |

The EUDI iOS approach is the strictest: every credential key must be Secure Enclave-backed. There is no software fallback. If the device does not have a Secure Enclave (which would mean a very old device), credential issuance is not possible.

The EUDI Android approach is more permissive, accepting TEE-level keys on devices without StrongBox. Software keys are a last resort and may not satisfy issuer requirements.

Procivis ONE is the most flexible, offering three explicit tiers that the credential schema can require. This allows the same wallet to issue low-security credentials (INTERNAL) and high-security credentials (UBIQU_RSE) on the same device.

---

## Hardware Requirements

| Implementation | Minimum Hardware | Recommended Hardware | Fallback |
|---------------|-----------------|---------------------|----------|
| EUDI Android | TEE (most modern Android devices) | StrongBox (Pixel 3+, Samsung S20+) | Software keystore |
| EUDI iOS | Secure Enclave (iPhone 5s+, iPad Air 2+) | Same | None |
| Procivis ONE | None (INTERNAL works without hardware SE) | Platform SE or RSE | INTERNAL (software) |
| Affinidi | None documented | None documented | N/A |

---

## Trust Chain Models

### EUDI Trust Chain

```
Google/Apple Root CA
       |
       | Hardware attestation certificate chain
       v
Device Hardware (TEE / StrongBox / Secure Enclave)
       |
       | Platform attestation APIs
       v
Android / iOS Platform
       |
       | Wallet attestation request
       v
EU Wallet Provider Backend
       |
       | Wallet attestation JWT + Key attestation JWT
       v
Credential Issuer
       |
       | Validates attestation chain
       v
Issues credential bound to attested key
```

**Trust decisions:**
1. The issuer trusts the wallet provider (via a trust registry).
2. The wallet provider trusts the platform (via Play Integrity / App Attest).
3. The platform trusts the hardware (via manufacturer certificate chains).

### Procivis ONE Trust Chain

```
For SECURE_ELEMENT tier:
    Device Hardware (Platform SE)
           |
           v
    Procivis Core Engine
           |
           v
    Credential Issuer

For UBIQU_RSE tier:
    Ubiqu Cloud HSM (FIPS/CC certified)
           |
           v
    Ubiqu SDK (PIN/biometric auth)
           |
           v
    Procivis Core Engine
           |
           v
    Credential Issuer

For INTERNAL tier:
    Procivis Core Engine
           |
           | (no hardware trust chain)
           v
    Credential Issuer
```

**Trust decisions:**
1. For RSE: the issuer trusts Ubiqu's HSM certification and Procivis's integration.
2. For SE: the issuer trusts the platform's hardware security.
3. For INTERNAL: the issuer trusts Procivis's software encryption.

### Affinidi Trust Chain

```
Affinidi Platform
       |
       | DID-based identity
       v
Affinidi Vault
       |
       | Proof signed with DID key
       v
Credential Issuer
       |
       | Validates DID and proof
       v
Issues credential bound to DID
```

**Trust decisions:**
1. The issuer trusts the Affinidi platform.
2. DID-based proof validates key possession but not storage properties.

---

## Attestation Flow Diagrams

### EUDI Complete Attestation Flow

```
Wallet App                Wallet Provider           Issuer
    |                          |                      |
    |  Generate key in SE      |                      |
    |  (hardware)              |                      |
    |                          |                      |
    |  POST /wallet-instance-attestation/jwk          |
    |  --------------------------->                    |
    |                          |                      |
    |  Wallet Attestation JWT  |                      |
    |  <---------------------------                   |
    |                          |                      |
    |  POST /wallet-unit-attestation/jwk-set          |
    |  (keys + c_nonce)        |                      |
    |  --------------------------->                    |
    |                          |                      |
    |  Key Attestation JWT     |                      |
    |  <---------------------------                   |
    |                          |                      |
    |  Credential Request                             |
    |  (proof + wallet attestation + key attestation) |
    |  ------------------------------------------------>
    |                          |                      |
    |  Issuer validates:                              |
    |  1. Wallet attestation JWT signature            |
    |  2. Key attestation JWT signature               |
    |  3. Proof JWT signature matches attested key    |
    |  4. Nonce in proof matches c_nonce              |
    |                          |                      |
    |  Credential Response                            |
    |  <------------------------------------------------
    |                          |                      |
```

### Procivis RSE Attestation Flow

```
Wallet App           Core Engine          Ubiqu SDK         Ubiqu HSM
    |                     |                   |                  |
    |  Issue credential   |                   |                  |
    |  (HIGH security)    |                   |                  |
    |  -----------------> |                   |                  |
    |                     |                   |                  |
    |                     |  Create key pair  |                  |
    |                     |  ----------------> |                  |
    |                     |                   |  Generate in HSM |
    |                     |                   |  --------------> |
    |                     |                   |                  |
    |                     |                   |  Public key      |
    |                     |                   |  <-------------- |
    |                     |  Public key       |                  |
    |                     |  <---------------- |                  |
    |                     |                   |                  |
    |  PIN required       |                   |                  |
    |  <----------------- |                   |                  |
    |  Enter PIN          |                   |                  |
    |  -----------------> |                   |                  |
    |                     |  Sign proof       |                  |
    |                     |  (with PIN auth)  |                  |
    |                     |  ----------------> |                  |
    |                     |                   |  Sign in HSM     |
    |                     |                   |  --------------> |
    |                     |                   |                  |
    |                     |                   |  Signature       |
    |                     |                   |  <-------------- |
    |                     |  Signature        |                  |
    |                     |  <---------------- |                  |
    |                     |                   |                  |
    |                     |  Send credential  |                  |
    |                     |  request to issuer|                  |
    |                     |                   |                  |
```

---

## Strengths and Weaknesses

### EUDI (Android & iOS)

**Strengths:**
- Most complete attestation model: separate wallet and key attestation with independent verification.
- Explicit nonce binding in key attestation prevents session confusion attacks.
- Hardware-backed by default, with StrongBox/Secure Enclave as the primary target.
- Trust chain extends from hardware manufacturer through platform to wallet provider, providing multiple layers of verification.
- Aligns with EUDI Architecture Reference Framework requirements for WSCD.

**Weaknesses:**
- Depends on wallet provider backend availability for attestation. If the backend is down, attestation fails and issuance is blocked.
- Android's hardware security varies significantly across devices. StrongBox is not universally available, and TEE quality varies by manufacturer.
- iOS restricts to P-256 keys only (Secure Enclave limitation), which may not satisfy all use cases.
- No remote secure element option. Devices without adequate local hardware cannot achieve the highest security levels.

### Procivis ONE

**Strengths:**
- Three-tier key storage provides maximum flexibility. Issuers can require different security levels for different credential types.
- Remote Secure Element (Ubiqu RSE) enables hardware-grade security on any device, regardless of local hardware capabilities.
- RSE's FIPS/CC certification may satisfy regulatory requirements that device TEE cannot.
- Explicit security level enumeration (`INTERNAL`, `SECURE_ELEMENT`, `UBIQU_RSE`) makes the security posture visible and configurable.

**Weaknesses:**
- No documented external attestation endpoints. The issuer trusts Procivis's declaration of security tier rather than independently verifying hardware attestation.
- RSE introduces network dependency for every signing operation, affecting offline usability.
- INTERNAL tier provides no hardware protection. Software-encrypted keys are vulnerable to device compromise.
- The trust model for RSE depends on the Ubiqu provider, adding a third-party dependency.

### Affinidi

**Strengths:**
- DID-based key management is simple and interoperable. Any DID method can be used.
- No hardware requirements lower the barrier to entry.
- Configuration-driven approach simplifies deployment.

**Weaknesses:**
- No visible hardware key attestation. The issuer cannot verify that keys are non-extractable.
- The trust model depends entirely on the Affinidi platform. There is no independent hardware verification.
- DID-based proof validates key possession but not key storage properties. A software-stored key and a hardware-stored key are indistinguishable from the issuer's perspective.
- May not satisfy regulatory requirements (e.g., eIDAS 2.0) that mandate hardware-backed key storage with verifiable attestation.

---

## Decision Framework

| If your scenario requires... | Consider... |
|------------------------------|------------|
| Full hardware attestation with certificate chains | EUDI (Android or iOS) |
| Regulatory compliance (eIDAS 2.0, WSCD) | EUDI |
| Maximum flexibility in security levels | Procivis ONE (three tiers) |
| Hardware security on devices without SE | Procivis ONE (Ubiqu RSE) |
| Certified HSM for key storage | Procivis ONE (Ubiqu RSE with FIPS/CC) |
| Simplest implementation (no attestation) | Affinidi |
| DID-based identity model | Affinidi |
| Offline signing capability | EUDI or Procivis (INTERNAL/SE tiers) |
| Independent key attestation verification | EUDI (attestation endpoints with JWK) |

---

**Previous**: [Implementation Details](./implementation-details.md)
**Section index**: [README](./README.md)
