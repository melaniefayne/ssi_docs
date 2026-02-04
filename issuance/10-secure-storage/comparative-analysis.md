# Secure Storage: Comparative Analysis

## Cross-Implementation Comparison

| Feature | EUDI (Android) | EUDI (iOS) | Procivis (Android) | Procivis (iOS) | Affinidi |
|---------|---------------|------------|--------------------|--------------------|----------|
| **Hardware backing** | TEE / Strongbox | Secure Enclave | TEE / Strongbox | Secure Enclave | None (cloud) |
| **Key extractability** | Non-extractable | Non-extractable | Non-extractable | Non-extractable | Cloud-managed |
| **Credential store** | Room Database | SwiftData | Encrypted DB | Encrypted DB | Cloud Vault |
| **Encryption at rest** | Room + Keystore-bound key | SwiftData + Data Protection | App-level encryption | App-level encryption | Cloud encryption |
| **Credential policies** | OneTimeUse, RotateUse | OneTimeUse, RotateUse | Configurable per credential type | Configurable per credential type | N/A |
| **Revocation tracking** | RevokedDocument table | SDRevokedDocument model | Internal DB | Internal DB | Cloud-managed |
| **Transaction logging** | TransactionLog table | SDTransactionLog model | Internal DB | Internal DB | Cloud audit |
| **Biometric gating** | Android Keystore biometric binding | Secure Enclave + Face ID/Touch ID | Keystore biometric binding | Secure Enclave biometric | Cloud auth |
| **Offline key operations** | Yes | Yes | Yes (local) / No (RSE) | Yes (local) / No (RSE) | No |
| **Multi-device support** | No | No | No (local) / Yes (RSE) | No (local) / Yes (RSE) | Yes |
| **Backup/restore** | Credentials not backed up (device-bound) | Credentials not backed up | Device-bound (local) | Device-bound (local) | Cloud-native |
| **Remote key management** | No | No | Yes (UBIQU_RSE) | Yes (UBIQU_RSE) | Yes (cloud) |

## Trust Boundary Analysis

Trust boundaries define where security transitions occur -- where data moves from a trusted zone to a less-trusted zone or vice versa. Understanding these boundaries is essential for threat modeling.

### EUDI Trust Boundaries

```
+================================================================+
|  HARDWARE TRUST BOUNDARY (TEE / Secure Enclave)                |
|                                                                |
|  Private Keys                                                  |
|  Signing Operations                                            |
|  Key Attestation                                               |
|                                                                |
+====================+============+==============================+
                     |            ^
              Sign Request     Signature
                     |            |
+====================v============+==============================+
|  APPLICATION TRUST BOUNDARY                                    |
|                                                                |
|  Wallet Application Code                                       |
|  Credential Metadata (Room / SwiftData)                        |
|  Revocation Tracking                                           |
|  Transaction Logs                                              |
|  PIN Storage (Keychain)                                        |
|                                                                |
+====================+============+==============================+
                     |            ^
              Presentation     Credential
                     |            |
+====================v============+==============================+
|  NETWORK TRUST BOUNDARY                                        |
|                                                                |
|  Issuer Communication (OID4VCI)                                |
|  Verifier Communication (OID4VP)                               |
|  TLS Channel                                                   |
|                                                                |
+================================================================+
```

**Key observations:**
- Private keys never cross the hardware trust boundary
- Credential metadata exists in the application trust boundary (protected by OS-level encryption)
- Presentations cross the network trust boundary (protected by TLS and cryptographic proofs)

### Procivis Trust Boundaries (Local Mode)

```
+================================================================+
|  HARDWARE TRUST BOUNDARY (TEE / Secure Enclave)                |
|                                                                |
|  Private Keys                                                  |
|  Signing Operations                                            |
|                                                                |
+====================+============+==============================+
                     |            ^
+====================v============+==============================+
|  SDK ABSTRACTION BOUNDARY (One Core)                           |
|                                                                |
|  Cross-platform Key Storage Interface                          |
|  Credential Management Logic                                   |
|  Protocol Implementations                                      |
|                                                                |
+====================+============+==============================+
                     |            ^
+====================v============+==============================+
|  APPLICATION TRUST BOUNDARY                                    |
|                                                                |
|  Platform-specific UI                                          |
|  Encrypted Database                                            |
|                                                                |
+====================+============+==============================+
                     |            ^
+====================v============+==============================+
|  NETWORK TRUST BOUNDARY                                        |
+================================================================+
```

**Key observation:** Procivis introduces an additional SDK abstraction boundary. The platform-specific code interacts with the One Core SDK, which in turn interacts with hardware. This adds a layer of abstraction but also an additional surface to audit.

### Procivis Trust Boundaries (RSE Mode)

```
+================================================================+
|  REMOTE HARDWARE TRUST BOUNDARY (HSM)                          |
|                                                                |
|  Private Keys                                                  |
|  Signing Operations                                            |
|                                                                |
+====================+============+==============================+
                     |            ^
              Sign Request     Signature
              (encrypted)      (encrypted)
                     |            |
+====================v============+==============================+
|  NETWORK TRUST BOUNDARY (TLS + Mutual Auth)                    |
+====================+============+==============================+
                     |            ^
+====================v============+==============================+
|  APPLICATION TRUST BOUNDARY                                    |
|                                                                |
|  Wallet App + One Core SDK                                     |
|  No local key material                                         |
|                                                                |
+================================================================+
```

**Key observation:** In RSE mode, the hardware trust boundary is remote. The application never has local key material. This shifts the threat model from device compromise to network and HSM compromise.

### Affinidi Trust Boundaries

```
+================================================================+
|  CLOUD TRUST BOUNDARY (Affinidi Vault)                         |
|                                                                |
|  Credentials                                                   |
|  Keys (cloud-managed)                                          |
|  Vault Encryption                                              |
|                                                                |
+====================+============+==============================+
                     |            ^
              API Calls       Responses
              (HTTPS)         (HTTPS)
                     |            |
+====================v============+==============================+
|  NETWORK TRUST BOUNDARY (TLS)                                  |
+====================+============+==============================+
                     |            ^
+====================v============+==============================+
|  APPLICATION TRUST BOUNDARY                                    |
|                                                                |
|  SDK / API Client                                              |
|  Cached Credentials (Edge Profile)                             |
|  Authentication Tokens                                         |
|                                                                |
+================================================================+
```

**Key observation:** Affinidi has no device-level hardware trust boundary. The highest trust boundary is the cloud service. This means the security guarantee depends on the cloud infrastructure rather than the user's device hardware.

## Backup and Restore

Credential backup is a complex topic because device-bound keys, by design, cannot be backed up.

| Scenario | EUDI | Procivis (Local) | Procivis (RSE) | Affinidi |
|----------|------|-------------------|----------------|----------|
| Device loss | Re-issuance required | Re-issuance required | Keys survive (remote) | Credentials survive (cloud) |
| Device upgrade | Re-issuance required | Re-issuance required | Re-authenticate to RSE | Re-authenticate to Vault |
| Factory reset | All credentials lost | All credentials lost | Keys survive (remote) | Credentials survive (cloud) |
| Backup contains keys | No (non-extractable) | No (non-extractable) | N/A (keys are remote) | N/A (keys are cloud) |
| Backup contains metadata | Platform-dependent | Platform-dependent | N/A | N/A |

### The Re-Issuance Trade-Off

Hardware-backed, device-bound keys provide the strongest security guarantee but create a usability challenge: if the device is lost, all credentials must be re-issued. This is by design -- if keys could be backed up, they could be extracted.

Cloud and remote key storage models avoid this problem at the cost of weaker device binding.

## Recommendation Matrix

| Requirement | Recommended Approach |
|-------------|---------------------|
| Maximum security for government ID | EUDI-style: hardware-backed, device-bound, one-time-use policies |
| Enterprise deployment with central key control | Procivis RSE: remote HSM with organizational management |
| Cross-platform with strong device security | Procivis local: One Core SDK with per-platform Secure Element |
| Cloud-first, multi-device access | Affinidi Vault: cloud-managed keys and credentials |
| Offline-capable wallet | EUDI or Procivis (local): all key operations are device-local |
| Regulatory compliance (eIDAS 2.0 WSCD) | EUDI or Procivis (local): hardware-backed keys with attestation |

## Summary

The fundamental trade-off in secure storage is between **device binding** and **portability**:

- **Device-bound keys** (EUDI, Procivis local) provide the strongest security: keys cannot be extracted, copied, or transferred. But credentials are lost if the device is lost.
- **Remote keys** (Procivis RSE) provide organizational control and survive device loss, but require network connectivity for every signing operation.
- **Cloud-managed keys** (Affinidi) provide the best multi-device UX and survive device loss, but offer no hardware-backed assurance on the user's device.

The right choice depends on the threat model, regulatory requirements, and user experience priorities of the deployment.
