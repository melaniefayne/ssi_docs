# Secure Storage: Conceptual Overview

## Why Secure Storage Matters

Verifiable credentials and the cryptographic keys that control them are **bearer tokens**. Whoever possesses the key material can present the credential and assert the associated identity claims. There is no server-side session, no password check at presentation time, and no out-of-band confirmation. If an attacker extracts the private key from a wallet, they can impersonate the holder indefinitely, and the holder may never know.

This makes secure storage the single most critical component of wallet security. Every other protection mechanism -- DPoP, key binding, wallet attestation -- assumes that the underlying keys cannot be extracted or copied.

## Hardware-Backed Storage

Hardware-backed storage means that cryptographic keys are generated inside, and never leave, a dedicated security processor. The application processor (where the operating system and apps run) can request signing operations, but it never sees the raw key material.

### Why Software-Only Storage Is Insufficient

A private key stored in application memory, a file, or even an encrypted database is ultimately protected only by software. An attacker with root access, a memory dump, or a compromised OS update can extract it. Software encryption adds a layer, but the encryption key itself must be stored somewhere accessible to the CPU.

### Hardware Isolation Model

Hardware-backed storage solves this by placing key material in a physically separate processor:

```
+---------------------------+       +------------------------+
|   Application Processor   |       |   Security Processor   |
|                           |       |                        |
|  Wallet App               |       |  Key Generation        |
|    |                      |       |  Key Storage            |
|    +-- Sign Request ------+------>|  Signing Operations    |
|    |                      |       |  Access Policy Enforce. |
|    +<-- Signature --------+-------+                        |
|                           |       |  Keys NEVER leave      |
+---------------------------+       +------------------------+
```

The security processor enforces access policies (biometric, passcode) and rate limits. Even a fully compromised application processor cannot extract the keys.

### Platform Implementations

| Platform | Security Processor | Characteristics |
|----------|-------------------|-----------------|
| Android  | TEE (Trusted Execution Environment) | Isolated execution environment on main SoC. Available on most devices. |
| Android  | Strongbox | Discrete secure element chip. Higher assurance than TEE. Available on newer devices. |
| iOS      | Secure Enclave | Hardware-isolated coprocessor on Apple SoC. Available on all modern Apple devices. |

## Encryption at Rest

Beyond key storage, credential metadata (claims, issuance dates, issuer information) must also be protected. Encryption at rest ensures that credential data stored in databases or files is unreadable without the appropriate decryption key, which itself should be hardware-protected.

Layers of encryption at rest:

1. **Full-disk encryption** -- provided by the operating system (Android FBE, iOS Data Protection)
2. **Database encryption** -- application-level encryption of the credential store
3. **Field-level encryption** -- selective encryption of sensitive claim values
4. **Hardware-bound decryption keys** -- the key that decrypts the database is stored in hardware

## Credential Lifecycle

A credential passes through a defined lifecycle. Secure storage must support every phase:

```
Issuance --> Storage --> Use --> Rotation/Revocation --> Deletion
```

### Issuance

The wallet receives a credential from an issuer. The credential is bound to a key pair controlled by the wallet. The private key is generated in hardware; the public key is sent to the issuer as part of the proof of possession. The issued credential is stored alongside metadata (issuer, issuance date, expiry, format).

### Storage

The credential and its metadata are persisted in an encrypted store. The binding key remains in hardware. The wallet maintains an index of stored credentials for discovery and presentation.

### Use (Presentation)

When presenting a credential to a verifier, the wallet signs a presentation proof using the hardware-backed key. The credential data is read from storage, selectively disclosed claims are computed (for SD-JWT or mDoc), and the signed presentation is transmitted.

### Rotation

Some credential policies require rotation after use. The wallet requests a fresh credential from the issuer, stores the new credential, and marks or deletes the old one. Rotation limits the window of exposure if a credential is intercepted.

### Revocation

The issuer may revoke a credential (e.g., a driver's license is suspended). The wallet should track revocation status and prevent presentation of revoked credentials. Some implementations store revocation status locally; others check a revocation registry.

### Deletion

When a credential is no longer needed or has been revoked, the wallet must securely delete both the credential data and the associated key material. Secure deletion means overwriting storage, not just removing file system references.

## Credential Policies

Credential policies define how credentials are consumed during presentation. The two primary policy types are:

### One-Time Use

A one-time-use credential is valid for a single presentation. After the wallet presents it to a verifier, the credential is consumed and cannot be used again. The wallet must pre-provision multiple copies to avoid running out.

**Characteristics:**
- Maximum privacy: each presentation uses a distinct credential, preventing cross-verifier correlation
- Higher issuance overhead: the wallet must maintain a pool of unused credentials
- Requires batch issuance support
- Suited for high-sensitivity credentials (e.g., government-issued PID)

### Rotate on Use

A rotate-on-use credential can be used once, but the wallet automatically requests a replacement after each presentation. The old credential is invalidated and a new one is issued.

**Characteristics:**
- Balances privacy and operational simplicity
- Requires the issuer to support efficient re-issuance
- The wallet manages the rotation transparently
- Suited for general-purpose credentials

### Policy Configuration

The choice of policy is typically configured per credential type:

| Credential Type | Recommended Policy | Pool Size | Rationale |
|----------------|-------------------|-----------|-----------|
| PID (Person Identification Data) | One-Time Use | 10+ | Maximum unlinkability for government ID |
| mDL (Mobile Driving License) | Rotate on Use | 1 | Frequent use; rotation balances privacy and overhead |
| Age Verification | One-Time Use | 5+ | Sensitive attribute; prevent tracking |
| Membership Card | Rotate on Use | 1 | Lower sensitivity; operational simplicity |

## Summary

Secure storage is not a single feature but a layered architecture:

1. **Hardware isolation** prevents key extraction
2. **Encryption at rest** protects credential data
3. **Access control** (biometric, passcode) gates usage
4. **Credential policies** limit exposure per presentation
5. **Lifecycle management** ensures credentials are rotated, revoked, and deleted appropriately

The following pages detail how each platform (Android, iOS) and each implementation (EUDI, Procivis, Affinidi) realize these principles.
