# Nonce & Replay Protection -- Comparative Analysis

This document compares how the four wallet implementations handle nonces and transaction codes. The comparison reveals significant differences in TxCode input flexibility, validation strictness, error handling granularity, and the boundary between protocol-level nonce management and application-level user interaction.

---

## Feature Comparison Matrix

| Feature | EUDI Android | EUDI iOS | Procivis ONE | Affinidi |
|---------|-------------|----------|--------------|----------|
| **c_nonce management** | SDK-internal | SDK-internal | Core engine | Issuance service |
| **Nonce in attestation** | Yes | Yes | Core engine handles | Service handles |
| **TxCode: numeric mode** | Yes | Yes | Yes | Yes (TX_CODE mode) |
| **TxCode: text mode** | Rejected | Rejected | Yes | Not documented |
| **Length validation** | Exact match (4-6 digits) | Exact match with debounce | Core engine + error codes | Service-side |
| **Input debounce** | None | 250ms | None documented | N/A |
| **Error codes** | UI-level | UI-level | BR_0169, BR_0170 | HTTP errors |
| **Alternative to TxCode** | None | None | None | FIXED_HOLDER (DID) |

---

## TxCode Input Modes

### Numeric-Only Implementations (EUDI Android & iOS)

Both EUDI wallets restrict TxCode input to numeric mode. If a credential offer specifies `"input_mode": "text"`, the offer is treated as unsupported. The keyboard is forced to numeric, and only digit characters are accepted.

**Rationale:** The OID4VCI specification defaults to numeric mode, and the EU regulatory context typically uses numeric PINs for transaction verification. Supporting only numeric reduces the attack surface (shorter character set) and simplifies validation.

**Consequence:** Issuers that use alphanumeric transaction codes (e.g., codes generated from a base32 or hex alphabet) are incompatible with EUDI wallets. This is a deliberate design constraint, not an oversight.

### Dual-Mode Implementation (Procivis ONE)

Procivis ONE supports both `NUMERIC` and `TEXT` modes through `OpenId4vciTxCodeInputModeBindingEnum`. The application renders the appropriate keyboard type based on the mode specified in the credential offer.

**Rationale:** Procivis ONE's multi-protocol architecture aims for broad issuer compatibility. Different issuers may use different TxCode formats depending on their infrastructure and security requirements.

**Consequence:** The wallet can work with any compliant issuer, regardless of TxCode format. However, supporting text mode increases the UI complexity and the validation logic.

### Configuration-Driven (Affinidi)

Affinidi supports numeric TxCodes through the `TX_CODE` claim mode. The `FIXED_HOLDER` mode provides an alternative that eliminates TxCodes entirely by using DID-based holder identification.

---

## Validation Strictness

### EUDI Android

- **Character validation:** Only digits (`0-9`). Non-digit input is prevented by the numeric keyboard.
- **Length validation:** Exact match against the `length` field in the TxCode descriptor. The submit button is disabled until the code is exactly the expected length.
- **Range:** Accepts codes between 4 and 6 digits (as specified in the offer).
- **Client-side rejection:** Invalid input is rejected before any network request. The user cannot submit an incorrectly formatted code.

### EUDI iOS

- **Character validation:** Same as Android -- digits only.
- **Length validation:** Exact match, but with a 250ms debounce on input changes.
- **Debounce behavior:** The validation function is not called on every keystroke. Instead, it waits for 250ms of input inactivity before validating. This prevents flickering validation states during rapid typing.
- **Client-side rejection:** Same as Android -- invalid codes are rejected before submission.

### Procivis ONE

- **Character validation:** Mode-dependent. Numeric mode accepts digits; text mode accepts alphanumeric characters.
- **Length validation:** Handled by the core engine after submission. The application does not enforce length at the UI level.
- **Error handling:** The core engine returns specific error codes:
  - `BR_0169`: Invalid format (wrong input mode or length mismatch). Returned before the code is sent to the issuer.
  - `BR_0170`: Verification failed (correct format, but the issuer rejected the code). Returned after the token endpoint responds.
- **Retry behavior:** On `BR_0170`, the user can re-enter the code. On `BR_0169`, the UI prompts for correction.

### Affinidi

- **Validation:** Performed server-side by the Credential Issuance Service. The Vault collects the code and submits it; the service validates format and correctness.
- **Error handling:** Standard HTTP error responses (e.g., `400 Bad Request` with `invalid_grant` error).

### Comparison

| Validation Aspect | EUDI Android | EUDI iOS | Procivis ONE | Affinidi |
|-------------------|-------------|----------|--------------|----------|
| Where validated | Client (UI) | Client (UI, debounced) | Client + core engine | Server |
| Invalid format | Prevented by UI | Prevented by UI | BR_0169 error code | HTTP 400 |
| Wrong code | Server rejects | Server rejects | BR_0170 error code | HTTP 400 |
| Retry allowed | Yes | Yes | Yes | Yes |
| Exact length enforced | UI-level | UI-level (debounced) | Core engine | Server |

---

## Nonce Management Transparency

All implementations treat `c_nonce` as an internal protocol detail. No implementation exposes the nonce value to the application layer or requires the application to manage nonce state.

| Implementation | Nonce Managed By | Application Visibility | Nonce Rotation |
|---------------|-----------------|----------------------|----------------|
| EUDI Android | `OpenId4VciManager` (SDK) | None | SDK handles automatically |
| EUDI iOS | `OpenId4VciManager` (SDK) | None | SDK handles automatically |
| Procivis ONE | Core engine | None | Core engine handles |
| Affinidi | Credential Issuance Service | None | Service handles |

This is an area of strong consensus across implementations. The nonce is a protocol-level concern and should not leak into the application layer. Application developers do not need to understand or manage nonces.

### Nonce Binding to Attestation

The EUDI wallets are the only implementations that explicitly pass the `c_nonce` to the key attestation provider:

| Implementation | Nonce in Attestation |
|---------------|---------------------|
| EUDI Android | `getKeyAttestation(keys, nonce)` -- nonce is an explicit parameter |
| EUDI iOS | `getKeysAttestation(keys:nonce:)` -- nonce is an explicit parameter |
| Procivis ONE | Handled internally by core engine |
| Affinidi | Handled internally by issuance service |

This explicit nonce-to-attestation binding in the EUDI implementations provides an additional layer of session integrity: the key attestation itself is bound to the issuance session, not just the proof JWT.

---

## Error Handling for Invalid Codes

### Error Granularity

| Implementation | Format Error | Wrong Code Error | User Message |
|---------------|-------------|-----------------|--------------|
| EUDI Android | UI prevents submission | Generic error from server | "Invalid code" |
| EUDI iOS | UI prevents submission (debounced) | Generic error from server | "Invalid code" |
| Procivis ONE | `BR_0169` (distinct code) | `BR_0170` (distinct code) | Specific per error type |
| Affinidi | Server returns `invalid_grant` | Server returns `invalid_grant` | Generic error |

Procivis ONE provides the most granular error handling, distinguishing between format errors and verification errors at the error code level. This allows the application to present more specific guidance to the user (e.g., "The code should be 6 digits" vs. "The code you entered is incorrect").

The EUDI wallets prevent format errors entirely through UI constraints, which is arguably a better user experience -- the user cannot submit an invalid format, so they never see a format error. However, this approach works only because EUDI restricts TxCode to numeric mode with known lengths.

---

## Strengths and Weaknesses

### EUDI (Android & iOS)

**Strengths:**
- Strict numeric-only validation prevents format errors before submission.
- SDK-internal nonce management eliminates an entire class of application-level bugs.
- Nonce binding to attestation provides session integrity beyond the proof JWT.
- iOS debounce prevents UI flicker during rapid input.

**Weaknesses:**
- Rejecting text mode limits issuer compatibility.
- Opaque nonce handling makes debugging issuance failures harder (cannot inspect nonce state).
- No distinct error codes for different failure types.

### Procivis ONE

**Strengths:**
- Supports both numeric and text TxCode modes, maximizing issuer compatibility.
- Distinct error codes (`BR_0169`, `BR_0170`) enable precise error reporting.
- Security level check after TxCode validation ensures key storage meets requirements before proceeding.

**Weaknesses:**
- Text mode support increases the surface area for input validation edge cases (character encoding, special characters).
- Length validation at the core engine level (rather than UI level) means the user may submit invalid codes that are rejected after a round trip.
- No documented input debounce may cause rapid validation cycles.

### Affinidi

**Strengths:**
- `FIXED_HOLDER` mode eliminates TxCode entirely for known users, providing a cleaner UX.
- Server-side validation ensures the issuance service is the single source of truth for code correctness.
- Two claim modes (TX_CODE and FIXED_HOLDER) provide flexibility in holder verification strategy.

**Weaknesses:**
- Server-side only validation means every code attempt requires a network round trip.
- No documented text mode support.
- No client-side format validation -- incorrectly formatted codes hit the network before being rejected.

---

## Decision Framework

| If your scenario requires... | Consider... |
|------------------------------|------------|
| Strictest input validation | EUDI (prevents invalid submissions at UI level) |
| Alphanumeric transaction codes | Procivis ONE (supports TEXT mode) |
| Elimination of TxCodes for known users | Affinidi (FIXED_HOLDER mode) |
| Granular error reporting | Procivis ONE (BR_0169 / BR_0170) |
| Simplest nonce management | Any (all implementations encapsulate nonces) |
| Nonce-bound key attestation | EUDI (explicit nonce parameter in attestation API) |

---

**Previous**: [Implementation Details](./implementation-details.md)
**Section index**: [README](./README.md)
