# iOS Credential Storage

## Secure Enclave

The Secure Enclave is a hardware-isolated coprocessor integrated into Apple's system-on-chip (SoC). It is present on all modern Apple devices (iPhone, iPad, Mac with Apple Silicon). The Secure Enclave has its own boot ROM, AES engine, and protected memory, and operates independently of the application processor.

### Architecture

```
+----------------------------+       +---------------------------+
|   Application Processor    |       |     Secure Enclave        |
|                            |       |                           |
|   iOS / App Code           |       |   Separate Boot ROM       |
|     |                      |       |   Dedicated AES Engine    |
|     +-- Key Gen Request ---+------>|   Key Generation          |
|     |                      |       |   Key Storage (UID-bound) |
|     +-- Sign Request ------+------>|   Signing Operations      |
|     +<-- Signature ---------+------+   Access Policy Enforcem. |
|                            |       |                           |
|   CANNOT access key data   |       |   Keys NEVER leave        |
+----------------------------+       +---------------------------+
```

### Key Properties

| Property | Detail |
|----------|--------|
| Generation | Keys are generated inside the Secure Enclave; the application processor never sees the private key |
| Storage | Keys are encrypted with a device-unique UID key fused into the Secure Enclave at manufacturing |
| Signing | All signing operations execute inside the Secure Enclave |
| Non-extractable | There is no API, no debug interface, and no physical mechanism to extract private keys |
| Biometric gating | Key usage can require Face ID or Touch ID authentication |
| Anti-replay | The Secure Enclave maintains a monotonic counter to prevent replay of old key states |

### Supported Algorithms

The Secure Enclave supports a focused set of algorithms:

- **ECDSA with P-256 (ES256)** -- the primary signing algorithm
- **ECDH with P-256** -- for key agreement
- **AES-256** -- for symmetric encryption (internal use)

Note: The Secure Enclave does not support RSA, Ed25519, or arbitrary curves. Wallets requiring EdDSA or secp256k1 must use software-based key storage on iOS.

---

## Keychain Services

Keychain Services is iOS's encrypted key-value store for sensitive data. Unlike the Secure Enclave (which stores cryptographic keys), the Keychain stores arbitrary secrets: passwords, tokens, PINs, session data.

### Access Control

Keychain items can be protected with multiple access control layers:

| Control | Effect |
|---------|--------|
| `kSecAttrAccessibleWhenUnlockedThisDeviceOnly` | Item accessible only when device is unlocked; not included in backups |
| `kSecAccessControlBiometryCurrentSet` | Requires current biometric enrollment (invalidated if biometrics change) |
| `kSecAccessControlDevicePasscode` | Requires device passcode |
| `kSecAccessControlUserPresence` | Requires either biometric or passcode |

### Keychain vs. Secure Enclave

| Aspect | Keychain | Secure Enclave |
|--------|----------|----------------|
| What it stores | Arbitrary data (strings, blobs) | Cryptographic keys only |
| Encryption | Software-based, key derived from device credentials | Hardware-based, UID-fused key |
| Key extraction | Data can be read by authorized app | Keys can never be read |
| Use case | Passwords, PINs, tokens, certificates | Private key operations (sign, decrypt) |

---

## EUDI iOS Implementation

The EUDI iOS Reference Wallet uses Apple's platform-native storage mechanisms, with SwiftData for metadata and Keychain for sensitive configuration.

### SwiftData Models

EUDI iOS uses SwiftData (Apple's modern persistence framework, successor to Core Data) for credential metadata:

#### SDBookmark

```swift
@Model
class SDBookmark {
    var documentId: String
    var isBookmarked: Bool
    var createdAt: Date
}
```

Tracks which credentials the user has bookmarked for quick access. Mirrors the `Bookmark` Room entity on Android.

#### SDRevokedDocument

```swift
@Model
class SDRevokedDocument {
    var documentId: String
    var revokedAt: Date
    var reason: String?
}
```

Records credentials that have been revoked by the issuer. The wallet checks this table before presentation to prevent use of revoked credentials.

#### SDTransactionLog

```swift
@Model
class SDTransactionLog {
    var transactionId: String
    var documentId: String
    var timestamp: Date
    var verifierName: String?
    var verifierURL: String?
    var outcome: String
}
```

Logs every credential presentation for audit and user review.

### WalletStorage (EudiWalletKit)

The `WalletStorage` class from EudiWalletKit manages the persistence of credential documents:

- Stores issued credentials (mDoc, SD-JWT) in the local data store
- Interfaces with the Secure Enclave for key operations
- Manages the credential pool for one-time-use and rotate-on-use policies
- Handles credential deletion and secure cleanup

### Keychain for PIN Storage

EUDI iOS uses the Keychain for storing the wallet PIN via `KeychainPinStorageProvider`:

```swift
// File: Modules/logic-authentication/Sources/Storage/KeychainPinStorageProvider.swift

final class KeychainPinStorageProvider: PinStorageProvider {

  private let keyChainController: KeyChainController

  init(keyChainController: KeyChainController) {
    self.keyChainController = keyChainController
  }

  func retrievePin() -> String? {
    keyChainController.getValue(key: KeyIdentifier.devicePin)
  }

  func setPin(with pin: String) {
    keyChainController.storeValue(key: KeyIdentifier.devicePin, value: pin)
  }

  func isPinValid(with pin: String) -> Bool {
    keyChainController.getValue(key: KeyIdentifier.devicePin) == pin
  }
}

private enum KeyIdentifier: String, KeyChainWrapper {

  public var value: String {
    self.rawValue
  }

  case devicePin
}
```

The `KeychainPinStorageProvider` delegates all storage operations to a `KeyChainController`, which wraps iOS Keychain Services. The `KeyIdentifier` enum uses `KeyChainWrapper` conformance to provide type-safe key names. The PIN is stored with `kSecAttrAccessibleWhenUnlockedThisDeviceOnly` to ensure it is:
- Encrypted at rest
- Accessible only when the device is unlocked
- Not included in iCloud or iTunes backups

### Credential Policies

EUDI iOS uses the same credential policy model as Android:

| Policy | Configuration | Behavior |
|--------|--------------|----------|
| OneTimeUse | `credentialsCount = 10` (PID default) | Each credential instance is consumed on presentation |
| RotateUse | `credentialsCount = 1` (default) | Credential is replaced after each presentation |

The policy is configured per credential type via `DocumentIssuanceConfig`, identical to the Android implementation.

---

## Procivis iOS Implementation

Procivis uses its cross-platform One Core SDK, with iOS-specific adapters for the Secure Enclave.

### Secure Enclave Integration

- Private keys are generated inside the Secure Enclave
- Signing operations are performed entirely within the enclave
- Key references (not key material) are passed through the One Core SDK abstraction layer

### Platform Abstraction via One Core SDK

```
One Core SDK (Rust, cross-platform)
  |
  +-- Key Storage Abstraction Layer
        |
        +-- iOS: Secure Enclave adapter
        +-- Android: Keystore adapter
        +-- Desktop: Software / HSM adapter
```

The One Core SDK defines a key storage interface. On iOS, this interface is implemented by an adapter that:
1. Maps key generation requests to Secure Enclave API calls
2. Stores key references (not key material) in the SDK's internal storage
3. Routes signing requests to the Secure Enclave
4. Returns signatures to the SDK layer

This abstraction allows Procivis to maintain a single codebase for credential management logic while delegating platform-specific key operations to native code.

### Credential Metadata

Non-key credential data (claims, issuer information, presentation history) is stored in the One Core SDK's internal encrypted database, which is platform-independent.

---

## Affinidi iOS Implementation

Affinidi on iOS follows the same cloud-first model as on Android.

### Affinidi Vault with Multi-Profile Support

Affinidi Vault supports multiple storage profiles:

#### Edge Profile

- Credential data is stored locally on the device
- Key operations may use platform-native mechanisms
- Provides offline access to cached credentials

#### Cloud Profile

- Credentials are stored in the Affinidi Vault cloud service
- Accessible from any authenticated device
- The primary storage model for Affinidi's architecture

### Multi-Profile Architecture

```
Affinidi Vault
  |
  +-- Edge Profile
  |     +-- Local storage on device
  |     +-- Cached credentials
  |     +-- Limited offline capability
  |
  +-- Cloud Profile
        +-- Cloud-hosted encrypted storage
        +-- Multi-device access
        +-- Full credential management
```

Users can switch between profiles or use both simultaneously. The Edge profile provides a local cache for performance and limited offline scenarios, while the Cloud profile is the authoritative store.

### Implications for iOS

| Aspect | Implication |
|--------|-------------|
| Secure Enclave usage | Minimal (not the primary key storage mechanism) |
| Keychain usage | May be used for authentication tokens to the Vault |
| iCloud Keychain | Not used (credentials are in Affinidi's cloud, not Apple's) |
| Device binding | Weak (credentials are cloud-portable) |

---

## iOS Storage Comparison

| Feature | EUDI | Procivis | Affinidi |
|---------|------|----------|----------|
| Key storage | Secure Enclave | Secure Enclave (via One Core) | Cloud Vault |
| Credential metadata | SwiftData | One Core encrypted DB | Affinidi Vault |
| PIN storage | Keychain (KeychainPinStorageProvider) | Platform-dependent | Cloud auth |
| Hardware-backed keys | Yes | Yes | No (cloud-managed) |
| Offline capability | Full | Full (local keys) | Limited (Edge profile) |
| Credential policies | OneTimeUse, RotateUse | Configurable | N/A |
| Revocation tracking | SDRevokedDocument | One Core DB | Cloud-managed |
| Transaction logging | SDTransactionLog | One Core DB | Cloud-managed |
| Multi-device | No (device-bound keys) | No (local) / Yes (RSE) | Yes (Cloud profile) |
| Backup included | No (device-only Keychain) | No | Yes (cloud-native) |
