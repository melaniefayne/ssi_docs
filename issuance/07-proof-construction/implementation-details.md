# Proof Construction -- Implementation Details

This document details how each wallet implementation constructs cryptographic proofs during credential issuance, including SDK encapsulation, user authentication gating, and signing key management.

---

## EUDI Android (Reference Implementation)

### SDK-Level Proof Construction

Proof construction in the EUDI Android wallet is handled internally by the **OpenId4VciManager** within the EUDI SDK. The application layer does not directly construct JWT proofs -- the SDK handles JWT header assembly, payload construction (including `c_nonce` injection), and signing. The application's responsibility is to handle user authentication events that gate access to the signing key.

### User Authentication Flow

The SDK emits events through the `IssueEvent` callback interface. When the proof must be signed, the SDK emits `IssueEvent.DocumentRequiresUserAuth`, signaling that the signing key requires user authentication before it can be used.

The application layer responds to this event by presenting a **BiometricPrompt** to the user:

```
SDK (OpenId4VciManager)              App Layer
  |                                     |
  |  IssueEvent.DocumentRequiresUserAuth|
  |------------------------------------>|
  |                                     |  Present BiometricPrompt
  |                                     |  User authenticates
  |                                     |
  |  Resume with keyUnlockDataMap       |
  |<------------------------------------|
  |                                     |
  |  Sign proof JWT with unlocked key   |
  |  Send credential request            |
  |                                     |
```

### BiometricPrompt Integration

The `DocumentRequiresUserAuth` event provides a `getCryptoObjectForSigning()` method that returns a `BiometricPrompt.CryptoObject` bound to the secure key in the Android Keystore. This crypto object is passed to the BiometricPrompt, ensuring that biometric authentication unlocks the specific key needed for signing.

```kotlin
// File: core-logic/src/main/java/eu/europa/ec/corelogic/controller/WalletCoreDocumentsController.kt

is IssueEvent.DocumentRequiresUserAuth -> {
    launch {
        val keyUnlockDataMap =
            event.keysRequireAuth.mapValues { (keyAlias, secureArea) ->
                getDefaultKeyUnlockData(secureArea, keyAlias)
            }

        val keyUnlockData =
            keyUnlockDataMap.values.first()

        val cryptoObject = keyUnlockData?.getCryptoObjectForSigning()

        trySendBlocking(
            IssueDocumentsPartialState.UserAuthRequired(
                crypto = BiometricCrypto(cryptoObject),
                resultHandler = DeviceAuthenticationResult(
                    onAuthenticationSuccess = { event.resume(keyUnlockDataMap) },
                    onAuthenticationError = { event.cancel(null) }
                )
            )
        )
    }
}
```

**Authenticator types:**

The BiometricPrompt is configured with:

```
BiometricPrompt.PromptInfo.Builder()
    .setAllowedAuthenticators(
        BiometricManager.Authenticators.BIOMETRIC_WEAK
        or BiometricManager.Authenticators.DEVICE_CREDENTIAL
    )
```

- `BIOMETRIC_WEAK` -- Allows fingerprint, face recognition, and iris scanning. The "weak" designation means it accepts Class 2 biometrics (not just Class 3/strong), broadening device compatibility.
- `DEVICE_CREDENTIAL` -- Allows the device lock screen credential (PIN, pattern, password) as a fallback when biometrics are unavailable or not enrolled.

### DeviceAuthenticationController

The **DeviceAuthenticationController** orchestrates the authentication flow. It manages the lifecycle of the BiometricPrompt, handles authentication callbacks (success, failure, error), and coordinates the resumption of the issuance flow with the authenticated crypto object.

On successful authentication:

1. The BiometricPrompt returns the authenticated `CryptoObject`.
2. The controller packages the authentication result into a `keyUnlockDataMap`.
3. The SDK event is resumed with this map, allowing the SDK to use the unlocked key for signing.
4. The SDK constructs and signs the JWT proof internally.

On authentication failure:

1. The controller may retry (if the failure is recoverable, e.g., fingerprint not recognized).
2. After maximum retries or user cancellation, the issuance flow is cancelled.
3. The SDK emits `IssueEvent.DocumentFailed` with the authentication error.

The SDK also emits `DocumentRequiresCreateSettings` before proof construction, allowing the app to configure credential policies per document type:

```kotlin
// File: core-logic/src/main/java/eu/europa/ec/corelogic/controller/WalletCoreDocumentsController.kt

is IssueEvent.DocumentRequiresCreateSettings -> {
    launch {
        val offeredDocIdentifier = event.offeredDocument.documentIdentifier

        val documentIssuanceRule = walletCoreConfig
            .documentIssuanceConfig
            .getRuleForDocument(documentIdentifier = offeredDocIdentifier)

        event.resume(
            eudiWallet.getDefaultCreateDocumentSettings(
                offeredDocument = event.offeredDocument,
                credentialPolicy = documentIssuanceRule.policy,
                numberOfCredentials = documentIssuanceRule.numberOfCredentials,
            )
        )
    }
}
```

### Key Storage

The signing key resides in the **Android Keystore**, backed by the device's Trusted Execution Environment (TEE) or StrongBox Secure Element (if available). The key is configured with `setUserAuthenticationRequired(true)`, meaning the Keystore enforces that user authentication must occur before each signing operation. The BiometricPrompt / device credential flow satisfies this requirement.

---

## EUDI iOS (Reference Implementation)

### SDK-Level Proof Construction

Proof construction in the EUDI iOS wallet is handled within the **EudiWalletKit** SDK. Similar to the Android implementation, the iOS SDK encapsulates JWT proof construction -- assembling the header with the holder's public key, populating the payload with `c_nonce` and audience claims, and signing with the holder's private key.

### DPoP Token Generation

The iOS implementation generates DPoP tokens using the **JOSESwift** library. DPoP proofs are constructed as separate JWTs with `typ: "dpop+jwt"` and include the HTTP method, target URI, and access token hash. The DPoP key pair is managed alongside the credential binding key pair, though they may be distinct keys.

### Biometric Authentication

Key access is gated by biometric authentication via the **SystemBiometryController**. This controller wraps the iOS Local Authentication framework and manages Face ID / Touch ID prompts.

The flow:

1. The SDK determines that the signing key requires biometric authentication.
2. `SystemBiometryController` presents a Face ID or Touch ID prompt to the user.
3. On successful authentication, the Secure Enclave unlocks the private key for a single signing operation.
4. The SDK signs the JWT proof with the unlocked key.

### Key Storage

The signing key resides in the **iOS Secure Enclave**, a hardware-isolated processor that performs cryptographic operations without exposing the private key to the application processor. Keys stored in the Secure Enclave are protected by the Secure Enclave's own access control policies, which can require biometric authentication (`kSecAccessControlBiometryCurrentSet` or `kSecAccessControlBiometryAny`).

The Secure Enclave supports `ES256` (P-256) natively. This is the primary algorithm used for proof construction in the EUDI iOS wallet.

---

## Procivis ONE

### Native Bridge Proof Construction

Procivis ONE handles proof construction internally within the **One Core SDK**, which is accessed from the React Native application layer via a native bridge. The application layer does not directly construct JWT proofs or manage signing keys -- these operations occur entirely within the native SDK.

### RSE Signing (Remote Signing Element)

When the credential binding uses a Remote Signing Element (RSE), proof construction requires PIN-based authentication:

1. The SDK initiates the signing operation and determines that RSE signing is required.
2. The SDK emits a `PinEventType.SHOW_PIN` event with `PinFlowType.TRANSACTION`.
3. The application layer presents the **RSESign** screen, prompting the user to enter their PIN.
4. The user enters the PIN, which is passed back to the SDK via the native bridge.
5. The SDK authenticates with the RSE using the PIN and performs the signing operation remotely.
6. The signed proof JWT is included in the credential request.

**RSE signing flow:**

```
One Core SDK                    React Native App
  |                                |
  |  PinEventType.SHOW_PIN         |
  |  PinFlowType.TRANSACTION       |
  |------------------------------->|
  |                                |  Show RSESign screen
  |                                |  User enters PIN
  |  PIN value                     |
  |<-------------------------------|
  |                                |
  |  Authenticate with RSE         |
  |  Sign proof JWT                |
  |  Continue credential request   |
  |                                |
```

### Non-RSE Signing

When RSE is not used, the platform's **Secure Element** handles signing directly. The key resides in the device's hardware-backed keystore (Android Keystore or iOS Secure Enclave), and the SDK performs the signing operation through the platform's native cryptographic APIs. No additional user authentication step is required beyond the initial app authentication, unless the key's access control policy mandates it.

### acceptCredential() Parameters

After proof construction and credential issuance, the credential is accepted into the wallet via `acceptCredential()`, which takes:

- `holderWalletUnitId` -- Identifies the wallet unit that will hold the credential.
- `interactionId` -- References the specific issuance interaction, binding the acceptance to the session.

### Supported Algorithms

Procivis ONE supports a broader algorithm set than the EUDI implementations:

- **ES256** (P-256) -- Standard ECDSA, compatible with EUDI ecosystem.
- **EdDSA** (Ed25519) -- Edwards-curve Digital Signature Algorithm.
- **CRYSTALS-DILITHIUM** -- Post-quantum lattice-based signature scheme. This is a forward-looking capability for environments that require quantum-resistant cryptography.

The algorithm selection is determined by the credential configuration and issuer requirements, not by user choice.

---

## Affinidi

### Vault-Based Proof Construction

In the Affinidi ecosystem, proof construction occurs within the **Affinidi Vault** during the credential claim process. The Vault is a cloud-based wallet that manages the holder's key material and performs signing operations server-side.

### Signing Algorithm

Affinidi uses **EcdsaSecp256k1Signature2019** as the primary signing suite. This is a JSON-LD signature suite that uses the secp256k1 elliptic curve (the same curve used in Bitcoin and Ethereum). The choice of secp256k1 reflects Affinidi's roots in the decentralized identity ecosystem, where this curve is prevalent.

### VP Proof Construction

When Affinidi constructs Verifiable Presentation (VP) proofs, the proof includes additional parameters:

- **`challenge`** -- A nonce provided by the verifier (or issuer, in the issuance context) that prevents replay.
- **`domain`** -- The intended audience for the proof, typically the verifier's or issuer's domain.

These parameters serve the same purpose as `nonce` and `aud` in the JWT proof type but follow the JSON-LD Linked Data Proofs specification rather than the OID4VCI JWT proof specification.

**Example VP proof structure:**

```json
{
  "type": "EcdsaSecp256k1Signature2019",
  "created": "2024-01-15T12:00:00Z",
  "verificationMethod": "did:elem:...",
  "proofPurpose": "authentication",
  "challenge": "abc123",
  "domain": "https://issuer.example.com",
  "jws": "eyJhbGciOi..."
}
```

### Cloud Key Management

Unlike the EUDI and Procivis implementations, which store signing keys in device-local hardware-backed keystores, Affinidi manages keys in the cloud within the Vault infrastructure. This means:

- Proof construction does not require device-level biometric or PIN authentication at signing time.
- Key portability is inherent -- the user can access their credentials and sign proofs from any device by authenticating to the Vault.
- The security model shifts from hardware-backed key protection to cloud infrastructure security and Vault access controls.

---

## Summary

Proof construction is handled at different abstraction levels across implementations. The EUDI wallets delegate proof construction to the SDK and surface user authentication requirements to the application layer via event callbacks. Procivis ONE encapsulates proof construction entirely within the native SDK, with PIN-based RSE signing as a distinct path. Affinidi constructs proofs server-side within the Vault using JSON-LD signature suites. The choice of signing mechanism -- biometric-gated, PIN-gated, or cloud-based -- reflects each implementation's security model, key management architecture, and target deployment context.
