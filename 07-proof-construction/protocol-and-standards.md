# Proof Construction -- Protocol & Standards

This document specifies the technical structure of proofs used in OID4VCI credential issuance, covering the JWT proof type, DPoP proof, and Key Binding JWT in SD-JWT credentials.

---

## JWT Proof Type in OID4VCI

The JWT proof type (`proof_type: "jwt"`) is the primary proof mechanism defined by the OID4VCI specification. The wallet constructs a signed JWT and includes it in the credential request to demonstrate possession of the private key that will be bound to the credential.

### Proof Object Structure

The proof is included in the credential request body:

```json
{
  "proof": {
    "proof_type": "jwt",
    "jwt": "<signed-jwt>"
  }
}
```

The `proof_type` field identifies the proof mechanism. The `jwt` field contains the compact serialization of the signed JWT.

### JWT Header

The JWT header contains the following parameters:

| Parameter | Required | Value | Description |
|-----------|----------|-------|-------------|
| `typ` | Yes | `openid4vci-proof+jwt` | Media type indicating this is an OID4VCI proof JWT. This distinguishes the proof from other JWTs in the protocol (e.g., access tokens, DPoP proofs). |
| `alg` | Yes | `ES256`, `EdDSA`, etc. | The cryptographic algorithm used to sign the JWT. Must be supported by the issuer (declared in issuer metadata). |
| `jwk` | Conditional | `{ "kty": "EC", "crv": "P-256", "x": "...", "y": "..." }` | The holder's public key in JWK format. Included when the key is not registered with the issuer. Mutually exclusive with `kid`. |
| `kid` | Conditional | `"did:key:z6Mkf..."` | A key identifier referencing the holder's public key. Used when the key is pre-registered with the issuer or identified by a DID. Mutually exclusive with `jwk`. |

**Header example with `jwk`:**

```json
{
  "typ": "openid4vci-proof+jwt",
  "alg": "ES256",
  "jwk": {
    "kty": "EC",
    "crv": "P-256",
    "x": "nGrAfI-IjYMnJh9TnDjRXBaLuPPIHdv4OkF5fOjwrtk",
    "y": "MWbhEYMiu0lDBPjhMQ2G8xvFOaG1fiAFgPSCMnGHuqM"
  }
}
```

**Header example with `kid`:**

```json
{
  "typ": "openid4vci-proof+jwt",
  "alg": "EdDSA",
  "kid": "did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK"
}
```

### JWT Payload

The JWT payload contains the following claims:

| Claim | Required | Description |
|-------|----------|-------------|
| `iss` | Conditional | The `client_id` of the wallet. Required when the wallet has a registered client identifier. In pre-authorized flows without client registration, this may be omitted. |
| `aud` | Yes | The `credential_issuer` identifier (the issuer's URL). This binds the proof to the specific issuer, preventing the proof from being replayed against a different issuer. |
| `iat` | Yes | Issued-at timestamp (seconds since Unix epoch). The issuer uses this to enforce time-based validity windows. |
| `nonce` | Yes | The `c_nonce` value received from the issuer in the token response or a previous credential response. This binds the proof to the current issuance session and prevents replay. |

**Payload example:**

```json
{
  "iss": "https://wallet.example.com",
  "aud": "https://issuer.example.com",
  "iat": 1701234567,
  "nonce": "tZignsnFbp"
}
```

### Signing

The complete JWT is signed with the holder's private key using the algorithm specified in the `alg` header parameter:

```
BASE64URL(header) + "." + BASE64URL(payload) + "." + BASE64URL(signature)
```

The issuer verifies the signature using the public key provided in the `jwk` header parameter (or resolved via the `kid` reference). A valid signature proves that the holder controls the private key corresponding to the public key that will be bound to the credential.

### Verification Steps (Issuer Side)

The issuer performs the following verification when it receives a credential request with a JWT proof:

1. **Decode the JWT** and extract the header and payload.
2. **Verify `typ`** is `openid4vci-proof+jwt`.
3. **Verify `alg`** is a supported algorithm.
4. **Extract the public key** from `jwk` (or resolve via `kid`).
5. **Verify the signature** against the extracted public key.
6. **Verify `aud`** matches the issuer's own `credential_issuer` identifier.
7. **Verify `nonce`** matches the `c_nonce` issued to this session.
8. **Verify `iat`** is within an acceptable time window (implementation-specific tolerance).
9. **Bind the credential** to the verified public key.

If any step fails, the issuer returns an error response (typically `invalid_proof`).

---

## Batch Proof Construction

When requesting multiple credentials in a single request, the wallet uses the `proofs` (plural) field instead of `proof`:

```json
{
  "proofs": {
    "jwt": [
      "<signed-jwt-1>",
      "<signed-jwt-2>",
      "<signed-jwt-3>"
    ]
  }
}
```

Each JWT in the array may use a different key pair, allowing the issuer to bind each credential instance to a distinct key. This is particularly relevant for one-time-use credential policies, where each credential instance is bound to a unique key to prevent linkability across presentations.

All JWTs in the batch share the same `nonce` (the `c_nonce` from the current session) but may differ in their `jwk` values.

---

## DPoP Proof

Demonstrating Proof of Possession (DPoP, defined in RFC 9449) is a complementary proof mechanism that binds the access token to the client's key pair. Unlike the credential request proof (which binds the credential to the holder's key), the DPoP proof binds the HTTP request to the token that authorizes it.

### DPoP JWT Structure

The DPoP proof is a JWT sent in the `DPoP` HTTP header:

```
DPoP: eyJhbGciOiJFUzI1NiIsInR5cCI6ImRwb3Arand0IiwiandrIjp7...
```

**Header:**

| Parameter | Value | Description |
|-----------|-------|-------------|
| `typ` | `dpop+jwt` | Media type for DPoP proof. |
| `alg` | `ES256`, `EdDSA`, etc. | Signing algorithm. |
| `jwk` | `{ ... }` | The DPoP public key. |

**Payload:**

| Claim | Description |
|-------|-------------|
| `jti` | Unique identifier for this DPoP proof (prevents replay). |
| `htm` | HTTP method of the request (e.g., `POST`). |
| `htu` | HTTP URI of the request (e.g., `https://issuer.example.com/credential`). |
| `iat` | Issued-at timestamp. |
| `ath` | Access token hash (base64url-encoded SHA-256 hash of the access token). Present when the DPoP proof accompanies a resource request. |

**Example DPoP payload:**

```json
{
  "jti": "e1j3V_bKic8-LAEB_lccDP",
  "htm": "POST",
  "htu": "https://issuer.example.com/credential",
  "iat": 1701234567,
  "ath": "fUHyO2r2Z3DZ53EsNrWBb0xWXoaNy59IiKCAqksmQEo"
}
```

### Relationship Between DPoP and Credential Proof

The DPoP proof and the credential request proof serve different purposes:

| Aspect | DPoP Proof | Credential Request Proof |
|--------|-----------|------------------------|
| **What it proves** | Possession of the key bound to the access token | Possession of the key to bind to the credential |
| **Where it appears** | `DPoP` HTTP header | `proof` field in request body |
| **Key pair** | DPoP key pair (transport) | Holder key pair (credential binding) |
| **Specification** | RFC 9449 | OID4VCI |
| **When verified** | Token endpoint and resource server | Credential endpoint only |

These may use the same key pair or different key pairs, depending on the implementation and security requirements. Using separate keys provides stronger isolation between transport-level and credential-level security.

---

## Key Binding JWT in SD-JWT

When an SD-JWT credential is presented, the holder includes a Key Binding JWT (KB-JWT) to prove possession of the key that was bound to the credential during issuance.

### SD-JWT Structure

An SD-JWT with selective disclosure and key binding has the following format:

```
<issuer-signed-jwt>~<disclosure1>~<disclosure2>~...~<kb-jwt>
```

The tilde (`~`) character separates the components. The disclosures are base64url-encoded JSON arrays of `[salt, claim_name, claim_value]`. The KB-JWT is the final component.

### KB-JWT Structure

**Header:**

```json
{
  "typ": "kb+jwt",
  "alg": "ES256"
}
```

**Payload:**

| Claim | Description |
|-------|-------------|
| `iat` | Issued-at timestamp. |
| `aud` | The verifier's identifier (binds the presentation to the intended recipient). |
| `nonce` | The verifier's nonce (prevents replay). |
| `sd_hash` | Hash of the SD-JWT and selected disclosures (binds the KB-JWT to the specific presentation). |

**Example payload:**

```json
{
  "iat": 1701234567,
  "aud": "https://verifier.example.com",
  "nonce": "n-0S6_WzA2Mj",
  "sd_hash": "X9yH0vQJxKMPdlJc2QUL_RYJczGdsjCF9J5Z6OEpMR0"
}
```

### Relationship to Issuance Proof

The KB-JWT used at presentation time is the downstream consequence of the proof constructed during issuance. During issuance, the holder proves possession of a key and the issuer binds the credential to that key (in the `cnf` claim). During presentation, the holder proves possession of the same key via the KB-JWT. The two proofs use the same key pair but serve different protocol contexts:

| Aspect | Issuance Proof | Presentation KB-JWT |
|--------|---------------|-------------------|
| **Context** | OID4VCI credential request | OID4VP credential presentation |
| **Audience** | Issuer | Verifier |
| **Nonce source** | `c_nonce` from issuer | Nonce from verifier |
| **Purpose** | Bind credential to key | Prove possession of bound key |

---

## Algorithm Requirements

The OID4VCI specification does not mandate specific algorithms, but issuer metadata declares which algorithms are supported. Common algorithms in the ecosystem:

| Algorithm | Curve / Parameters | Usage |
|-----------|-------------------|-------|
| `ES256` | P-256 (secp256r1) | Dominant algorithm for EUDI wallet ecosystem. Hardware-backed on both Android (StrongBox/TEE) and iOS (Secure Enclave). |
| `EdDSA` | Ed25519 | Used by some issuers. Supported by Procivis ONE. |
| `ES384` | P-384 | Higher security level. Less common in mobile wallets due to limited hardware support. |
| `ES512` | P-521 | Highest ECDSA security level. Rare in current deployments. |
| `CRYSTALS-DILITHIUM` | Lattice-based | Post-quantum algorithm. Supported by Procivis ONE as a forward-looking option. |

The HAIP (High Assurance Interoperability Profile) restricts the algorithm set to ensure interoperability, typically requiring `ES256` for broad compatibility.

---

## Summary

The JWT proof type is the primary proof mechanism in OID4VCI. It consists of a signed JWT with a specific header (`typ`, `alg`, `jwk`/`kid`) and payload (`iss`, `aud`, `iat`, `nonce`) that demonstrates the holder controls the private key to which the credential will be bound. DPoP proofs complement this by binding access tokens to transport keys, and Key Binding JWTs extend the chain of trust into the presentation phase. Together, these mechanisms ensure that credentials are bound to specific holders from issuance through presentation.
