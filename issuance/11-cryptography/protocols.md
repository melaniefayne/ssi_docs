# Protocol-Level Cryptography

This page documents the cryptographic mechanisms used at the protocol layer -- how credentials are securely exchanged between issuers, wallets, and verifiers. These protocols protect the communication channels and bind tokens to their intended recipients.

---

## OID4VCI Token Exchange

### Overview

OpenID for Verifiable Credential Issuance (OID4VCI) uses an OAuth 2.0-based token exchange to authorize credential issuance. The wallet obtains an access token from the issuer's token endpoint, then uses that token to request credential issuance from the credential endpoint.

### Flow

```
Wallet                          Issuer
  |                               |
  |  1. Token Request             |
  |  (grant_type, pre-auth code,  |
  |   tx_code, DPoP proof)        |
  +------------------------------>|
  |                               |
  |  2. Token Response            |
  |  (access_token, c_nonce,      |
  |   token_type: DPoP)           |
  |<------------------------------+
  |                               |
  |  3. Credential Request        |
  |  (format, proof with c_nonce, |
  |   DPoP proof, access token)   |
  +------------------------------>|
  |                               |
  |  4. Credential Response       |
  |  (credential, c_nonce_new)    |
  |<------------------------------+
```

### Cryptographic Bindings

| Step | What Is Protected | Mechanism |
|------|------------------|-----------|
| Token request | Token bound to wallet's key pair | DPoP proof JWT |
| Token response | Nonce for replay prevention | c_nonce |
| Credential request | Proof of key possession | JWT proof with c_nonce |
| Credential request | Token cannot be replayed | DPoP proof with ath (access token hash) |

### Access Token Binding

The access token returned by the issuer is a **sender-constrained token** when DPoP is used. It is bound to the wallet's DPoP key pair via the token's `cnf` (confirmation) claim:

```json
{
  "active": true,
  "cnf": {
    "jkt": "SHA-256 thumbprint of wallet's DPoP public key"
  }
}
```

When the wallet presents this token to the credential endpoint, it must also present a DPoP proof signed by the same key. The issuer verifies that the DPoP proof's key thumbprint matches the token's `jkt`, confirming the presenter is the legitimate token holder.

---

## PAR (Pushed Authorization Requests) -- RFC 9126

### What Problem It Solves

In a standard OAuth 2.0 authorization code flow, the authorization request is sent as URL query parameters in the browser redirect. This creates several problems:
- **Request tampering**: Parameters can be modified by the browser, extensions, or intermediaries
- **URL length limits**: Complex requests may exceed URL size limits
- **Log exposure**: Authorization parameters appear in server logs and browser history
- **No integrity protection**: The authorization server cannot verify the request was not modified in transit

PAR solves these by allowing the client to **pre-register** the authorization request directly with the authorization server over a back-channel.

### How It Works

```
Wallet                          Issuer (Authorization Server)
  |                               |
  |  1. POST /par                 |
  |  (client_id, redirect_uri,    |
  |   scope, code_challenge,      |
  |   state, etc.)                |
  +------------------------------>|
  |                               |  Server stores the request
  |  2. PAR Response              |  and returns a URI reference
  |  (request_uri, expires_in)    |
  |<------------------------------+
  |                               |
  |  3. Authorization Request     |
  |  (client_id, request_uri)     |
  |  [via browser redirect]       |
  +------------------------------>|
  |                               |  Server looks up the stored
  |                               |  request using request_uri
```

### Cryptographic Properties

| Property | How PAR Achieves It |
|----------|-------------------|
| Request integrity | The request is sent directly to the server over TLS, not through the browser |
| Request confidentiality | Request parameters are not exposed in URLs |
| Request authentication | The client authenticates when pushing the request (client_id + optional client authentication) |
| Replay prevention | The request_uri is single-use and time-limited (expires_in) |

### Relationship to PKCE

PAR and PKCE are complementary. PAR protects the authorization request from tampering. PKCE protects the authorization code from interception. In a properly secured OID4VCI flow, both are used together.

---

## DPoP (Demonstration of Proof-of-Possession) -- RFC 9449

### What Problem It Solves

Standard OAuth 2.0 bearer tokens are vulnerable to token theft: if an attacker intercepts the access token, they can use it from any device. There is no way for the resource server to verify that the presenter of the token is the same entity that originally requested it.

DPoP solves this by **binding the access token to a specific key pair**. The token can only be used by the entity that controls the corresponding private key.

### DPoP JWT Structure

A DPoP proof is a JWT with the following structure:

**Header:**
```json
{
  "typ": "dpop+jwt",
  "alg": "ES256",
  "jwk": {
    "kty": "EC",
    "crv": "P-256",
    "x": "base64url-encoded x coordinate",
    "y": "base64url-encoded y coordinate"
  }
}
```

**Payload:**
```json
{
  "jti": "unique-identifier-for-this-proof",
  "htm": "POST",
  "htu": "https://issuer.example.com/credential",
  "iat": 1700001000,
  "ath": "base64url(SHA-256(access_token))"
}
```

### Field Definitions

| Field | Name | Purpose |
|-------|------|---------|
| `jti` | JWT ID | Unique identifier for this proof. Prevents replay. The server tracks seen jti values. |
| `htm` | HTTP Method | The HTTP method of the request (GET, POST). Binds the proof to a specific operation. |
| `htu` | HTTP URI | The URL of the endpoint being accessed. Binds the proof to a specific endpoint. |
| `iat` | Issued At | Timestamp of proof creation. The server rejects proofs that are too old (clock skew tolerance). |
| `ath` | Access Token Hash | SHA-256 hash of the access token (base64url-encoded). Binds the DPoP proof to a specific access token. |

### How DPoP Prevents Token Theft

```
Scenario: Attacker intercepts access token

Without DPoP:
  Attacker sends: Authorization: Bearer <stolen_token>
  Server accepts: Token is valid  --> ATTACK SUCCEEDS

With DPoP:
  Attacker sends: Authorization: DPoP <stolen_token>
                  DPoP: <proof signed with attacker's key>
  Server checks: DPoP key thumbprint != token's cnf.jkt
  Server rejects: Key mismatch  --> ATTACK FAILS

  Attacker cannot sign a valid DPoP proof without the wallet's private key.
  The private key is in hardware (TEE/Secure Enclave) and cannot be extracted.
```

### DPoP in OID4VCI

DPoP is used at two points in the OID4VCI flow:

1. **Token endpoint**: The wallet sends a DPoP proof with the token request. The issuer binds the access token to the wallet's DPoP key.
2. **Credential endpoint**: The wallet sends a DPoP proof with the credential request, including the `ath` field to bind the proof to the specific access token.

---

## PKCE (Proof Key for Code Exchange) -- RFC 7636

### What Problem It Solves

In the OAuth 2.0 authorization code flow, the authorization code is returned to the client via a browser redirect. On mobile devices, this redirect can be intercepted by a malicious app that has registered the same custom URL scheme. The attacker could then exchange the intercepted code for an access token.

PKCE prevents this by requiring the client to prove it is the same entity that initiated the authorization request.

### How It Works

**Setup (before authorization request):**
1. The wallet generates a random **code verifier**: a high-entropy string (43-128 characters)
2. The wallet computes the **code challenge**: `BASE64URL(SHA-256(code_verifier))` (S256 method)
3. The wallet includes `code_challenge` and `code_challenge_method=S256` in the authorization request

**Exchange (when redeeming the authorization code):**
1. The wallet sends the original `code_verifier` with the token request
2. The issuer computes `BASE64URL(SHA-256(code_verifier))` and checks it matches the stored `code_challenge`
3. If they match, the issuer issues the token. If not, the request is rejected.

```
Wallet                          Issuer
  |                               |
  |  verifier = random_string()   |
  |  challenge = SHA256(verifier)  |
  |                               |
  |  Authorization Request        |
  |  (code_challenge=challenge)   |
  +------------------------------>|  Stores challenge
  |                               |
  |  Authorization Code           |
  |<------------------------------+
  |                               |
  |  Token Request                |
  |  (code=auth_code,             |
  |   code_verifier=verifier)     |
  +------------------------------>|  Checks:
  |                               |  SHA256(verifier) == challenge?
  |  Access Token                 |
  |<------------------------------+
```

### Why This Works

An attacker who intercepts the authorization code does not know the code verifier (it was generated locally on the wallet and never sent through the browser). Without the verifier, the attacker cannot produce the correct value at the token endpoint. The SHA-256 hash is one-way, so the challenge (which the attacker might see in the authorization request) cannot be reversed to obtain the verifier.

---

## JWK (JSON Web Key) -- RFC 7517

### Overview

JWK is a JSON format for representing cryptographic keys. It is used throughout OID4VCI for key exchange, proofs, and attestation.

### EC Key Example (P-256)

```json
{
  "kty": "EC",
  "crv": "P-256",
  "x": "f83OJ3D2xF1Bg8vub9tLe1gHMzV76e8Tus9uPHvRVEU",
  "y": "x_FEzRu9m36HLN_tue659LNpXW6pCyStikYjKIWI5a0",
  "kid": "key-1",
  "use": "sig"
}
```

### Field Definitions

| Field | Name | Purpose |
|-------|------|---------|
| `kty` | Key Type | Algorithm family (EC, RSA, OKP) |
| `crv` | Curve | The elliptic curve (P-256, Ed25519, secp256k1) |
| `x`, `y` | Coordinates | The public key coordinates (base64url-encoded) |
| `d` | Private Key | The private key value (only present in private JWKs -- never transmitted) |
| `kid` | Key ID | Identifier for the key, used to match keys across documents |
| `use` | Key Use | Intended use: `sig` (signing) or `enc` (encryption) |

### Where JWK Is Used in SSI

| Context | Usage |
|---------|-------|
| DPoP JWT header | The `jwk` field contains the wallet's public key |
| JWT proof in OID4VCI | The proof JWT header references or embeds the key |
| Issuer metadata (JWKS) | The issuer publishes signing keys as a JWK Set |
| SD-JWT `cnf` claim | The `jwk` in `cnf` binds the credential to the holder's key |
| DID documents | Keys are often expressed as JWK in DID document verification methods |

### JWK Thumbprint (RFC 7638)

A JWK thumbprint is a SHA-256 hash of the canonical form of a JWK. It provides a compact, deterministic identifier for a key. DPoP uses the JWK thumbprint (`jkt`) to bind access tokens to keys without embedding the full key in the token.

```
Thumbprint = BASE64URL(SHA-256(canonical_jwk))

Where canonical_jwk contains only required fields, in lexicographic order:
  {"crv":"P-256","kty":"EC","x":"...","y":"..."}
```

---

## JWS (JSON Web Signature) -- RFC 7515

### Overview

JWS provides integrity protection for arbitrary payloads using digital signatures or MACs. It is the foundation for JWT (which is a JWS with a JSON payload) and is used throughout OID4VCI.

### Structure

JWS Compact Serialization:

```
BASE64URL(header) . BASE64URL(payload) . BASE64URL(signature)
```

**Header (JOSE Header):**
```json
{
  "alg": "ES256",
  "typ": "JWT",
  "kid": "key-1"
}
```

**Payload:** The data being signed (for JWT, this is a JSON object with claims).

**Signature:** The digital signature computed over `BASE64URL(header) . BASE64URL(payload)` using the algorithm specified in `alg`.

### Verification Process

1. Parse the three base64url-encoded segments
2. Decode the header, identify the algorithm and key
3. Resolve the signing key (from `kid`, `jwk`, or out-of-band)
4. Compute the expected signature over `header_b64 . payload_b64`
5. Compare with the provided signature

### JWS in SSI Protocols

| Protocol Context | What Is Protected | Algorithm |
|-----------------|------------------|-----------|
| JWT proof of possession | Holder's key binding to credential request | ES256 |
| DPoP proof | Sender-constrained access token | ES256 |
| SD-JWT issuer token | Credential claims and disclosure hashes | ES256, EdDSA |
| KB-JWT | Holder's proof of possession at presentation | ES256 |
| Wallet attestation JWT | Wallet integrity assertion | ES256 |

---

## COSE (CBOR Object Signing and Encryption) -- RFC 9052

### Overview

COSE is the CBOR equivalent of JOSE. Where JOSE (JWS/JWE/JWK) uses JSON encoding, COSE uses CBOR, producing more compact binary representations. COSE is used exclusively in the mDoc credential format.

### COSE_Sign1 Structure

COSE_Sign1 is a single-signer signature structure. It is the COSE equivalent of JWS Compact Serialization.

```
COSE_Sign1 = [
  protected_header,    // CBOR-encoded, signed
  unprotected_header,  // CBOR map, NOT signed
  payload,             // The data being signed (or null for detached)
  signature            // The digital signature
]
```

**Protected Header (typical for mDoc):**
```
{
  1: -7,        // Algorithm: ES256
  33: <cert>    // x5chain: issuer certificate chain
}
```

COSE algorithm identifiers use integers rather than strings:

| COSE ID | Algorithm | JOSE Equivalent |
|---------|-----------|-----------------|
| -7 | ES256 (ECDSA w/ SHA-256, P-256) | "ES256" |
| -35 | ES384 (ECDSA w/ SHA-384, P-384) | "ES384" |
| -36 | ES512 (ECDSA w/ SHA-512, P-521) | "ES512" |
| -8 | EdDSA | "EdDSA" |

### COSE in mDoc

| Structure | Used For | Signer |
|-----------|----------|--------|
| COSE_Sign1 (IssuerAuth) | Signs the MSO (Mobile Security Object) | Issuer |
| COSE_Sign1 (DeviceAuth) | Signs the device authentication data | Holder (device key) |
| COSE_Mac0 (DeviceAuth, alternative) | MACs the device authentication data | Holder (session key from key agreement) |

### COSE vs. JOSE

| Aspect | COSE | JOSE |
|--------|------|------|
| Encoding | CBOR (binary) | JSON (text) |
| Size | More compact | Larger (base64 overhead) |
| Parsing speed | Faster | Slower |
| Ecosystem | IoT, mDoc, FIDO | Web APIs, OAuth, JWT |
| Human readability | Requires CBOR decoder | Directly readable (base64) |
| Standard | RFC 9052 | RFC 7515 (JWS), RFC 7516 (JWE) |

---

## Protocol Cryptography Summary

The following table maps each protocol mechanism to the threat it mitigates:

| Mechanism | RFC / Standard | Threat Mitigated | How |
|-----------|---------------|-----------------|-----|
| PAR | RFC 9126 | Request tampering | Pre-registers request via back-channel |
| PKCE | RFC 7636 | Authorization code interception | Code verifier/challenge binding |
| DPoP | RFC 9449 | Access token theft | Sender-constrained tokens via proof-of-possession |
| c_nonce | OID4VCI | Replay attacks | Fresh nonce in every credential request proof |
| JWS | RFC 7515 | Data tampering | Digital signature over payload |
| COSE | RFC 9052 | Data tampering (binary) | Digital signature over CBOR payload |
| JWK / JWK Thumbprint | RFC 7517 / 7638 | Key ambiguity | Canonical key representation and identification |
| KB-JWT | SD-JWT spec | Credential transfer | Holder proves possession of bound key |
