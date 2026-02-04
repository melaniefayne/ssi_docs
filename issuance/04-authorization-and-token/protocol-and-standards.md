# Authorization & Token Exchange -- Protocol & Standards

This document details the protocol mechanisms used during the authorization and token exchange phase of OID4VCI. It covers the OAuth 2.0 Authorization Code Flow with PKCE, Pushed Authorization Requests (PAR), Demonstration of Proof-of-Possession (DPoP), the Pre-Authorized Code Grant, and the token endpoint response structure.

---

## 1. OAuth 2.0 Authorization Code Flow with PKCE

### Base Protocol

The Authorization Code Flow is defined in [RFC 6749 Section 4.1](https://datatracker.ietf.org/doc/html/rfc6749#section-4.1). In the OID4VCI context, the wallet acts as the OAuth 2.0 client, the issuer (or its associated authorization server) acts as the authorization server, and the user is the resource owner.

### PKCE Extension (RFC 7636)

Proof Key for Code Exchange ([RFC 7636](https://datatracker.ietf.org/doc/html/rfc7636)) mitigates authorization code interception attacks. This is critical for mobile wallet applications, where custom URI scheme redirects can be intercepted by malicious apps registered for the same scheme.

**PKCE flow:**

```
Wallet                             Authorization Server
  |                                        |
  |  1. Generate code_verifier             |
  |     (cryptographic random string)      |
  |                                        |
  |  2. Derive code_challenge              |
  |     code_challenge =                   |
  |       BASE64URL(SHA256(code_verifier)) |
  |                                        |
  |  3. Authorization Request              |
  |     GET /authorize?                    |
  |       response_type=code&              |
  |       client_id=...&                   |
  |       redirect_uri=...&               |
  |       code_challenge=...&              |
  |       code_challenge_method=S256       |
  |  ----------------------------------->  |
  |                                        |
  |  4. User authenticates (browser)       |
  |                                        |
  |  5. Authorization Response             |
  |     redirect_uri?code=AUTH_CODE        |
  |  <-----------------------------------  |
  |                                        |
  |  6. Token Request                      |
  |     POST /token                        |
  |       grant_type=authorization_code&   |
  |       code=AUTH_CODE&                  |
  |       code_verifier=...               |
  |  ----------------------------------->  |
  |                                        |
  |  7. Server verifies:                   |
  |     BASE64URL(SHA256(code_verifier))   |
  |       == stored code_challenge         |
  |                                        |
  |  8. Token Response                     |
  |     { access_token, c_nonce, ... }     |
  |  <-----------------------------------  |
  |                                        |
```

**Key parameters:**

| Parameter | Location | Description |
|-----------|----------|-------------|
| `code_verifier` | Token request | Random string, 43-128 characters, from unreserved character set |
| `code_challenge` | Authorization request | `BASE64URL(SHA256(code_verifier))` |
| `code_challenge_method` | Authorization request | Must be `S256` (SHA-256). `plain` is permitted by the RFC but insecure and should not be used. |

**Why it matters:** Without PKCE, a malicious app on the same device that has registered a handler for the wallet's redirect URI can intercept the authorization code and exchange it for an access token. PKCE prevents this because the attacker does not possess the `code_verifier`.

---

## 2. Pushed Authorization Requests (PAR) -- RFC 9126

### Problem

In a standard authorization code flow, the authorization request parameters are sent as query parameters in the redirect URL. This has security implications:

- Authorization parameters are visible in browser history, server logs, and referrer headers.
- Complex requests with large parameters (e.g., request objects) may exceed URL length limits.
- The authorization server cannot authenticate the client before the user is redirected.

### Solution

[RFC 9126](https://datatracker.ietf.org/doc/html/rfc9126) introduces a two-step process:

```
Wallet                          PAR Endpoint              Authorization Endpoint
  |                                  |                           |
  |  1. POST /par                    |                           |
  |     { all auth request params }  |                           |
  |  ------------------------------> |                           |
  |                                  |                           |
  |  2. PAR Response                 |                           |
  |     { request_uri,               |                           |
  |       expires_in }               |                           |
  |  <------------------------------ |                           |
  |                                  |                           |
  |  3. Redirect user                |                           |
  |     GET /authorize?              |                           |
  |       client_id=...&             |                           |
  |       request_uri=...            |                           |
  |  -------------------------------------------------->        |
  |                                  |                           |
  |  4. Normal auth flow continues   |                           |
  |  <--------------------------------------------------        |
  |                                  |                           |
```

**Step 1 -- Push request:** The wallet sends all authorization parameters as a POST body to the PAR endpoint (typically `/par` or a path discovered from authorization server metadata). The authorization server validates the request and stores it server-side.

**Step 2 -- Receive request_uri:** The authorization server returns a `request_uri` -- an opaque reference to the stored request. This URI is time-limited (`expires_in`).

**Step 3 -- Redirect with request_uri:** Instead of including all parameters in the redirect URL, the wallet redirects the user to the authorization endpoint with only `client_id` and `request_uri`. The authorization server looks up the stored request using the `request_uri`.

**PAR response format:**

```json
{
  "request_uri": "urn:ietf:params:oauth:request_uri:bwc4JK-ESC0w8acc191e-Y1LTC2",
  "expires_in": 60
}
```

**Why it matters for OID4VCI:** The EUDI Architecture Reference Framework recommends PAR for all authorization code flows. PAR allows the authorization server to authenticate the wallet before the user interaction begins, and it keeps sensitive parameters out of the browser's address bar.

---

## 3. DPoP (Demonstration of Proof-of-Possession) -- RFC 9449

### Problem

Standard OAuth 2.0 Bearer tokens ([RFC 6750](https://datatracker.ietf.org/doc/html/rfc6750)) are usable by any party that possesses them. If an access token is leaked (through log files, man-in-the-middle attacks, or compromised TLS), the attacker can use it to make authorized requests.

### Solution

[RFC 9449](https://datatracker.ietf.org/doc/html/rfc9449) defines DPoP -- a mechanism that binds an access token to the client's key pair, creating a sender-constrained token. The token is usable only by the party that possesses the corresponding private key.

**DPoP flow:**

```
Wallet                             Token Endpoint / Credential Endpoint
  |                                        |
  |  1. Generate DPoP key pair             |
  |     (asymmetric, e.g., ES256)          |
  |                                        |
  |  2. Token Request                      |
  |     POST /token                        |
  |     DPoP: <dpop_proof_jwt>             |
  |     Body: { grant_type=..., ... }      |
  |  ----------------------------------->  |
  |                                        |
  |  3. Token Response                     |
  |     { access_token,                    |
  |       token_type: "DPoP",             |
  |       c_nonce, ... }                   |
  |  <-----------------------------------  |
  |                                        |
  |  4. Credential Request                 |
  |     POST /credential                   |
  |     Authorization: DPoP <token>        |
  |     DPoP: <new_dpop_proof_jwt>         |
  |     Body: { ... }                      |
  |  ----------------------------------->  |
  |                                        |
```

**DPoP proof JWT structure:**

The DPoP proof is a JWT signed by the wallet's DPoP key. It contains:

```json
{
  "typ": "dpop+jwt",
  "alg": "ES256",
  "jwk": {
    "kty": "EC",
    "crv": "P-256",
    "x": "...",
    "y": "..."
  }
}
```

Payload:

```json
{
  "jti": "<unique identifier>",
  "htm": "POST",
  "htu": "https://issuer.example.com/token",
  "iat": 1700000000,
  "ath": "<base64url SHA-256 hash of access_token>"
}
```

| Claim | Description |
|-------|-------------|
| `jti` | Unique identifier for this proof (prevents replay) |
| `htm` | HTTP method of the request |
| `htu` | HTTP URI of the request (scheme + authority + path, no query) |
| `iat` | Issued-at timestamp |
| `ath` | Access token hash (only in resource requests, not token requests) |

**How binding works:**

1. The wallet includes the DPoP proof (containing the public key in the `jwk` header) when requesting the access token.
2. The authorization server binds the access token to the public key from the DPoP proof by including a `jkt` (JWK Thumbprint) confirmation in the token.
3. When the wallet uses the access token at the credential endpoint, it must include a new DPoP proof signed by the same key.
4. The credential endpoint verifies that the DPoP proof's public key matches the key bound to the access token.

**Why it matters:** DPoP prevents token theft. Even if an attacker intercepts the access token, they cannot use it without the wallet's DPoP private key. This is especially important in mobile environments where tokens may traverse multiple system components.

---

## 4. Pre-Authorized Code Grant

The Pre-Authorized Code Grant is defined in the OID4VCI specification (not in base OAuth 2.0). It provides a simplified token exchange for cases where the issuer has already authenticated the user.

**Token request:**

```http
POST /token HTTP/1.1
Content-Type: application/x-www-form-urlencoded

grant_type=urn:ietf:params:oauth:grant-type:pre-authorized_code
&pre-authorized_code=SplxlOBeZQQYbYS6WxSbIA
&tx_code=493536
```

**Parameters:**

| Parameter | Required | Description |
|-----------|----------|-------------|
| `grant_type` | Yes | `urn:ietf:params:oauth:grant-type:pre-authorized_code` |
| `pre-authorized_code` | Yes | The code from the credential offer |
| `tx_code` | Conditional | Required if the credential offer specified a `tx_code` object in the grant |

### Transaction Code Requirement

If the credential offer's grant object includes a `tx_code` descriptor, the wallet must collect the transaction code from the user and include it in the token request. The `tx_code` descriptor in the offer specifies:

```json
{
  "grants": {
    "urn:ietf:params:oauth:grant-type:pre-authorized_code": {
      "pre-authorized_code": "SplxlOBeZQQYbYS6WxSbIA",
      "tx_code": {
        "input_mode": "numeric",
        "length": 6,
        "description": "Enter the 6-digit code sent to your email"
      }
    }
  }
}
```

The `tx_code` fields are detailed in [05 -- Nonce & Replay Protection](../05-nonce-and-replay-protection/protocol-and-standards.md).

### Pre-Authorized Code Security Properties

- The pre-authorized code is single-use. After a successful token exchange, the code is invalidated.
- The code is time-limited. The issuer sets an expiration based on the use case.
- If a `tx_code` is required, both the pre-authorized code and the transaction code must match for the exchange to succeed.
- The pre-authorized code is bound to the credential offer. It cannot be used to request credentials not included in the original offer.

---

## 5. Token Endpoint Response

Regardless of the grant type used, the token endpoint returns a response with the following structure:

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "Bearer",
  "expires_in": 300,
  "c_nonce": "tZignsnFbp",
  "c_nonce_expires_in": 86400
}
```

| Field | Type | Description |
|-------|------|-------------|
| `access_token` | string | The access token for the credential endpoint. May be a JWT or an opaque string. |
| `token_type` | string | `Bearer` for standard tokens, `DPoP` for DPoP-bound tokens. |
| `expires_in` | integer | Access token lifetime in seconds. |
| `c_nonce` | string | Credential nonce. Must be included in the proof JWT's `nonce` claim. |
| `c_nonce_expires_in` | integer | Nonce lifetime in seconds. The wallet must submit its credential request before this expires. |

### DPoP Token Response

When DPoP is used, the `token_type` is `DPoP` instead of `Bearer`:

```json
{
  "access_token": "Kz~8mXK1EalYznwH-LC-1fBAo...",
  "token_type": "DPoP",
  "expires_in": 300,
  "c_nonce": "tZignsnFbp",
  "c_nonce_expires_in": 86400
}
```

The wallet must then use the `DPoP` authorization scheme instead of `Bearer` when calling the credential endpoint, and must include a fresh DPoP proof in each request.

---

## 6. Protocol Flow Summary

The following diagram shows the complete authorization flow with all optional security mechanisms (PAR, PKCE, DPoP) in the Authorization Code Grant path:

```
Wallet                    PAR Endpoint    Auth Endpoint    Token Endpoint
  |                            |                |                |
  |  POST /par                 |                |                |
  |  (all auth params + PKCE   |                |                |
  |   code_challenge)          |                |                |
  |  ------------------------> |                |                |
  |                            |                |                |
  |  { request_uri }           |                |                |
  |  <------------------------ |                |                |
  |                            |                |                |
  |  Redirect user to /authorize?request_uri=...                 |
  |  -------------------------------------->    |                |
  |                            |                |                |
  |  User authenticates in browser              |                |
  |  <--------------------------------------    |                |
  |  redirect_uri?code=AUTH_CODE                |                |
  |                            |                |                |
  |  POST /token                                                 |
  |  (grant_type=authorization_code,                             |
  |   code=AUTH_CODE,                                            |
  |   code_verifier=...)                                         |
  |  DPoP: <dpop_proof>                                          |
  |  ------------------------------------------------------->    |
  |                            |                |                |
  |  { access_token (DPoP-bound), c_nonce }                      |
  |  <-------------------------------------------------------    |
  |                            |                |                |
```

And the Pre-Authorized Code path:

```
Wallet                                        Token Endpoint
  |                                                |
  |  POST /token                                   |
  |  (grant_type=pre-authorized_code,              |
  |   pre-authorized_code=...,                     |
  |   tx_code=...)                                 |
  |  DPoP: <dpop_proof>  (optional)                |
  |  --------------------------------------------> |
  |                                                |
  |  { access_token, c_nonce }                     |
  |  <-------------------------------------------- |
  |                                                |
```

---

## 7. Standards Reference

| Standard | Title | Relevance |
|----------|-------|-----------|
| [RFC 6749](https://datatracker.ietf.org/doc/html/rfc6749) | OAuth 2.0 Authorization Framework | Base authorization code flow |
| [RFC 7636](https://datatracker.ietf.org/doc/html/rfc7636) | PKCE (Proof Key for Code Exchange) | Authorization code interception mitigation |
| [RFC 9126](https://datatracker.ietf.org/doc/html/rfc9126) | Pushed Authorization Requests (PAR) | Pre-registered authorization requests |
| [RFC 9449](https://datatracker.ietf.org/doc/html/rfc9449) | DPoP (Demonstration of Proof-of-Possession) | Sender-constrained access tokens |
| OID4VCI | OpenID for Verifiable Credential Issuance | Pre-authorized code grant, c_nonce, credential endpoint |

---

**Next**: [Implementation Details](./implementation-details.md) -- How each wallet implements these protocols.
