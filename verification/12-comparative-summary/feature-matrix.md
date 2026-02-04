# Feature Matrix

Comprehensive side-by-side comparison of verification features across implementations.

---

## Protocol Support

| Feature | EUDI | Procivis | Affinidi |
|---------|:----:|:--------:|:--------:|
| OID4VP | ✓ v1.0 | ✓ | ✓ |
| ISO 18013-5 | ✓ | ✓ | ✗ |
| SIOPv2 | ✓ | - | - |
| DIF PEX v1 | ✓ (via DCQL) | ✓ | ✓ |
| DIF PEX v2 | ✓ (via DCQL) | ✓ | - |
| DCQL | ✓ | - | - |

---

## Credential Formats

| Format | EUDI | Procivis | Affinidi |
|--------|:----:|:--------:|:--------:|
| SD-JWT-VC | ✓ | ✓ | ✓ |
| MSO-MDOC | ✓ | ✓ | - |
| W3C VC (JWT) | - | ✓ | ✓ |
| W3C VC (JSON-LD) | - | ✓ | ✓ |

---

## Transport Modes

| Mode | EUDI | Procivis | Affinidi |
|------|:----:|:--------:|:--------:|
| HTTP (remote) | ✓ | ✓ | ✓ |
| Deep links | ✓ | ✓ | - |
| BLE | ✓ | ✓ | ✗ |
| NFC | ✓ | ✓ | ✗ |
| WebSocket | - | - | ✓ |
| Redirect | ✓ | ✓ | ✓ |

---

## Client ID Schemes

| Scheme | EUDI | Procivis | Affinidi |
|--------|:----:|:--------:|:--------:|
| redirect_uri | ✓ | ✓ | ✓ |
| x509_san_dns | ✓ | - | - |
| x509_san_hash | ✓ | - | - |
| DID | ✓ | ✓ | - |
| verifier_attestation | - | - | - |

---

## Response Modes

| Mode | EUDI | Procivis | Affinidi |
|------|:----:|:--------:|:--------:|
| direct_post | ✓ | ✓ | ✓ |
| direct_post.jwt | ✓ | - | - |
| fragment | ✓ | - | - |

---

## Key Management

| Feature | EUDI | Procivis | Affinidi |
|---------|:----:|:--------:|:--------:|
| Device keystore | ✓ | - | - |
| Secure Enclave (iOS) | ✓ | - | - |
| Android Keystore | ✓ | - | - |
| StrongBox support | ✓ | - | - |
| Remote Secure Element | - | ✓ | - |
| Cloud HSM | - | - | ✓ |
| Key attestation | ✓ | - | - |

---

## Authentication

| Feature | EUDI | Procivis | Affinidi |
|---------|:----:|:--------:|:--------:|
| Biometric (device) | ✓ | - | - |
| PIN | ✓ | ✓ | - |
| RSE PIN | - | ✓ | - |
| Vault authentication | - | - | ✓ |

---

## Selective Disclosure

| Feature | EUDI | Procivis | Affinidi |
|---------|:----:|:--------:|:--------:|
| Claim-level selection | ✓ | ✓ | ✓ |
| UI claim toggles | ✓ | ✓ | ✓ |
| limit_disclosure | ✓ | ✓ | ✓ |
| Optional claims | ✓ | ✓ | ✓ |

---

## Verifier Trust

| Feature | EUDI | Procivis | Affinidi |
|---------|:----:|:--------:|:--------:|
| Certificate validation | ✓ | - | - |
| Trust store | ✓ | - | - |
| Verifier display name | ✓ | ✓ | ✓ |
| Trust indicator UI | ✓ | - | ✓ |

---

## Platform Support

| Platform | EUDI | Procivis | Affinidi |
|----------|:----:|:--------:|:--------:|
| Android | ✓ (native) | ✓ (RN) | ✓ (web SDK) |
| iOS | ✓ (native) | ✓ (RN) | ✓ (web SDK) |
| Web | - | - | ✓ |
| Server SDK | - | - | ✓ |

---

## Developer Experience

| Aspect | EUDI | Procivis | Affinidi |
|--------|:----:|:--------:|:--------:|
| Open source | ✓ | Partial | SDK only |
| Documentation | ✓ | ✓ | ✓ |
| Sample apps | ✓ | ✓ | ✓ |
| Portal/UI config | - | ✓ | ✓ |
| PEX editor | - | - | ✓ |

---

## Offline Capabilities

| Feature | EUDI | Procivis | Affinidi |
|---------|:----:|:--------:|:--------:|
| Receive request (QR) | ✓ | ✓ | ✗ |
| Sign presentation | ✓ | ✗ | ✗ |
| Send via BLE | ✓ | ✓ | ✗ |
| Send via NFC | ✓ | ✓ | ✗ |

---

## Error Handling

| Feature | EUDI | Procivis | Affinidi |
|---------|:----:|:--------:|:--------:|
| No matching credentials | ✓ | ✓ | ✓ |
| Revoked credentials | ✓ | ✓ | ✓ |
| Partial matches | ✓ | ✓ | ✓ |
| Network errors | ✓ | ✓ | ✓ |
| RSE errors | - | ✓ | - |

---

## Legend

- ✓ = Fully supported
- - = Not supported or not documented
- ✗ = Explicitly not supported
