# Credential Formats

This page provides a deep-dive into every credential format used across the EUDI, Procivis, and Affinidi SSI implementations. Each format represents a different set of design trade-offs around encoding, signing, selective disclosure, and ecosystem compatibility.

---

## mDoc (ISO 18013-5)

### Used By

EUDI, Procivis

### Overview

mDoc is the credential format defined by ISO 18013-5 for mobile driving licenses (mDL) and extended to other identity documents by ISO 23220. It uses CBOR encoding, a binary format that is compact and efficient for constrained environments. mDoc was designed specifically for offline presentation scenarios (e.g., presenting a license at a traffic stop via BLE or NFC).

### Structure

An mDoc credential consists of:

```
mDoc
  +-- DocType (e.g., "org.iso.18013.5.1.mDL")
  +-- IssuerSigned
  |     +-- NameSpaces
  |     |     +-- "org.iso.18013.5.1"
  |     |           +-- IssuerSignedItem (claim: "family_name", value: "Smith")
  |     |           +-- IssuerSignedItem (claim: "given_name", value: "Alice")
  |     |           +-- IssuerSignedItem (claim: "birth_date", value: "1990-01-15")
  |     |           +-- IssuerSignedItem (claim: "driving_privileges", value: [...])
  |     +-- IssuerAuth (COSE_Sign1)
  |           +-- Mobile Security Object (MSO)
  |                 +-- Digest Algorithm (SHA-256)
  |                 +-- Value Digests (per namespace, per claim)
  |                 +-- Device Key Info (holder's public key)
  |                 +-- Validity Info (signed, validFrom, validUntil)
  +-- DeviceSigned
        +-- DeviceAuth (COSE_Sign1 or COSE_Mac0)
              +-- Signed by holder's device key
```

### Mobile Security Object (MSO)

The MSO is the integrity mechanism for mDoc. Instead of signing the claims directly, the issuer:

1. Computes a SHA-256 hash of each individual claim (IssuerSignedItem)
2. Collects all hashes into a digest map, organized by namespace
3. Signs the digest map (not the claims) using COSE_Sign1

This design enables **selective disclosure**: the holder can present only a subset of claims along with their hashes. The verifier checks that the presented claims hash to values in the signed MSO, confirming their authenticity without seeing the undisclosed claims.

```
Selective Disclosure in mDoc:

Issuer signs MSO containing:
  hash(family_name) = 0xA1B2...
  hash(given_name)  = 0xC3D4...
  hash(birth_date)  = 0xE5F6...
  hash(driving_privileges) = 0x7890...

Holder presents to verifier:
  Claim: birth_date = "1990-01-15"
  MSO: signed digest map (all hashes)

Verifier checks:
  SHA-256("1990-01-15" + salt) == 0xE5F6...  (matches MSO)
  MSO signature valid against issuer's public key
  Result: birth_date is authentic
```

### Namespace-Based Claims

mDoc organizes claims into namespaces. This allows different issuers or standards to define claims without naming collisions:

| Namespace | Standard | Example Claims |
|-----------|----------|---------------|
| `org.iso.18013.5.1` | ISO 18013-5 (mDL) | family_name, given_name, birth_date, driving_privileges |
| `eu.europa.ec.eudi.pid.1` | EU PID | family_name, given_name, birth_date, nationality, issuance_date |

### Encoding: CBOR

mDoc uses CBOR (Concise Binary Object Representation, RFC 8949) rather than JSON. CBOR is:
- **Binary**: More compact than JSON text
- **Schema-flexible**: Supports maps, arrays, byte strings, tagged types
- **Deterministic**: Has a canonical encoding for consistent hashing
- **Efficient**: Faster to parse than JSON, important for constrained devices (BLE, NFC)

### Signing: COSE

mDoc uses COSE (CBOR Object Signing and Encryption, RFC 9052) for cryptographic operations. COSE is the CBOR equivalent of JOSE (JSON Object Signing and Encryption). The MSO is signed using COSE_Sign1 (single-signer signature).

### Key Binding

The MSO includes `DeviceKeyInfo`, which contains the holder's public key. During presentation, the holder signs a `DeviceAuth` structure with the corresponding private key, proving possession of the device key and preventing credential transfer.

---

## SD-JWT (Selective Disclosure JWT)

### Used By

EUDI, Procivis

### Overview

SD-JWT extends the standard JWT format with a hash-based selective disclosure mechanism. The issuer creates a JWT containing hashed claim references; the actual claim values are provided as separate disclosures. The holder can choose which disclosures to present, revealing only the needed claims.

### Structure

An SD-JWT credential has the following wire format:

```
<issuer-jwt>~<disclosure1>~<disclosure2>~...~<kb-jwt>
```

Where:
- `<issuer-jwt>` is a standard JWT signed by the issuer
- Each `<disclosure>` is a base64url-encoded JSON array: `[salt, claim_name, claim_value]`
- `<kb-jwt>` is an optional key binding JWT signed by the holder
- `~` is the delimiter character

### Issuer JWT

The issuer JWT payload contains `_sd` arrays with hashes of the disclosures:

```json
{
  "iss": "https://issuer.example.com",
  "iat": 1700000000,
  "exp": 1700086400,
  "vct": "IdentityCredential",
  "_sd_alg": "sha-256",
  "_sd": [
    "hashed_disclosure_for_family_name",
    "hashed_disclosure_for_given_name",
    "hashed_disclosure_for_birth_date"
  ],
  "cnf": {
    "jwk": {
      "kty": "EC",
      "crv": "P-256",
      "x": "...",
      "y": "..."
    }
  }
}
```

### Disclosure Mechanism

Each disclosure is constructed as:

```
disclosure = base64url([salt, claim_name, claim_value])
hash = base64url(SHA-256(disclosure))
```

The hash is included in the `_sd` array of the issuer JWT. During presentation, the holder includes only the disclosures for claims they wish to reveal. The verifier hashes each received disclosure and checks that the hash appears in the `_sd` array of the issuer JWT.

```
Issuance (all claims):
  Issuer JWT: { "_sd": [hash1, hash2, hash3] }
  Disclosures: [d1: family_name, d2: given_name, d3: birth_date]

Presentation (selective):
  Issuer JWT: { "_sd": [hash1, hash2, hash3] }  (unchanged)
  Disclosures: [d3: birth_date]                  (only birth_date revealed)
  KB-JWT: signed by holder's key                 (proves possession)

Verifier checks:
  SHA-256(d3) == hash3  (yes, in _sd array)
  Issuer JWT signature valid
  KB-JWT signature valid against cnf.jwk
```

### Key Binding JWT (KB-JWT)

The KB-JWT is signed by the holder's private key (the key referenced in the issuer JWT's `cnf` claim). It contains:

```json
{
  "typ": "kb+jwt",
  "alg": "ES256"
}
{
  "iat": 1700001000,
  "aud": "https://verifier.example.com",
  "nonce": "verifier-provided-nonce",
  "sd_hash": "hash_of_issuer_jwt_and_disclosures"
}
```

The KB-JWT serves three purposes:
1. **Proves holder possession** of the key bound to the credential
2. **Binds to the verifier** via the `aud` claim
3. **Binds to the session** via the `nonce` and `sd_hash`

### Comparison to mDoc Selective Disclosure

| Aspect | SD-JWT | mDoc |
|--------|--------|------|
| Encoding | JSON (text-based) | CBOR (binary) |
| Disclosure mechanism | Hash of [salt, name, value] | Hash of IssuerSignedItem |
| Integrity object | JWT with _sd array | MSO with digest map |
| Key binding | KB-JWT (separate JWT) | DeviceAuth (COSE_Sign1) |
| Transport efficiency | Less compact (base64 text) | More compact (binary) |
| Web ecosystem fit | Excellent (JWT is ubiquitous) | Requires CBOR libraries |

---

## W3C Verifiable Credential with JSON-LD

### Used By

Affinidi, Procivis

### Overview

The W3C Verifiable Credentials Data Model (v1.1 / v2.0) defines a JSON-LD-based format for expressing credentials. JSON-LD (JSON for Linked Data) adds semantic meaning to JSON through `@context` references, enabling interoperability across different credential schemas.

### Structure

```json
{
  "@context": [
    "https://www.w3.org/2018/credentials/v1",
    "https://www.w3.org/2018/credentials/examples/v1"
  ],
  "id": "https://issuer.example.com/credentials/123",
  "type": ["VerifiableCredential", "IdentityCredential"],
  "issuer": "did:web:issuer.example.com",
  "issuanceDate": "2024-01-15T00:00:00Z",
  "credentialSubject": {
    "id": "did:key:z6Mk...",
    "familyName": "Smith",
    "givenName": "Alice",
    "birthDate": "1990-01-15"
  },
  "proof": {
    "type": "EcdsaSecp256k1Signature2019",
    "created": "2024-01-15T00:00:00Z",
    "verificationMethod": "did:web:issuer.example.com#key-1",
    "proofPurpose": "assertionMethod",
    "proofValue": "z58DAdFfa9SkqZMVPxAQp..."
  }
}
```

### Key Components

| Component | Purpose |
|-----------|---------|
| `@context` | Defines the semantic meaning of terms. Enables machines to understand the credential schema without out-of-band agreement. |
| `type` | Classifies the credential. Must include `VerifiableCredential`. |
| `issuer` | Identifies the issuer, typically as a DID. |
| `credentialSubject` | Contains the claims about the subject. The `id` field (if present) identifies the holder. |
| `proof` | Contains the cryptographic proof of authenticity. |

### Data Integrity Proofs

JSON-LD VCs use Data Integrity proofs (formerly Linked Data Proofs). The proof process:

1. **Canonicalize** the JSON-LD document using URDNA2015 (Universal RDF Dataset Normalization Algorithm 2015). This produces a deterministic serialization regardless of JSON key ordering or whitespace.
2. **Hash** the canonicalized output using the specified hash algorithm.
3. **Sign** the hash using the specified signature algorithm.
4. **Embed** the proof in the credential document.

The canonicalization step is critical: without it, the same logical credential could have different JSON serializations, breaking signature verification.

### Selective Disclosure

Standard JSON-LD VCs do not natively support selective disclosure. The entire `credentialSubject` is either presented or not. Selective disclosure requires either:
- Issuing multiple single-claim credentials
- Using a ZKP-capable proof format (e.g., BBS+ with JSON-LD)
- Layering SD mechanisms on top (less common for JSON-LD)

---

## JWT VC

### Used By

Procivis

### Overview

A JWT VC encodes a W3C Verifiable Credential as a standard JWT. This is the simplest credential format: the VC claims are the JWT payload, and the JWT signature provides integrity and authenticity.

### Structure

```
Header (JOSE):
{
  "alg": "ES256",
  "typ": "JWT",
  "kid": "did:web:issuer.example.com#key-1"
}

Payload:
{
  "iss": "did:web:issuer.example.com",
  "sub": "did:key:z6Mk...",
  "iat": 1700000000,
  "exp": 1700086400,
  "vc": {
    "@context": ["https://www.w3.org/2018/credentials/v1"],
    "type": ["VerifiableCredential", "IdentityCredential"],
    "credentialSubject": {
      "familyName": "Smith",
      "givenName": "Alice",
      "birthDate": "1990-01-15"
    }
  }
}

Signature:
  ES256 signature over base64url(header) + "." + base64url(payload)
```

### Characteristics

| Aspect | Detail |
|--------|--------|
| Encoding | JSON (base64url in JWT compact serialization) |
| Signing | Standard JWS (RFC 7515) |
| Selective disclosure | None (entire payload is visible) |
| Key binding | Via `sub` claim (DID of holder) |
| Ecosystem fit | Maximum compatibility with existing JWT infrastructure |
| Semantic richness | Lower than full JSON-LD (optional @context) |

### When to Use JWT VC

JWT VC is appropriate when:
- Selective disclosure is not required
- The verifier needs to see all claims
- Simplicity and broad library support are priorities
- The credential will be transmitted over standard web protocols

---

## Format Comparison

| Feature | mDoc (ISO 18013-5) | SD-JWT | W3C VC (JSON-LD) | JWT VC |
|---------|--------------------| -------|-------------------|--------|
| **Encoding** | CBOR (binary) | JSON (text) | JSON-LD (text) | JSON/JWT (text) |
| **Signing** | COSE_Sign1 | JWS | Data Integrity Proof | JWS |
| **Selective disclosure** | Yes (MSO hash-based) | Yes (disclosure hash-based) | No (native) / Yes (with BBS+) | No |
| **Key binding** | DeviceAuth | KB-JWT (cnf claim) | credentialSubject.id | sub claim |
| **Offline presentation** | Designed for it (BLE/NFC) | Possible but not primary | Not designed for it | Not designed for it |
| **Compactness** | Most compact (binary) | Moderate | Least compact (JSON-LD contexts) | Moderate |
| **Semantic richness** | Namespace-based | Minimal | Richest (linked data) | Minimal |
| **Library ecosystem** | Specialized (CBOR/COSE) | Broad (JWT libraries) | Moderate (JSON-LD processing) | Broadest (JWT ubiquitous) |
| **Standards body** | ISO | IETF | W3C | IETF + W3C |
| **Primary use case** | Government ID, mDL | General-purpose with privacy | Decentralized identity | Simple credentials |
| **Used by** | EUDI, Procivis | EUDI, Procivis | Affinidi, Procivis | Procivis |

### Format Selection Guidance

```
Does the credential need selective disclosure?
  |
  +-- Yes --> Is offline presentation required?
  |             |
  |             +-- Yes --> mDoc (ISO 18013-5)
  |             +-- No  --> SD-JWT
  |
  +-- No  --> Is semantic interoperability critical?
                |
                +-- Yes --> W3C VC (JSON-LD)
                +-- No  --> JWT VC
```

### Multi-Format Support

Procivis is notable for supporting all four formats, allowing credential issuers to choose the format that best fits their use case. EUDI supports mDoc and SD-JWT, covering the two formats required by the EU Digital Identity framework. Affinidi focuses on JSON-LD VCs, aligning with the W3C decentralized identity ecosystem.

| Implementation | mDoc | SD-JWT | JSON-LD VC | JWT VC |
|---------------|------|--------|------------|--------|
| EUDI | Yes | Yes | No | No |
| Procivis | Yes | Yes | Yes | Yes |
| Affinidi | No | No | Yes | No |
