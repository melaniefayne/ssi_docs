# Cryptographic Algorithms

This page documents every cryptographic algorithm used across the EUDI, Procivis, and Affinidi SSI implementations. For each algorithm, we explain: what problem it solves, how it works conceptually, what is signed or encrypted or hashed, and who controls which keys.

---

## ES256 (ECDSA with P-256)

### Used By

EUDI, Procivis

### What Problem It Solves

ES256 provides digital signatures: a way for one party to sign data such that any other party can verify the signature was produced by the holder of a specific private key, without learning the private key itself. In SSI, this is used to prove that a credential holder controls the key bound to their credential.

### How It Works

ES256 is the ECDSA (Elliptic Curve Digital Signature Algorithm) instantiated with the NIST P-256 curve (also known as secp256r1 or prime256v1).

**Curve parameters:**
- Curve: P-256 (NIST) / secp256r1
- Key size: 256-bit private key, 512-bit public key (uncompressed)
- Signature size: 64 bytes (two 32-byte integers, r and s)
- Security level: 128-bit equivalent

**Signing process:**
1. Hash the message using SHA-256 to produce a 256-bit digest
2. Generate a random nonce k (critical: k must be unique per signature)
3. Compute the curve point (x, y) = k * G (where G is the generator point)
4. Compute r = x mod n (n is the curve order)
5. Compute s = k^(-1) * (hash + r * privateKey) mod n
6. The signature is (r, s)

**Verification process:**
1. Hash the message using SHA-256
2. Compute intermediate values using the public key, r, s, and the hash
3. Check that the computed point matches r

### What Is Signed

| Context | Data Signed | Signer |
|---------|-------------|--------|
| JWT proof of possession | JWT header + payload (base64url-encoded) | Wallet (holder) |
| DPoP token | DPoP JWT (htm, htu, iat, jti, ath) | Wallet (holder) |
| Credential issuance | Credential payload | Issuer |
| Presentation response | VP Token | Wallet (holder) |

### Who Controls Which Keys

| Role | Key | Storage |
|------|-----|---------|
| Wallet (holder) | Private key for credential binding | Hardware (TEE/Secure Enclave) |
| Wallet (holder) | Public key sent to issuer during issuance | Included in credential proof |
| Issuer | Private key for signing credentials | Issuer's HSM or key management system |
| Issuer | Public key published in metadata | Issuer's JWKS endpoint |

### Why P-256

P-256 is the most widely supported elliptic curve:
- Supported by iOS Secure Enclave (the only EC curve supported)
- Supported by Android Keystore (TEE and Strongbox)
- Supported by all major TLS implementations
- Required by HAIP (High Assurance Interoperability Profile) for OID4VCI
- Mandated by many government security standards (FIPS 186-4)

---

## EdDSA (Ed25519)

### Used By

Procivis

### What Problem It Solves

EdDSA solves the same fundamental problem as ECDSA -- digital signatures -- but with a different curve and algorithm design that provides deterministic signatures, faster operations, and resistance to certain implementation pitfalls.

### How It Works

Ed25519 is the EdDSA (Edwards-curve Digital Signature Algorithm) instantiated with the Curve25519 Edwards form (Ed25519).

**Curve parameters:**
- Curve: Curve25519 (Bernstein)
- Key size: 256-bit private key, 256-bit public key (compressed)
- Signature size: 64 bytes
- Security level: 128-bit equivalent

**Key difference from ECDSA:** Ed25519 signatures are **deterministic**. The nonce is derived from the private key and the message via a hash, not from a random number generator. This eliminates an entire class of vulnerabilities where a poor random number generator leads to key recovery (the failure mode that compromised the PlayStation 3 ECDSA keys).

**Signing process:**
1. Derive nonce r = SHA-512(private_key_prefix || message) -- deterministic
2. Compute R = r * B (where B is the base point)
3. Compute S = r + SHA-512(R || public_key || message) * private_key
4. The signature is (R, S) -- 64 bytes

### What Is Signed

| Context | Data Signed | Signer |
|---------|-------------|--------|
| Credential signing | Verifiable Credential payload | Issuer (Procivis) |
| DID document authentication | DID document proof | DID controller |
| Proof of possession | Cryptographic proof payload | Wallet (holder) |

### Who Controls Which Keys

| Role | Key | Storage |
|------|-----|---------|
| Issuer | Ed25519 private key | Procivis server key management |
| Holder | Ed25519 private key (if supported by device) | Software key store (not Secure Enclave) |
| Verifier | Uses issuer's public key to verify | Resolved from DID document or JWKS |

### Limitations

- **Not supported by iOS Secure Enclave** -- iOS hardware only supports P-256. Ed25519 keys must be stored in software on iOS.
- **Not supported by all Android Strongbox implementations** -- TEE support varies by vendor.
- **Not specified in HAIP** -- the High Assurance Interoperability Profile requires ES256, not EdDSA.

For these reasons, Ed25519 is used primarily in server-side signing (where HSM support is available) rather than in device-bound key operations.

---

## EcdsaSecp256k1Signature2019

### Used By

Affinidi

### What Problem It Solves

This is a signature suite for signing Verifiable Credentials using ECDSA with the secp256k1 curve, expressed as a JSON-LD Data Integrity proof. It enables VC verification using the same cryptographic infrastructure as Bitcoin and Ethereum.

### How It Works

**Curve parameters:**
- Curve: secp256k1 (Koblitz curve, used in Bitcoin/Ethereum)
- Key size: 256-bit private key, 512-bit public key (uncompressed)
- Signature size: 64 bytes (r, s) + recovery byte
- Security level: 128-bit equivalent

The algorithm is ECDSA, the same as ES256, but on a different curve. The secp256k1 curve was chosen for Bitcoin because of its efficient computation properties and its Koblitz-curve structure, not because of a NIST recommendation.

**Signature process for JSON-LD Data Integrity:**
1. Canonicalize the JSON-LD document using the URDNA2015 algorithm (ensures deterministic serialization)
2. Hash the canonicalized document using SHA-256
3. Sign the hash using ECDSA with the secp256k1 private key
4. Embed the signature in the credential's `proof` field

### What Is Signed

| Context | Data Signed | Signer |
|---------|-------------|--------|
| Verifiable Credential | Canonicalized JSON-LD credential | Issuer (Affinidi) |
| Verifiable Presentation | Canonicalized JSON-LD presentation | Holder |

### Who Controls Which Keys

| Role | Key | Storage |
|------|-----|---------|
| Issuer | secp256k1 private key | Affinidi cloud key management |
| Holder | secp256k1 private key | Affinidi Vault |
| Verifier | Resolves issuer's public key from DID | DID resolution |

### Why secp256k1

Affinidi's choice of secp256k1 aligns with the decentralized identity ecosystem that emerged from blockchain communities:
- Compatible with did:key, did:web, and did:ethr DID methods
- Existing tooling from Ethereum/Bitcoin ecosystems
- Broad library support in JavaScript/TypeScript (Affinidi's primary SDK language)

**Trade-off:** secp256k1 is not supported by hardware security modules on mobile devices (neither iOS Secure Enclave nor Android Strongbox). This is one reason Affinidi uses cloud-based key storage rather than device-bound keys.

---

## CRYSTALS-DILITHIUM 3

### Used By

Procivis

### What Problem It Solves

CRYSTALS-DILITHIUM is a post-quantum signature scheme. It provides digital signatures that are secure against attacks by both classical computers and quantum computers. Current elliptic curve algorithms (ES256, Ed25519, secp256k1) can be broken by a sufficiently powerful quantum computer running Shor's algorithm.

### How It Works

DILITHIUM is a lattice-based signature scheme. Unlike elliptic curve cryptography (which relies on the difficulty of the discrete logarithm problem on elliptic curves), DILITHIUM relies on the hardness of the Module Learning With Errors (Module-LWE) problem.

**Parameters (DILITHIUM Level 3):**
- Security level: NIST Level 3 (equivalent to AES-192)
- Public key size: 1,952 bytes
- Signature size: 3,293 bytes
- Private key size: 4,000 bytes

**Conceptual signing process:**
1. Sample a random masking vector y
2. Compute w = A * y (where A is a public matrix)
3. Compute the challenge c = H(message || w)
4. Compute the response z = y + c * s (where s is the secret key)
5. Check that z has small coefficients (rejection sampling -- retry if not)
6. The signature is (z, c)

### Comparison to Classical Algorithms

| Property | ES256 / Ed25519 | DILITHIUM 3 |
|----------|----------------|-------------|
| Quantum-resistant | No | Yes |
| Public key size | 32-64 bytes | 1,952 bytes |
| Signature size | 64 bytes | 3,293 bytes |
| Key generation speed | Fast | Moderate |
| Signing speed | Fast | Moderate |
| Verification speed | Fast | Fast |
| Standardization | Mature (decades) | NIST PQC Standard (2024) |

### What Is Signed

| Context | Data Signed | Signer |
|---------|-------------|--------|
| Verifiable Credential | Credential payload | Issuer (Procivis) |
| Future-proofed presentations | VP Token | Wallet (holder) |

### Who Controls Which Keys

| Role | Key | Storage |
|------|-----|---------|
| Issuer | DILITHIUM private key | Procivis server HSM or key management |
| Holder | DILITHIUM private key (if supported) | Software key store (no hardware support yet) |

### Limitations

- **No hardware support**: Neither iOS Secure Enclave, Android TEE, nor Strongbox support DILITHIUM. Keys must be stored in software.
- **Large sizes**: The significantly larger key and signature sizes increase bandwidth and storage requirements.
- **Ecosystem maturity**: Tooling, libraries, and interoperability are still maturing.
- **Hybrid approach recommended**: Use DILITHIUM alongside classical algorithms during the transition period, not as a sole signature scheme.

---

## BBS+ Signatures

### Used By

Procivis

### What Problem It Solves

BBS+ enables **zero-knowledge proofs** and **selective disclosure** at the cryptographic level. Unlike SD-JWT (which achieves selective disclosure through a hash-based mechanism layered on top of JWTs), BBS+ provides selective disclosure as a native property of the signature scheme itself.

### How It Works

BBS+ is a signature scheme over a set of messages (claims). The signer signs all messages at once, producing a single signature. The holder can then derive a **proof** that reveals only a subset of the signed messages, without revealing the others or the original signature.

**Key operations:**

1. **Sign** (Issuer): Signs a vector of messages [m1, m2, ..., mn] producing signature sigma
2. **Derive Proof** (Holder): Given sigma and the messages, produces a zero-knowledge proof that reveals only selected messages [mi, mj, ...] while hiding the rest
3. **Verify Proof** (Verifier): Verifies the proof against the issuer's public key, confirming the revealed messages were part of the original signed set

```
Issuer signs: [name, birthdate, address, nationality]
                           |
                    Full signature (sigma)
                           |
Holder derives proof revealing only: [nationality]
                           |
Verifier verifies: nationality is authentic, learns nothing else
```

### Comparison to SD-JWT Selective Disclosure

| Property | BBS+ | SD-JWT |
|----------|------|--------|
| Selective disclosure | Cryptographic (native) | Hash-based (layered) |
| Zero-knowledge proofs | Yes | No |
| Predicate proofs (e.g., "age > 18") | Possible with extensions | Not natively supported |
| Unlinkability | Yes (derived proofs are unlinkable) | Partial (depends on implementation) |
| Issuer complexity | Higher (multi-message signing) | Lower (standard JWT signing) |
| Ecosystem support | Limited | Broad and growing |

### What Is Signed

| Context | Data Signed | Signer |
|---------|-------------|--------|
| Credential issuance | Vector of credential claims | Issuer (Procivis) |
| Derived proof for presentation | Subset of claims + ZKP | Wallet (holder) |

### Who Controls Which Keys

| Role | Key | Storage |
|------|-----|---------|
| Issuer | BBS+ private key (signing key) | Procivis server |
| Issuer | BBS+ public key (verification key) | Published in DID document or metadata |
| Holder | No separate holder key needed for ZKP derivation | N/A |
| Verifier | Uses issuer's public key | DID resolution |

### Limitations

- **Not supported by mobile hardware**: No Secure Enclave or Keystore support
- **Not specified in HAIP or OID4VCI**: Not part of the current interoperability profiles
- **Limited library support**: Fewer production-grade implementations than ECDSA or EdDSA
- **Performance**: Proof generation and verification are computationally heavier than ECDSA

---

## SHA-256

### Used By

All implementations (EUDI, Procivis, Affinidi)

### What Problem It Solves

SHA-256 is a cryptographic hash function. It produces a fixed-size (256-bit) digest from arbitrary-length input. The hash is deterministic, collision-resistant, and pre-image resistant. It does not provide signatures or encryption -- it provides **integrity** and is a building block for other cryptographic operations.

### How It Works

SHA-256 processes input in 512-bit blocks through 64 rounds of mixing, shifting, and modular addition. The output is a 256-bit (32-byte) digest.

**Properties:**
- **Deterministic**: Same input always produces the same hash
- **Collision-resistant**: Computationally infeasible to find two inputs with the same hash
- **Pre-image resistant**: Given a hash, computationally infeasible to find the input
- **Avalanche effect**: Changing one bit of input changes approximately 50% of output bits

### Where SHA-256 Is Used in SSI

| Context | What Is Hashed | Purpose |
|---------|---------------|---------|
| ECDSA signing (ES256) | Message being signed | Produces the digest that is actually signed |
| SD-JWT disclosures | Salt + claim name + claim value | Creates the disclosure hash included in the JWT |
| Content integrity | Credential payloads | Ensures data has not been tampered with |
| Nonce generation | Random input | Produces c_nonce values for replay prevention |
| PKCE code challenge | Code verifier (random string) | S256 method: challenge = BASE64URL(SHA-256(verifier)) |
| mDoc MSO | Credential data elements | Mobile Security Object integrity hashes |

### Who Controls What

SHA-256 is a keyless operation -- anyone can hash any data. It is a public function with no secret input. Its security properties come from the mathematical structure of the algorithm, not from any key material.

---

## Algorithm Support Matrix

| Algorithm | EUDI | Procivis | Affinidi | Hardware Support | Quantum-Safe |
|-----------|------|----------|----------|-----------------|-------------|
| ES256 (P-256) | Yes | Yes | No | TEE, Strongbox, Secure Enclave | No |
| EdDSA (Ed25519) | No | Yes | No | Limited TEE | No |
| ECDSA secp256k1 | No | No | Yes | None (mobile) | No |
| CRYSTALS-DILITHIUM 3 | No | Yes | No | None | Yes |
| BBS+ | No | Yes | No | None | No |
| SHA-256 | Yes | Yes | Yes | Accelerated on most hardware | Yes (with larger output) |
