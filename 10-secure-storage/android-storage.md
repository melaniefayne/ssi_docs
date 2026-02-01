# Android Credential Storage

## Android Keystore System

The Android Keystore system provides hardware-backed cryptographic key storage. Keys generated within the Keystore are bound to the device's security hardware and cannot be extracted -- not by the application, not by the operating system, and not by an attacker with root access.

### TEE (Trusted Execution Environment)

The TEE is an isolated execution environment that runs alongside the main Android operating system on the same system-on-chip (SoC). It has its own processor context, memory space, and storage. The main OS communicates with the TEE through a narrow, well-defined interface.

**Capabilities:**
- Key generation and storage
- Signing and encryption operations
- Access policy enforcement (biometric, timeout, authentication)
- Attestation of key properties (the TEE can prove a key is hardware-backed)

**Limitations:**
- Shares the SoC with the application processor (side-channel attacks are theoretically possible, though mitigated)
- Implementation quality varies by device manufacturer

### Strongbox (Discrete Secure Element)

Strongbox is a physically separate secure element chip, distinct from the main SoC. It provides a higher level of assurance than TEE because it has independent power, clock, and memory, making it resistant to side-channel and fault-injection attacks.

**Capabilities:**
- All TEE capabilities, plus:
- Resistance to physical tampering
- Independent tamper-detection circuitry
- Certified to higher assurance levels (e.g., Common Criteria EAL)

**Availability:**
- Required on devices that ship with Android 9+ and include the hardware
- Not available on all devices (the wallet must fall back to TEE if Strongbox is absent)

### Key Properties

Keys generated in Android Keystore can be configured with the following properties:

| Property | Description |
|----------|-------------|
| Non-extractable | The key material never leaves the security hardware |
| Hardware-backed | The key is stored in TEE or Strongbox, not software |
| Biometric-protected | Each use requires biometric authentication |
| Time-bounded | The key is valid only for a configured duration after authentication |
| Purpose-restricted | The key can only be used for specified operations (sign, encrypt, etc.) |

```
Android Keystore Architecture:

+------------------+     +------------------+     +------------------+
|   Wallet App     |     |   Android OS     |     |   TEE/Strongbox  |
|                  |     |                  |     |                  |
|  KeyGeneration   +---->|  Keystore API    +---->|  Key Generated   |
|  Request         |     |  (KeyMaster HAL) |     |  Key Stored      |
|                  |     |                  |     |                  |
|  Sign Request    +---->|  Keystore API    +---->|  Signing Op      |
|                  |     |                  |     |  (inside HW)     |
|  Signature <-----+-----+  Signature <-----+-----+  Returns sig    |
+------------------+     +------------------+     +------------------+
```

---

## EUDI Android Implementation

The EUDI Android Reference Wallet stores credential metadata in a local database and delegates key operations to the Android Keystore.

### Room Database (v2.8.4)

EUDI uses Android Room (version 2.8.4) as its local persistence layer for credential metadata. Room provides compile-time verification of SQL queries and a typed abstraction over SQLite.

#### Database Tables

| Table | Purpose | Key Fields |
|-------|---------|------------|
| `Bookmark` | Tracks bookmarked/pinned credentials for quick access | Document ID, bookmark status |
| `RevokedDocument` | Records credentials that have been revoked by their issuer | Document ID, revocation timestamp, reason |
| `TransactionLog` | Logs every credential presentation for audit and debugging | Transaction ID, timestamp, verifier, credential used, outcome |

#### Entity Relationships

```
Bookmark
  +-- documentId (references stored credential)

RevokedDocument
  +-- documentId (references revoked credential)
  +-- revokedAt (timestamp)

TransactionLog
  +-- transactionId (unique)
  +-- documentId (references credential)
  +-- timestamp
  +-- verifierInfo
```

### Credential Policies via DocumentIssuanceConfig

EUDI configures credential behavior through `DocumentIssuanceConfig`, which includes a `CredentialPolicy` enum:

#### CredentialPolicy.OneTimeUse

```
OneTimeUse(credentialsCount: Int)
```

- Each credential instance is used for exactly one presentation
- The wallet pre-provisions `credentialsCount` instances during issuance
- After presentation, the used instance is consumed and cannot be reused
- When the pool is depleted, the wallet must re-issue

**Default configuration for PID:** `OneTimeUse(credentialsCount = 10)`

This means 10 credential instances are provisioned at issuance, each valid for a single presentation.

#### CredentialPolicy.RotateUse

```
RotateUse(credentialsCount: Int)
```

- The credential is used once, then the wallet automatically requests a fresh instance
- `credentialsCount` specifies how many instances to maintain in the pool (typically 1)
- The old credential is replaced after each presentation

**Default for most credential types:** `RotateUse(credentialsCount = 1)`

### WalletCoreConfig

The `WalletCoreConfig` class configures the secure areas available to the wallet:

```
WalletCoreConfig:
  +-- secureAreas: List<SecureArea>
  |     +-- AndroidKeystoreSecureArea (TEE or Strongbox)
  |     +-- SoftwareSecureArea (fallback, development only)
  +-- documentIssuanceConfig: DocumentIssuanceConfig
        +-- credentialPolicy: CredentialPolicy
```

The wallet preferentially selects hardware-backed secure areas. Software-backed areas exist only for testing and development and must not be used in production deployments.

---

## Procivis Android Implementation

Procivis uses a cross-platform core SDK (One Core) with platform-specific key storage adapters.

### Android Keystore via SECURE_ELEMENT Configuration

Procivis accesses the Android Keystore through a `SECURE_ELEMENT` key storage type, configured with an `aliasPrefix` to namespace keys:

```
Key Storage Configuration:
  type: SECURE_ELEMENT
  params:
    aliasPrefix: "com.procivis.one."
```

**Key properties:**
- Keys are generated in TEE or Strongbox (depending on device capability)
- The `aliasPrefix` ensures Procivis keys do not collide with keys from other applications
- Key operations (signing, key agreement) are performed inside the secure hardware
- Keys are non-extractable

### INTERNAL Encrypted Database

Credential metadata and non-key data are stored in an encrypted local database:

- The database is encrypted using a key derived from the device's hardware-backed key
- The encryption is transparent to the application layer
- The database stores credential payloads, issuer metadata, presentation history, and configuration

### UBIQU_RSE (Remote Secure Element)

For enterprise deployments where keys must be managed centrally, Procivis supports a Remote Secure Element (RSE) via the UBIQU service:

- Cryptographic keys are generated and stored on a remote HSM (Hardware Security Module)
- Signing operations are performed remotely
- The wallet communicates with the RSE over an authenticated, encrypted channel
- This model supports organizational control over keys while still enabling mobile wallet UX

**Trade-offs of RSE:**
| Dimension | Local (SECURE_ELEMENT) | Remote (UBIQU_RSE) |
|-----------|----------------------|---------------------|
| Latency | Microseconds | Network round-trip |
| Availability | Always (offline) | Requires network |
| Key control | User-controlled | Organization-controlled |
| Assurance level | Device-dependent | HSM-certified |

---

## Affinidi Android Implementation

Affinidi takes a fundamentally different approach: credentials and keys are stored in a cloud-based Vault rather than on the device's hardware.

### Affinidi Vault

- Credential data is stored in the Affinidi Vault, a cloud-hosted encrypted storage service
- The Vault is accessed via authenticated API calls
- The device does not directly use the Android Keystore for credential key storage

### Implications

| Aspect | Implication |
|--------|-------------|
| Offline access | Not supported without cached credentials |
| Key extraction risk | Shifts from device to cloud infrastructure |
| Multi-device access | Natively supported (credentials accessible from any authenticated device) |
| Hardware attestation | Not applicable (keys are not hardware-backed on device) |
| Regulatory compliance | May not satisfy eIDAS 2.0 WSCD requirements |

This model trades hardware-backed device security for cloud convenience and multi-device accessibility. It is well-suited for issuer-focused use cases where the credential holder is consuming a service, but may not meet the security requirements for government-issued identity credentials.

---

## Android Storage Comparison

| Feature | EUDI | Procivis | Affinidi |
|---------|------|----------|----------|
| Key storage location | Android Keystore (TEE/Strongbox) | Android Keystore (TEE/Strongbox) or RSE | Cloud Vault |
| Credential metadata store | Room Database | Encrypted local DB | Cloud Vault |
| Hardware-backed keys | Yes | Yes (SECURE_ELEMENT) | No |
| Offline key operations | Yes | Yes (local) / No (RSE) | No |
| Credential policies | OneTimeUse, RotateUse | Configurable | N/A |
| Revocation tracking | RevokedDocument table | Local DB | Cloud-managed |
| Transaction logging | TransactionLog table | Local DB | Cloud-managed |
| Multi-device support | No (keys are device-bound) | No (local) / Yes (RSE) | Yes |
