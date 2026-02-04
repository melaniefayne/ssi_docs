# Credential Request -- Comparative Analysis

This document provides a critical comparison of how EUDI Android, EUDI iOS, Procivis ONE, and Affinidi construct and send credential requests, with particular focus on draft version support, batch issuance, and HAIP compliance.

---

## Draft Support Matrix

| Implementation | Draft 13 | Draft 13 (swiyu) | v1.0 Final | v1.0 Final + HAIP |
|----------------|----------|-------------------|------------|-------------------|
| EUDI Android | No | No | **Yes** | Partial (moving toward full) |
| EUDI iOS | No | No | **Yes** | Partial (moving toward full) |
| Procivis ONE | **Yes** | **Yes** | **Yes** | **Yes** |
| Affinidi | No | No | No (proprietary flow) | No |

### Analysis

**EUDI Android and iOS** target v1.0 Final exclusively. This is a deliberate design choice: the EUDI Reference Wallet is intended to demonstrate compliance with the European Digital Identity framework, which mandates v1.0 Final. There is no backward compatibility with Draft 13 issuers. If a Draft 13 issuer presents a credential offer using the `credentials` array (instead of `credential_configuration_ids`), the EUDI wallet will fail to parse it.

**Procivis ONE** stands alone in supporting all four protocol variants. This multi-draft capability is architecturally significant:

- It enables Procivis to operate across the entire OID4VCI ecosystem without requiring issuer migration.
- Each protocol variant has its own configuration, including encryption parameters, ensuring correct behavior regardless of the issuer's version.
- The swiyu-specific variant demonstrates national/regional profiling of the base specification.
- HAIP support positions Procivis for eIDAS compliance scenarios.

**Affinidi** does not directly implement the OID4VCI credential request specification in the same sense as the other implementations. The Vault's credential claim flow is based on pre-authorized code exchange, but the specific request payload format follows Affinidi's own service conventions rather than strict OID4VCI compliance with either draft.

---

## Single-Draft vs. Multi-Draft: Interoperability Implications

### Single-Draft Approach (EUDI)

**Advantages:**
- Simpler implementation and testing -- only one code path for credential request construction.
- Clear compliance story -- the wallet is definitively v1.0 Final compliant.
- No ambiguity in protocol behavior -- every issuer interaction follows the same specification.

**Disadvantages:**
- Cannot interoperate with Draft 13 issuers. This is a real limitation during the transition period.
- If an issuer has not migrated to v1.0 Final, the EUDI wallet cannot obtain credentials from it.
- Forces the ecosystem toward v1.0 Final adoption, which may be premature for some deployments.

### Multi-Draft Approach (Procivis)

**Advantages:**
- Maximum interoperability -- works with issuers running any supported draft version.
- Enables gradual ecosystem migration -- does not force issuers to upgrade on any timeline.
- Can bridge between Draft 13 and v1.0 Final ecosystems in the same wallet.
- National ecosystem profiling (swiyu) without losing base specification compatibility.

**Disadvantages:**
- Higher implementation complexity -- four protocol handlers with distinct field naming and behaviors.
- Larger testing surface -- each protocol variant requires dedicated testing against compliant issuers.
- Risk of version detection errors -- if the SDK incorrectly identifies the issuer's version, the request will fail.
- Maintenance burden -- specification updates must be applied to each variant independently.

---

## Batch Issuance Comparison

| Aspect | EUDI Android | EUDI iOS | Procivis ONE | Affinidi |
|--------|-------------|---------|-------------|---------|
| Batch issuance supported | Yes | Yes | No | No |
| Configurable batch size | Yes (`numberOfCredentials`) | Yes (`numberOfCredentials`) | N/A | N/A |
| Default policy | `rotateUse` (1 credential) | `.rotateUse` (1 credential) | Single credential | Single credential |
| One-time-use policy | `OneTimeUse` (configurable, e.g., 10) | `.oneTimeUse` (configurable, e.g., 10) | Not available | Not available |
| PID batch size | 10 (typical) | 10 (typical) | 1 | 1 |
| Proofs per request | Multiple (`proofs` array) | Multiple (`proofs` array) | Single (`proof`) | Single (`proof`) |

### Analysis

**EUDI wallets** implement batch issuance as a first-class feature. The credential policy system (`rotateUse` / `oneTimeUse`) provides a clean abstraction for controlling batch behavior. For PID credentials, the one-time-use policy with 10 instances is the recommended configuration, ensuring the user has a pool of credentials available for presentations without repeated issuance round-trips.

The batch approach has clear advantages for privacy: each presentation uses a different credential instance with a different key, preventing verifiers from correlating presentations across contexts based on key material.

**Procivis ONE and Affinidi** issue single credentials per request. For Procivis, this means one-time-use credential policies would require 10 separate issuance requests to achieve the same pool as a single EUDI batch request. The protocol overhead (10 token exchanges and 10 credential requests vs. 1 token exchange and 1 credential request with 10 proofs) is significant, particularly for deployments with high issuance volume.

---

## Credential Format Support in Requests

| Format | EUDI Android | EUDI iOS | Procivis ONE | Affinidi |
|--------|-------------|---------|-------------|---------|
| `mso_mdoc` (ISO 18013-5) | Yes | Yes | Yes | No |
| `vc+sd-jwt` (SD-JWT VC) | Yes | Yes | Yes | No |
| `ldp_vc` (JSON-LD VC) | No | No | Partial | Yes (primary) |
| `jwt_vc_json` (JWT VC) | No | No | Partial | No |

### Analysis

**EUDI wallets** focus on the two formats mandated by the EU Digital Identity framework: mDoc (for mobile driving licenses and similar documents) and SD-JWT VC (for Verifiable Credentials with selective disclosure). This is a deliberate scope limitation aligned with eIDAS requirements.

**Procivis ONE** supports mDoc and SD-JWT VC for EUDI ecosystem compatibility, with additional support for JSON-LD and JWT VC formats for broader interoperability. The format selection is determined by the issuer's metadata and the credential configuration.

**Affinidi** uses JSON-LD VCs exclusively. This is a fundamental format divergence from the EUDI ecosystem and means Affinidi-issued credentials cannot be directly consumed by EUDI-compliant verifiers without format translation.

---

## HAIP Profile Requirements and Compliance

The High Assurance Interoperability Profile (HAIP) adds mandatory requirements on top of OID4VCI v1.0 Final. The following table assesses each implementation against HAIP requirements:

| HAIP Requirement | EUDI Android | EUDI iOS | Procivis ONE | Affinidi |
|-----------------|-------------|---------|-------------|---------|
| PAR mandatory | Yes | Yes | Yes (HAIP variant) | No |
| DPoP mandatory | Yes | Yes | Yes (HAIP variant) | No |
| Hardware key attestation | Yes (Android Keystore) | Yes (Secure Enclave) | Partial (depends on signing mode) | No |
| ES256 required | Yes | Yes | Yes | No (secp256k1) |
| v1.0 Final base | Yes | Yes | Yes (HAIP variant) | No |
| mso_mdoc support | Yes | Yes | Yes | No |
| vc+sd-jwt support | Yes | Yes | Yes | No |

### Analysis

**EUDI Android and iOS** are the closest to full HAIP compliance. Both implement PAR, DPoP, hardware key attestation (via Android Keystore / iOS Secure Enclave), and ES256. Remaining gaps relate to specific attestation format requirements and evolving HAIP specifications.

**Procivis ONE** achieves HAIP compliance through its dedicated `OPENID4VCI_FINAL1_HAIP` protocol variant. When using this variant, the SDK enforces PAR, DPoP, and ES256. Hardware key attestation depends on the signing mode: device Secure Element signing satisfies the requirement; RSE signing may or may not, depending on the RSE's certification level under eIDAS.

**Affinidi** does not meet HAIP requirements. The secp256k1 curve, JSON-LD format, and cloud-based key management are all outside the HAIP specification. This does not diminish Affinidi's utility in its target ecosystem (decentralized identity) but means it is not suitable for eIDAS/EUDI compliance scenarios.

---

## Decision Framework: When to Use Which Draft

### Use v1.0 Final When:

- Building for the European Digital Identity ecosystem (eIDAS, EUDI).
- Compliance with ratified standards is a requirement.
- HAIP profile features (PAR, DPoP, hardware key attestation) are needed.
- The deployment is greenfield (no legacy Draft 13 infrastructure).
- Long-term maintenance cost minimization is a priority.

### Use Draft 13 When:

- Integrating with existing Draft 13 issuers that have not migrated.
- Deploying in the Swiss swiyu ecosystem.
- Supporting pilots or production systems built before v1.0 ratification.
- The issuer ecosystem is fixed and migration is not planned.

### Support Both When:

- Building a wallet that must work across diverse issuer ecosystems.
- Operating during the transition period where both versions coexist.
- Maximizing the number of issuers the wallet can interact with.
- Serving as a bridge between legacy and standards-compliant deployments.

---

## Strengths and Weaknesses

| Implementation | Strengths | Weaknesses |
|----------------|-----------|------------|
| **EUDI Android** | Standards-compliant (v1.0 Final). Batch issuance with configurable policies. Moving toward full HAIP compliance. Strong SDK encapsulation. | No Draft 13 backward compatibility. Cannot interoperate with pre-Final issuers. Single-format focus (mDoc + SD-JWT only). |
| **EUDI iOS** | Same v1.0 Final compliance as Android. Batch issuance parity. Clean async API. | Same limitations as Android: no Draft 13 support, limited format coverage. |
| **Procivis ONE** | Multi-draft support is a unique and significant capability. HAIP-ready. Broadest protocol coverage. National ecosystem profiling (swiyu). | No batch issuance. Higher complexity and maintenance burden. Single credential per request limits one-time-use efficiency. |
| **Affinidi** | Simple pre-authorized flow. Clean single-credential model. JSON-LD VC support for decentralized identity ecosystem. | No OID4VCI draft compliance (neither Draft 13 nor Final). No batch issuance. No HAIP support. Single-use claim links limit scalability. secp256k1 incompatible with EUDI ecosystem. |
