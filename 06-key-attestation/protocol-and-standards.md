# Key Attestation -- Protocol & Standards

This document details the technical standards and mechanisms underlying key attestation in OID4VCI credential issuance. It covers JWK key representation, the role of key attestation in OID4VCI proofs, wallet instance attestation, and platform-specific attestation mechanisms for Android and iOS.

---

## 1. JWK Key Representation (RFC 7517)

JSON Web Key ([RFC 7517](https://datatracker.ietf.org/doc/html/rfc7517)) is the standard format for representing cryptographic keys in the OID4VCI ecosystem. All key material exchanged between wallet, wallet provider, and issuer uses JWK encoding.

### EC Key Example (P-256)

```json
{
  "kty": "EC",
  "crv": "P-256",
  "x": "f83OJ3D2xF1Bg8vub9tLe1gHMzV76e8Tus9uPHvRVEU",
  "y": "x_FEzRu9m36HLN_tue659LNpXW6pCyStikYjKIWI5a0",
  "kid": "wallet-key-1"
}
```

### JWK Set (Multiple Keys)

When attesting multiple keys (e.g., for batch issuance), a JWK Set is used:

```json
{
  "keys": [
    {
      "kty": "EC",
      "crv": "P-256",
      "x": "...",
      "y": "...",
      "kid": "key-1"
    },
    {
      "kty": "EC",
      "crv": "P-256",
      "x": "...",
      "y": "...",
      "kid": "key-2"
    }
  ]
}
```

### Key Properties Relevant to Attestation

| Property | Description |
|----------|-------------|
| `kty` | Key type. `EC` for elliptic curve, `RSA` for RSA keys. |
| `crv` | Curve identifier. `P-256` is the most common in OID4VCI. |
| `x`, `y` | Public key coordinates (EC keys). |
| `kid` | Key identifier. Used to reference the key in subsequent protocol messages. |
| `key_ops` | Key operations permitted (e.g., `["sign", "verify"]`). |

Only the public key is transmitted. The private key remains in the wallet's secure storage and is never serialized or transmitted.

---

## 2. Key Attestation in OID4VCI

In OID4VCI, key attestation is integrated into the proof-of-possession mechanism. When the wallet sends a credential request, the proof JWT may include attestation information that allows the issuer to verify the key's security properties.

### Proof JWT with Key Attestation

The standard OID4VCI proof JWT includes the public key in its header:

```json
{
  "typ": "openid4vci-proof+jwt",
  "alg": "ES256",
  "jwk": {
    "kty": "EC",
    "crv": "P-256",
    "x": "...",
    "y": "..."
  }
}
```

Key attestation extends this by providing additional evidence about the key's provenance. The specific mechanism depends on the implementation:

- **Embedded attestation:** The wallet includes a key attestation certificate chain in the proof or alongside it.
- **Referenced attestation:** The wallet includes a reference (e.g., a URI) to a key attestation that the issuer can fetch.
- **Backend attestation:** The wallet provider attests the key through a backend-to-backend channel, and the issuer trusts the wallet provider's assertion.

### Attestation JWT

A key attestation JWT issued by the wallet provider might contain:

```json
{
  "iss": "https://wallet-provider.example.com",
  "sub": "wallet-instance-123",
  "iat": 1700000000,
  "exp": 1700003600,
  "cnf": {
    "jwk": {
      "kty": "EC",
      "crv": "P-256",
      "x": "...",
      "y": "..."
    }
  },
  "key_type": "hardware",
  "security_level": "strongbox",
  "user_authentication": "biometric"
}
```

This JWT asserts that the key identified in the `cnf` (confirmation) claim is hardware-backed, stored in StrongBox, and protected by biometric authentication. The issuer validates this JWT against the wallet provider's public key.

---

## 3. Wallet Instance Attestation

Wallet instance attestation is a separate mechanism from key attestation. It proves that the wallet application is genuine, not that specific keys are hardware-backed.

### Flow

```
Wallet App            Wallet Provider Backend            Issuer
    |                         |                            |
    |  Attestation Request    |                            |
    |  (app identity proof)   |                            |
    |  ---------------------> |                            |
    |                         |                            |
    |  Wallet Attestation JWT |                            |
    |  <--------------------- |                            |
    |                         |                            |
    |                         |                            |
    |  Credential Request with wallet attestation          |
    |  --------------------------------------------------> |
    |                         |                            |
    |  Issuer validates attestation against                 |
    |  wallet provider's public key                        |
    |                         |                            |
```

### Wallet Attestation JWT Structure

```json
{
  "iss": "https://wallet-provider.example.com",
  "sub": "wallet-instance-abc",
  "iat": 1700000000,
  "exp": 1700086400,
  "wallet_name": "EUDI Wallet",
  "wallet_version": "1.5.0",
  "platform": "android",
  "device_integrity": "verified"
}
```

The wallet provider signs this JWT with its own key. The issuer trusts the wallet provider (through a pre-established trust relationship or a trust registry) and validates the JWT signature.

---

## 4. Android Key Attestation

Android provides hardware-backed key attestation through the Keystore system.

### Android Keystore Architecture

```
Application Layer
       |
       v
Android Keystore API (java.security.KeyStore)
       |
       v
Keymaster / KeyMint HAL
       |
       +--> TEE (Trusted Execution Environment)
       |         ARM TrustZone on most devices
       |         Key generation and signing happen here
       |
       +--> StrongBox (if available)
                Dedicated secure element chip
                Highest security level
                Tamper-resistant hardware
```

### Key Attestation Certificate Chain

When a key is generated with attestation requested, Android produces a certificate chain:

```
Root CA Certificate (Google Hardware Attestation Root)
       |
       v
Intermediate CA Certificate (device-specific)
       |
       v
Attestation Certificate (key-specific)
       |
       Contains:
       - Public key
       - Key properties (algorithm, purpose, digest)
       - Security level (TEE or StrongBox)
       - Attestation challenge (the nonce provided by the requester)
       - OS version, patch level
       - Application ID (package name + signing certificate hash)
```

### Security Levels

| Level | Description | Hardware |
|-------|-------------|----------|
| `Software` | Key stored in Android Keystore but not hardware-backed | None (software emulation) |
| `TrustedEnvironment` | Key stored in TEE (Trusted Execution Environment) | ARM TrustZone |
| `StrongBox` | Key stored in dedicated secure element | Separate chip (e.g., Titan M on Pixel) |

For EUDI compliance, `TrustedEnvironment` or `StrongBox` is required. `Software`-level keys are insufficient for credential issuance.

### Attestation Challenge

The attestation challenge is a nonce provided at key generation time:

```kotlin
val keyGenParameterSpec = KeyGenParameterSpec.Builder("my-key", PURPOSE_SIGN)
    .setDigests(KeyProperties.DIGEST_SHA256)
    .setAlgorithmParameterSpec(ECGenParameterSpec("secp256r1"))
    .setAttestationChallenge(nonce.toByteArray()) // Server-provided nonce
    .build()
```

The nonce is embedded in the attestation certificate, proving that the key was generated in response to a specific request (preventing pre-generation attacks).

---

## 5. iOS Secure Enclave and App Attest

iOS provides key attestation through two complementary mechanisms: the Secure Enclave for key generation/storage and App Attest for application integrity.

### Secure Enclave

The Secure Enclave is a dedicated security coprocessor on Apple devices (iPhone, iPad, Mac with Apple Silicon). It:

- Generates P-256 (secp256r1) key pairs internally.
- Never exposes private keys to the application processor.
- Performs signing operations inside the coprocessor.
- Maintains a hardware-fused unique identifier (UID) that cannot be read by software.

```swift
// Generate a key in the Secure Enclave
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
        kSecAttrIsPermanent as String: true,
        kSecAttrApplicationTag as String: "wallet-key-1"
    ]
]

var error: Unmanaged<CFError>?
let privateKey = SecKeyCreateRandomKey(attributes as CFDictionary, &error)
```

### App Attest (DeviceCheck Framework)

App Attest verifies the authenticity of the application:

1. The app generates an attestation key in the Secure Enclave via the `DCAppAttestService`.
2. The app requests attestation from Apple's servers, which return a signed attestation object.
3. The attestation object contains the app's identity (team ID, bundle ID) and the key's fingerprint.
4. The server validates the attestation against Apple's root CA.

```swift
let service = DCAppAttestService.shared
let keyId = try await service.generateKey()
let challenge = serverProvidedChallenge.sha256()
let attestation = try await service.attestKey(keyId, clientDataHash: challenge)
// Send attestation to server for validation
```

### Limitations

- Secure Enclave supports only P-256 keys. Other curves or key types require software-based key storage.
- App Attest requires a network connection to Apple's servers during attestation (not during subsequent signing).
- Older devices without Secure Enclave cannot provide hardware key attestation.

---

## 6. Remote Secure Elements

A Remote Secure Element (RSE) is a cloud-hosted hardware security module (HSM) that provides key generation, storage, and signing as a service. The private key never leaves the HSM; the wallet communicates with the RSE over an authenticated, encrypted channel.

### RSE Architecture

```
Wallet App             RSE SDK              RSE Cloud (HSM)
    |                     |                       |
    |  Sign request       |                       |
    |  (data + auth)      |                       |
    |  -----------------> |                       |
    |                     |  Authenticated request |
    |                     |  -------------------> |
    |                     |                       |
    |                     |  Key never leaves HSM |
    |                     |  Signing in hardware  |
    |                     |                       |
    |                     |  Signature             |
    |                     |  <------------------- |
    |  Signature          |                       |
    |  <----------------- |                       |
    |                     |                       |
```

### RSE Properties

| Property | Value |
|----------|-------|
| Key generation | In HSM hardware |
| Key storage | In HSM, never exported |
| Signing | In HSM, result transmitted to client |
| Authentication | PIN, biometric, or both (enforced by RSE SDK) |
| Network requirement | Online for every signing operation |
| Certification | HSM may be FIPS 140-2/3 or Common Criteria certified |

### Trade-offs

**Advantages:**
- Provides hardware-grade key security regardless of device capabilities.
- Works on devices without local secure elements.
- HSM certification may satisfy regulatory requirements that device TEE does not.
- Key recovery is possible (under appropriate controls) if the device is lost.

**Disadvantages:**
- Requires network connectivity for every signing operation.
- Introduces a dependency on the RSE provider's availability.
- Adds latency to signing operations.
- The user must trust the RSE provider not to misuse the key.

---

## 7. Standards Reference

| Standard | Title | Relevance |
|----------|-------|-----------|
| [RFC 7517](https://datatracker.ietf.org/doc/html/rfc7517) | JSON Web Key (JWK) | Key representation format |
| [RFC 7638](https://datatracker.ietf.org/doc/html/rfc7638) | JSON Web Key Thumbprint | Key identification by hash |
| [RFC 7515](https://datatracker.ietf.org/doc/html/rfc7515) | JSON Web Signature (JWS) | Signature format for attestation JWTs |
| OID4VCI | OpenID for Verifiable Credential Issuance | Proof format, key binding, attestation integration |
| EUDI ARF | EU Digital Identity Architecture Reference Framework | WSCD requirements, attestation trust model |
| Android Keystore | Android Key Attestation | Hardware-backed key generation and attestation |
| Apple DeviceCheck | App Attest API | Application integrity verification |

---

**Next**: [Implementation Details](./implementation-details.md) -- How each wallet implements key attestation.
