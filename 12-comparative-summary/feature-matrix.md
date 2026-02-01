# Feature Matrix

This page provides a comprehensive comparison of all features across the EUDI Reference Wallet, Procivis One, and Affinidi implementations.

**Legend:**
- **Yes** -- Fully supported and implemented
- **Partial** -- Supported with limitations or caveats
- **No** -- Not supported
- **N/A** -- Not applicable to this implementation's architecture

---

## Protocol Support

| Feature | EUDI | Procivis | Affinidi |
|---------|------|----------|----------|
| OID4VCI (credential issuance) | Yes (Draft 13+) | Yes (Draft 13, v1.0, multi-draft) | Yes (OID4VCI-based) |
| OID4VP (credential presentation) | Yes | Yes | Partial (verifier SDK) |
| Pre-Authorized Code flow | Yes | Yes | Yes |
| Authorization Code flow | Yes | Yes | No |
| PAR (Pushed Authorization Requests) | Yes | Yes | No |
| DPoP (Proof-of-Possession tokens) | Yes | Yes | No |
| PKCE | Yes | Yes | N/A |
| Deferred issuance | Yes | Yes | No |
| Batch issuance | Yes | Yes | No |
| Notification endpoint | Yes | Yes | No |
| TxCode (transaction code / PIN) | Yes | Yes | Yes |

## Credential Formats

| Feature | EUDI | Procivis | Affinidi |
|---------|------|----------|----------|
| mDoc (ISO 18013-5) | Yes | Yes | No |
| SD-JWT | Yes | Yes | No |
| W3C VC (JSON-LD) | No | Yes | Yes |
| JWT VC | No | Yes | No |
| Multiple formats simultaneously | Yes (mDoc + SD-JWT) | Yes (all four) | No (JSON-LD only) |

## Cryptographic Algorithms

| Feature | EUDI | Procivis | Affinidi |
|---------|------|----------|----------|
| ES256 (ECDSA P-256) | Yes | Yes | No |
| EdDSA (Ed25519) | No | Yes | No |
| ECDSA secp256k1 | No | No | Yes |
| CRYSTALS-DILITHIUM 3 (post-quantum) | No | Yes | No |
| BBS+ (zero-knowledge proofs) | No | Yes | No |
| SHA-256 | Yes | Yes | Yes |

## Key Storage

| Feature | EUDI | Procivis | Affinidi |
|---------|------|----------|----------|
| Hardware-backed keys (TEE/Secure Enclave) | Yes | Yes | No |
| Strongbox support (Android) | Yes | Yes | No |
| Secure Enclave support (iOS) | Yes | Yes | No |
| Software key storage (fallback) | Yes (dev only) | Yes | N/A |
| Remote Secure Element (HSM) | No | Yes (UBIQU_RSE) | No |
| Cloud key management | No | No | Yes (Vault) |
| Key attestation | Yes | Yes | No |

## Transport

| Feature | EUDI | Procivis | Affinidi |
|---------|------|----------|----------|
| HTTPS (online issuance) | Yes | Yes | Yes |
| HTTPS (online presentation) | Yes | Yes | Yes |
| BLE (offline presentation) | Yes | Yes | No |
| NFC | Partial | Yes | No |
| MQTT | No | Yes | No |
| Deep links / Custom URL schemes | Yes | Yes | Yes |
| QR code scanning | Yes | Yes | Yes |

## Attestation

| Feature | EUDI | Procivis | Affinidi |
|---------|------|----------|----------|
| Wallet attestation | Yes | Yes | No |
| Key attestation (hardware) | Yes | Yes | No |
| Platform app attestation (Play Integrity / App Attest) | Yes | Partial | No |
| Wallet instance attestation | Yes | Yes | No |

## Credential Lifecycle

| Feature | EUDI | Procivis | Affinidi |
|---------|------|----------|----------|
| Credential issuance | Yes | Yes | Yes |
| Credential storage | Yes (Room/SwiftData) | Yes (encrypted DB) | Yes (Vault) |
| Credential presentation | Yes | Yes | Yes |
| Selective disclosure (SD-JWT) | Yes | Yes | No |
| Selective disclosure (mDoc) | Yes | Yes | No |
| Selective disclosure (BBS+) | No | Yes | No |
| Credential rotation | Yes (RotateUse policy) | Yes | No |
| One-time use credentials | Yes (OneTimeUse policy) | Yes | No |
| Credential deletion | Yes | Yes | Yes |
| Revocation tracking | Yes (RevokedDocument) | Yes | Partial |
| Transaction logging | Yes (TransactionLog) | Yes | Partial |

## Architecture

| Feature | EUDI | Procivis | Affinidi |
|---------|------|----------|----------|
| Native mobile app | Yes (Android + iOS separate) | Yes (cross-platform core) | No (SDK/API) |
| Cross-platform core | No (platform-specific) | Yes (Rust-based One Core) | Yes (cloud service) |
| Cloud service component | No | Optional (RSE) | Yes (primary) |
| Open source | Yes | Partial (SDK open, core proprietary) | Partial (SDKs open) |
| Multi-language SDK | No | Partial | Yes (TypeScript, Python, Kotlin, Swift) |
| Offline operation | Yes | Yes (local keys) | No |
| Multi-device credential access | No | Partial (RSE mode) | Yes |

## Error Handling

| Feature | EUDI | Procivis | Affinidi |
|---------|------|----------|----------|
| Structured error responses | Yes | Yes | Yes |
| Error recovery (retry logic) | Partial | Yes | Yes |
| Deferred issuance fallback | Yes | Yes | No |
| Graceful degradation (no hardware) | Yes (software fallback) | Yes (software fallback) | N/A |
| User-facing error messages | Yes | Yes | Yes |

## Standards Compliance

| Feature | EUDI | Procivis | Affinidi |
|---------|------|----------|----------|
| eIDAS 2.0 alignment | Yes (primary target) | Yes (supported) | No |
| HAIP compliance | Yes | Yes | No |
| ISO 18013-5 compliance | Yes | Yes | No |
| W3C VC Data Model | Partial (via SD-JWT VC) | Yes | Yes |
| DID Core support | Partial | Yes (multiple methods) | Yes (multiple methods) |

---

## Summary Heatmap

The following provides an at-a-glance view of implementation strength by category:

| Category | EUDI | Procivis | Affinidi |
|----------|------|----------|----------|
| Protocol completeness | Strong | Strongest | Basic |
| Credential format coverage | Good (2 formats) | Best (4 formats) | Limited (1 format) |
| Cryptographic breadth | Focused (ES256) | Broadest (5 algorithms) | Focused (secp256k1) |
| Hardware security | Strong | Strong + RSE option | None (cloud) |
| Transport options | Good | Best (5 transports) | Basic (2 transports) |
| Offline capability | Strong | Strong | None |
| Multi-device support | None | Partial | Strong |
| Developer experience | Moderate (native) | Moderate (cross-platform) | Easiest (API/SDK) |
| Time to market | Moderate | Moderate | Fastest |
| Regulatory readiness | Strongest | Strong | Limited |
