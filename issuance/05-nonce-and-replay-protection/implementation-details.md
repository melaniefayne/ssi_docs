# Nonce & Replay Protection -- Implementation Details

This document describes how each wallet implementation handles `c_nonce` management and TxCode (transaction code) validation. Nonce handling is largely SDK-internal across all implementations, but TxCode validation surfaces at the application layer and reveals significant differences in input handling, validation strictness, and error reporting.

---

## 1. EUDI Android

### Nonce Handling

The `c_nonce` lifecycle is managed entirely within `OpenId4VciManager` in the `eudi-lib-android-wallet-core` SDK. The application layer does not see, store, or manipulate nonce values directly. The SDK:

1. Extracts `c_nonce` and `c_nonce_expires_in` from the token response.
2. Stores the nonce in the manager's internal state.
3. Includes the nonce as the `nonce` claim in the proof JWT payload.
4. After a credential response, updates the stored nonce if a new `c_nonce` is provided.

### Nonce in Attestation

The nonce is also passed to the attestation provider for key attestation binding:

```kotlin
// File: core-logic/src/main/java/eu/europa/ec/corelogic/provider/WalletCoreAttestationProvider.kt

class WalletCoreAttestationProviderImpl(
    private val walletCoreConfig: WalletCoreConfig,
    private val walletAttestationRepository: WalletAttestationRepository
) : WalletCoreAttestationProvider {

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

The `nonce` parameter in `getKeyAttestation` is the `c_nonce` from the token response. This binds the key attestation to the specific issuance session, preventing an attacker from using a pre-generated key attestation with a different session's nonce.

### TxCode Validation

TxCode validation occurs in `DocumentOfferInteractor` and `DocumentOfferCodeViewModel`.

**Input mode restriction:** The EUDI Android wallet accepts only `numeric` input mode. If the credential offer specifies `text` mode for the TxCode, the wallet rejects it:

```kotlin
// File: issuance-feature/src/main/java/eu/europa/ec/issuancefeature/interactor/DocumentOfferInteractor.kt

val codeMinLength = 4
val codeMaxLength = 6

safeLet(
    response.offer.txCodeSpec?.inputMode,
    response.offer.txCodeSpec?.length
) { inputMode, length ->

    if ((length !in codeMinLength..codeMaxLength) || inputMode == TxCodeInputMode.TEXT) {
        return@map ResolveDocumentOfferInteractorPartialState.Failure(
            errorMessage = resourceProvider.getString(
                R.string.issuance_document_offer_error_invalid_txcode_format,
                codeMinLength,
                codeMaxLength
            )
        )
    }
}
```

**Digit range:** The wallet accepts TxCodes between 4 and 6 digits in length (as specified in the offer). Codes outside this range or containing non-numeric characters are rejected at the UI level before submission.

**UI behavior:** The code input field uses a numeric keyboard. The submit button is disabled until the entered code matches the expected length.

---

## 2. EUDI iOS

### Nonce Handling

The iOS SDK manages nonces identically to Android. The `c_nonce` is extracted from the token response, stored internally, and included in proof JWTs. The application layer does not interact with nonce values.

### Nonce in Attestation

The nonce is passed to the iOS attestation provider:

```swift
// WalletKitAttestationProvider
func getKeysAttestation(keys: [SecKey], nonce: String) async throws -> String
```

The `nonce` parameter serves the same session-binding purpose as on Android.

### TxCode Validation

TxCode validation occurs in `OfferCodeViewModel`.

**Debounced input:** The iOS wallet debounces TxCode input with a 250ms delay. This prevents rapid validation cycles as the user types:

```swift
// File: Modules/feature-issuance/Sources/Interactor/DocumentOfferInteractor.swift

let codeMinLength = 4
let codeMaxLength = 6

let offer = try await walletController.resolveOfferUrlDocTypes(offerUri: uri)

if let spec = offer.txCodeSpec,
   let codeLength = spec.length,
   !(codeMinLength...codeMaxLength).contains(codeLength) || spec.inputMode == .text {
  return .failure(WalletCoreError.transactionCodeFormat(["\(codeMinLength)", "\(codeMaxLength)"]))
}
```

**Length validation:** Exact length match is required. The submit action is gated on `code.count == expectedLength`.

**Keyboard type:** The input field presents a numeric keypad (`.numberPad`), preventing alphabetic input at the system level.

---

## 3. Procivis ONE

### Nonce Handling

Nonce management is handled by the Procivis native core engine (`@procivis/react-native-one-core`). The React Native application layer does not access nonce values. The core engine:

1. Receives `c_nonce` from the token endpoint.
2. Includes it in proof construction.
3. Handles nonce rotation from credential responses.

### TxCode Validation

TxCode input is handled in `credential-confirmation-code-screen.tsx`.

**Input mode support:** Unlike the EUDI wallets, Procivis ONE supports both `NUMERIC` and `TEXT` input modes:

```typescript
// File: app/screens/credential/credential-confirmation-code-screen.tsx

const {
    invalidCode,
    invitationResult: { txCode: optionalTxCode, keyStorageSecurityLevels = [] },
  } = route.params;
  const txCode = optionalTxCode!;
  const [code, setCode] = useState<string | undefined>(invalidCode);
  const isInputLengthValid = code && code.length === txCode.length;
  const isInvalid = invalidCode && code === invalidCode;
  const isNumericInput =
    txCode.inputMode === OpenId4vciTxCodeInputModeBindingEnum.NUMERIC;
  const keyboardType = isNumericInput ? 'number-pad' : 'default';
  const submitBtnDisabled = Boolean(!code || !isInputLengthValid);
```

The `OpenId4vciTxCodeInputModeBindingEnum` maps directly to the OID4VCI specification's `input_mode` values:

| Enum Value | Keyboard | Accepted Characters |
|-----------|----------|---------------------|
| `NUMERIC` | Numeric keypad | Digits only (0-9) |
| `TEXT` | Full keyboard | Alphanumeric characters |

**Error handling:** Procivis ONE uses specific error codes for TxCode validation failures:

| Error Code | Description |
|------------|-------------|
| `BR_0169` | Invalid transaction code format (wrong mode or length) |
| `BR_0170` | Transaction code verification failed (wrong code submitted to issuer) |

These error codes are returned by the core engine and displayed to the user with appropriate messages.

**Security level check:** Before proceeding with issuance after TxCode validation, Procivis ONE checks the required security level. If the credential requires RSE (Remote Secure Element) key storage, the RSE setup flow is triggered before the credential request:

```typescript
// File: app/screens/credential/credential-confirmation-code-screen.tsx

const handleSubmit = useCallback(() => {
    const needsRSESetup =
      keyStorageSecurityLevels?.includes(KeyStorageSecurityBindingEnum.HIGH) &&
      !walletStore.isRSESetup;

    navigation.replace(needsRSESetup ? 'RSEInfo' : 'CredentialOffer', {
      invitationResult: route.params.invitationResult,
      txCode: code,
    });
  }, [
    code,
    navigation,
    route.params.invitationResult,
    keyStorageSecurityLevels,
    walletStore.isRSESetup,
  ]);
```

---

## 4. Affinidi

### Nonce Handling

Nonce management is handled by the Affinidi Credential Issuance Service and the Vault application. The `c_nonce` is included in the token response from the issuance service. The Vault's TDK client library includes the nonce in proof construction.

### TxCode -- TX_CODE Claim Mode

Affinidi's transaction code implementation uses the `TX_CODE` claim mode in the issuance configuration:

```json
{
  "claimMode": "TX_CODE",
  "credential_configuration_ids": ["VerifiedEmail"],
  "tx_code": {
    "input_mode": "numeric",
    "length": 6
  }
}
```

When `TX_CODE` mode is configured:

1. The Credential Issuance Service generates a transaction code and includes it in the credential offer.
2. The transaction code is delivered to the user through a separate channel (e.g., the issuer's application displays it, or it is sent via email/SMS).
3. The user enters the code in the Affinidi Vault during the claiming process.
4. The Vault includes the code in the token request.
5. The Issuance Service validates the code against the generated value.

### FIXED_HOLDER Mode (Alternative to TxCode)

In `FIXED_HOLDER` mode, DID-based validation replaces the transaction code:

```json
{
  "claimMode": "FIXED_HOLDER",
  "holderDid": "did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK"
}
```

In this mode:
- No transaction code is generated or required.
- The Vault presents the holder's DID during the token exchange.
- The Issuance Service validates that the presented DID matches the `holderDid` in the configuration.
- If the DID matches, the access token is issued without a TxCode.

This is functionally equivalent to TxCode verification in that it binds the issuance to a specific intended recipient, but it uses cryptographic identity (DID) rather than a shared secret (code).

### Validation Flow

```
Issuance Configuration:
  claimMode = TX_CODE
       |
       v
Generate offer with tx_code requirement
       |
       v
User receives code out-of-band
       |
       v
User enters code in Vault
       |
       v
Vault sends to token endpoint:
  pre-authorized_code + tx_code
       |
       v
Issuance Service validates code
       |
       +--> Match: issue access_token + c_nonce
       +--> Mismatch: return error
```

---

## Summary: Nonce and TxCode Handling by Implementation

| Aspect | EUDI Android | EUDI iOS | Procivis ONE | Affinidi |
|--------|-------------|----------|--------------|----------|
| Nonce visibility | SDK-internal | SDK-internal | Core engine-internal | Service-internal |
| Nonce in attestation | `getKeyAttestation(keys, nonce)` | `getKeysAttestation(keys:nonce:)` | N/A (core handles) | N/A (service handles) |
| TxCode input modes | Numeric only | Numeric only | Numeric and Text | Numeric (TX_CODE mode) |
| Length validation | Exact match, 4-6 digits | Exact match with 250ms debounce | Via core engine + error codes | Service-side validation |
| Text mode support | Rejected | Rejected | Supported | Not documented |
| Error codes | UI-level validation | UI-level validation | BR_0169, BR_0170 | HTTP error responses |
| Alternative to TxCode | N/A | N/A | N/A | FIXED_HOLDER (DID validation) |

---

**Next**: [Comparative Analysis](./comparative-analysis.md) -- Strengths, weaknesses, and trade-offs across implementations.
