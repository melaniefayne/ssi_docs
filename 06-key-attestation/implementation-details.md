# Key Attestation -- Implementation Details

This document describes how each wallet implementation handles key attestation and wallet attestation. The implementations span a wide range of approaches: from dedicated attestation endpoints with JWK exchange (EUDI) to multi-tier key storage with remote secure elements (Procivis ONE) to configuration-driven DID-based signing (Affinidi).

---

## 1. EUDI Android

### Attestation Provider

The EUDI Android wallet implements key attestation through `WalletCoreAttestationProvider`, which exposes two methods:

```kotlin
// File: core-logic/src/main/java/eu/europa/ec/corelogic/provider/WalletCoreAttestationProvider.kt

interface WalletCoreAttestationProvider : WalletAttestationsProvider

class WalletCoreAttestationProviderImpl(
    private val walletCoreConfig: WalletCoreConfig,
    private val walletAttestationRepository: WalletAttestationRepository
) : WalletCoreAttestationProvider {

    override suspend fun getWalletAttestation(
        keyInfo: KeyInfo
    ): Result<String> = walletAttestationRepository.getWalletAttestation(
        baseUrl = walletCoreConfig.walletProviderHost,
        keyInfo = keyInfo.publicKey.toJwk()
    )

    override suspend fun getKeyAttestation(
        keys: List<KeyInfo>,
        nonce: Nonce?
    ): Result<String> = walletAttestationRepository.getKeyAttestation(
        baseUrl = walletCoreConfig.walletProviderHost,
        keys = keys.map { it.publicKey.toJwk() },
        nonce = nonce?.value
    )
}
```

| Method | Purpose | Input | Output |
|--------|---------|-------|--------|
| `getWalletAttestation` | Prove the wallet instance is genuine | Key information for the wallet instance | Wallet attestation JWT |
| `getKeyAttestation` | Prove specific keys are hardware-backed | List of keys + session nonce | Key attestation JWT |

### Attestation Endpoints

The attestation provider communicates with the wallet provider's backend through `WalletAttestationRepository`, which makes HTTP POST requests to two endpoints:

| Endpoint | Method | Request Body | Response |
|----------|--------|-------------|----------|
| `/wallet-instance-attestation/jwk` | POST | JWK (public key of the wallet instance) | Wallet attestation JWT |
| `/wallet-unit-attestation/jwk-set` | POST | JWK Set (public keys to be attested) | Key attestation JWT |

**Wallet instance attestation flow:**

```
Wallet App                      Wallet Provider Backend
    |                                    |
    |  POST /wallet-instance-attestation/jwk
    |  Body: { "kty": "EC", "crv": "P-256", "x": "...", "y": "..." }
    |  --------------------------------> |
    |                                    |
    |  Wallet Attestation JWT            |
    |  (signed by wallet provider)       |
    |  <-------------------------------- |
    |                                    |
```

**Key attestation flow:**

```
Wallet App                      Wallet Provider Backend
    |                                    |
    |  POST /wallet-unit-attestation/jwk-set
    |  Body: {
    |    "keys": [
    |      { "kty": "EC", "crv": "P-256", "x": "...", "y": "..." },
    |      { "kty": "EC", "crv": "P-256", "x": "...", "y": "..." }
    |    ]
    |  }
    |  --------------------------------> |
    |                                    |
    |  Key Attestation JWT               |
    |  (signed by wallet provider,       |
    |   includes nonce binding)          |
    |  <-------------------------------- |
    |                                    |
```

### Configuration

The wallet provider host is configured through `WalletCoreConfig`:

```kotlin
// WalletCoreConfig -- simplified
data class WalletCoreConfig(
    val walletProviderHost: String,  // e.g., "https://wallet-provider.eudiw.dev"
    // Other configuration...
)
```

The full attestation endpoint URLs are constructed by appending the path to the `walletProviderHost`.

### Hardware-Backed Keys

Keys are generated using Android's `SecureArea` API from the wallet-core SDK, which wraps the Android Keystore:

```kotlin
// Conceptual -- key generation
val secureArea = AndroidKeystoreSecureArea(context)
val keyAlias = "holder-key-${UUID.randomUUID()}"

secureArea.createKey(
    alias = keyAlias,
    createKeySettings = AndroidKeystoreCreateKeySettings.Builder(
        challenge = nonce.toByteArray()
    )
    .setKeyPurposes(setOf(KeyPurpose.SIGN))
    .setEcCurve(EcCurve.P256)
    .setUserAuthenticationRequired(true)
    .setStrongBoxBacked(true)  // Use StrongBox if available
    .build()
)
```

When StrongBox is available on the device, keys are generated in the dedicated secure element. When StrongBox is not available, keys fall back to the TEE (Trusted Execution Environment). The `SecureArea` abstraction reports the actual security level achieved, which is included in the attestation.

### JWK Serialization

Public keys are serialized to JWK format before being sent to the attestation endpoints. The SDK handles the conversion from Android `KeyStore.Entry` to JWK using internal serialization logic.

---

## 2. EUDI iOS

### Attestation Provider

The iOS implementation uses `WalletKitAttestationProvider` with the same two-method pattern:

```swift
// File: Modules/logic-core/Sources/Provider/WalletKitAttestationProvider.swift

protocol WalletKitAttestationProvider: WalletAttestationsProvider {
  var baseUrl: String { get }
  func getWalletAttestation(key: any JOSESwift.JWK) async throws -> String
  func getKeysAttestation(keys: [any JOSESwift.JWK], nonce: String?) async throws -> String
}

final class WalletKitAttestationProviderImpl: WalletKitAttestationProvider {

  let repository: WalletAttestationRepository
  let baseUrl: String

  init(with repository: WalletAttestationRepository, and configLogic: WalletProviderAttestationConfig) {
    self.repository = repository
    self.baseUrl = configLogic.walletProviderAttestationUrl
  }

  func getWalletAttestation(key: any JOSESwift.JWK) async throws -> String {
    let jwkDict = try key.toDictionary()
    let payload = ["jwk": jwkDict]
    let encodedPayload = try JSONSerialization.data(withJSONObject: payload, options: [])
    let response = try await repository.issueWalletInstanceAttestation(host: self.baseUrl, payload: encodedPayload)
    return response.walletInstanceAttestation
  }

  func getKeysAttestation(keys: [any JOSESwift.JWK], nonce: String?) async throws -> String {
    let jwkDict = try keys.map { try $0.toDictionary() }
    var payload: [String: Any] = [
      "jwkSet": ["keys": jwkDict]
    ]
    if let nonce {
      payload["nonce"] = nonce
    }
    let encodedPayload = try JSONSerialization.data(withJSONObject: payload, options: [])
    let response = try await repository.issueWalletUnitAttestation(host: self.baseUrl, payload: encodedPayload)
    return response.walletUnitAttestation
  }
}
```

| Method | Purpose | Input | Output |
|--------|---------|-------|--------|
| `getWalletAttestation(key:)` | Prove wallet instance authenticity | Wallet instance public key (`SecKey`) | Wallet attestation JWT |
| `getKeysAttestation(keys:nonce:)` | Prove keys are Secure Enclave-backed | Array of public keys + session nonce | Key attestation JWT |

### Attestation Endpoints

The iOS wallet communicates with the same wallet provider endpoints as Android:

| Endpoint | Method | Request Body | Response |
|----------|--------|-------------|----------|
| `/wallet-instance-attestation/jwk` | POST | JWK | Wallet attestation JWT |
| `/wallet-unit-attestation/jwk-set` | POST | JWK Set | Key attestation JWT |

### JWK Serialization with JOSESwift

The iOS wallet uses the `JOSESwift` library for JWK serialization:

As shown in the `WalletKitAttestationProviderImpl` above, JWK serialization uses the `JOSESwift` library's `toDictionary()` method on `JWK` objects. For key attestation, multiple keys are serialized into a JWK Set structure with optional nonce binding:

```swift
// From WalletKitAttestationProviderImpl.getKeysAttestation()
let jwkDict = try keys.map { try $0.toDictionary() }
var payload: [String: Any] = [
  "jwkSet": ["keys": jwkDict]
]
if let nonce {
  payload["nonce"] = nonce
}
```

### Nonce Binding

The `nonce` parameter in `getKeysAttestation(keys:nonce:)` is included in the key attestation request payload, binding the attestation to the specific issuance session. This prevents an attacker from pre-generating key attestations and using them with different sessions.

### Configuration

The wallet provider base URL is configured through `WalletProviderAttestationConfig`:

```swift
struct WalletProviderAttestationConfig {
    let baseURL: URL  // e.g., "https://wallet-provider.eudiw.dev"
}
```

### Secure Enclave Keys

Keys are generated in the iOS Secure Enclave:

```swift
let access = SecAccessControlCreateWithFlags(
    kCFAllocatorDefault,
    kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly,
    [.privateKeyUsage, .biometryCurrentSet],
    nil
)!

let attributes: [String: Any] = [
    kSecAttrKeyType as String: kSecAttrKeyTypeECSECPrimeRandom,
    kSecAttrKeySizeInBits as String: 256,
    kSecAttrTokenID as String: kSecAttrTokenIDSecureEnclave,
    kSecPrivateKeyAttrs as String: [
        kSecAttrAccessControl as String: access,
        kSecAttrIsPermanent as String: true
    ]
]
```

All credential keys are Secure Enclave-backed. The Secure Enclave supports only P-256 (secp256r1) keys, which aligns with the OID4VCI specification's preference for ES256.

---

## 3. Procivis ONE

### Three-Tier Key Storage

Procivis ONE defines three key storage tiers, each with different security properties:

```typescript
enum KeyStorageSecurityBindingEnum {
  INTERNAL = 'INTERNAL',
  SECURE_ELEMENT = 'SECURE_ELEMENT',
  UBIQU_RSE = 'UBIQU_RSE',
}
```

| Tier | Description | Key Location | Authentication |
|------|-------------|-------------|----------------|
| `INTERNAL` | Encrypted application database | App sandbox | App-level (PIN/biometric to open wallet) |
| `SECURE_ELEMENT` | Platform secure element (Android Keystore / iOS Secure Enclave) | Hardware on device | Platform-level (biometric/PIN per key use) |
| `UBIQU_RSE` | Remote Secure Element via Ubiqu | Cloud HSM | PIN + biometric via Ubiqu SDK |

### INTERNAL Key Storage

Keys at the `INTERNAL` tier are stored in the wallet's encrypted database. The database itself is encrypted, but the keys are software-managed:

```
Application Layer
       |
       v
Encrypted Database (SQLCipher or equivalent)
       |
       Contains: private key material (encrypted at rest)
```

This tier provides defense against physical device extraction but does not prevent key extraction by software running in the same application context.

### SECURE_ELEMENT Key Storage

Keys at the `SECURE_ELEMENT` tier use the platform's hardware security module. The core engine generates keys with a platform-specific alias prefix:

```typescript
// Core engine key generation -- conceptual
const keyAlias = `procivis-se-${credentialId}`;
// Key generated in platform Keystore/Secure Enclave with this alias
```

The alias prefix allows the wallet to identify and manage secure element keys. Signing operations go through the platform's key access APIs, requiring biometric or PIN authentication per the key's access control settings.

### UBIQU_RSE (Remote Secure Element)

The `UBIQU_RSE` tier integrates with Ubiqu's Remote Secure Element service. This is the highest security tier and is selected when the credential requires `KeyStorageSecurityBindingEnum.HIGH`:

**RSE Setup Flow:**

```
Wallet App              Ubiqu SDK              Ubiqu Cloud HSM
    |                       |                        |
    |  Initialize RSE       |                        |
    |  --------------------> |                        |
    |                       |  Create key pair       |
    |                       |  --------------------> |
    |                       |                        |
    |                       |  Key pair created      |
    |                       |  (private key in HSM)  |
    |                       |  <-------------------- |
    |                       |                        |
    |  RSE initialized      |                        |
    |  (public key returned) |                        |
    |  <-------------------- |                        |
    |                       |                        |
```

**Signing with RSE:**

Every signing operation requires user authentication through the Ubiqu SDK:

```typescript
// PinEventType events during RSE signing
enum PinEventType {
  PIN_REQUIRED = 'PIN_REQUIRED',
  BIOMETRIC_REQUIRED = 'BIOMETRIC_REQUIRED',
  PIN_INCORRECT = 'PIN_INCORRECT',
}
```

The wallet listens for `PinEventType` events and prompts the user accordingly:

```typescript
// RSE signing flow -- conceptual
const signWithRSE = async (data: Uint8Array) => {
  ubiquSDK.onEvent((event: PinEventType) => {
    switch (event) {
      case PinEventType.PIN_REQUIRED:
        showPinDialog();
        break;
      case PinEventType.BIOMETRIC_REQUIRED:
        showBiometricPrompt();
        break;
      case PinEventType.PIN_INCORRECT:
        showPinError();
        break;
    }
  });

  const signature = await ubiquSDK.sign(data);
  return signature;
};
```

### Security Level Selection

The required security level is determined by the credential configuration. When a credential specifies `HIGH` security, the RSE flow is mandatory:

```typescript
if (credentialConfig.keyStorageSecurity === KeyStorageSecurityBindingEnum.HIGH) {
  // RSE must be set up before credential can be issued
  await ensureRSEInitialized();
}
```

---

## 4. Affinidi

### DID-Based Key Management

Affinidi uses Decentralized Identifiers (DIDs) as the primary key management abstraction. The wallet (Affinidi Vault) holds a `did:key` that represents the holder's identity and signing capability:

```
Affinidi Vault
    |
    +--> did:key:z6Mkh... (holder DID)
    |         |
    |         +--> Private key (used for signing proofs)
    |         +--> Public key (embedded in DID, shared with issuer)
    |
    +--> Signing operations performed by Vault
```

### Key Attestation Visibility

Affinidi's public documentation and TDK do not expose a hardware key attestation mechanism comparable to EUDI's attestation provider or Procivis ONE's security tiers. The attestation model is configuration-driven:

- The Credential Issuance Service is configured with the expected holder DID or claim mode.
- The Vault proves key possession by signing proofs with the DID's private key.
- The trust model relies on the Affinidi platform's integrity rather than hardware attestation of individual keys.

### Configuration-Driven Attestation

Instead of explicit key attestation endpoints, Affinidi uses issuance configuration to define trust:

```json
{
  "issuerDid": "did:key:z6Mkissuer...",
  "credentialOfferDuration": 600,
  "claimMode": "FIXED_HOLDER",
  "holderDid": "did:key:z6Mkholder..."
}
```

The `FIXED_HOLDER` mode binds the credential to a specific DID at configuration time. The Vault proves possession of the corresponding private key during the issuance flow. This is a form of key attestation through DID resolution -- the issuer verifies that the wallet controls the private key for the declared DID -- but it does not attest to hardware storage properties.

### Implications

| Property | Affinidi Approach | EUDI/Procivis Approach |
|----------|-------------------|----------------------|
| Hardware attestation | Not visible in public docs | Explicit (attestation endpoints, security levels) |
| Key binding | DID-based (prove possession of DID key) | JWK-based (prove hardware storage of key) |
| Trust anchor | Affinidi platform | Hardware manufacturer + wallet provider |
| Extractability proof | Not provided | Hardware attestation certificate chain |

This does not necessarily mean Affinidi's keys are less secure -- the Vault may implement hardware-backed storage internally. However, the attestation of that storage is not part of the documented public protocol.

---

## Summary: Key Attestation by Implementation

| Aspect | EUDI Android | EUDI iOS | Procivis ONE | Affinidi |
|--------|-------------|----------|--------------|----------|
| Attestation provider | `WalletCoreAttestationProvider` | `WalletKitAttestationProvider` | Core engine (per tier) | DID-based signing |
| Wallet attestation endpoint | `/wallet-instance-attestation/jwk` | `/wallet-instance-attestation/jwk` | N/A | N/A |
| Key attestation endpoint | `/wallet-unit-attestation/jwk-set` | `/wallet-unit-attestation/jwk-set` | N/A | N/A |
| JWK serialization | SDK-internal | JOSESwift | Core engine | TDK |
| Hardware tiers | TEE, StrongBox | Secure Enclave | INTERNAL, SECURE_ELEMENT, UBIQU_RSE | Not documented |
| Nonce binding in attestation | Yes (`nonce` parameter) | Yes (`nonce` parameter) | Handled by core engine | Session-level |
| Remote SE support | No | No | Yes (Ubiqu RSE) | No |
| Trust chain | Hardware manufacturer --> platform --> wallet provider --> issuer | Same | Varies by tier | Affinidi platform --> issuer |

---

**Next**: [Comparative Analysis](./comparative-analysis.md) -- Strengths, weaknesses, and trade-offs across implementations.
