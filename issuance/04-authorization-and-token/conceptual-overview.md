# Authorization in OID4VCI -- Conceptual Overview

Before a wallet can request a credential from an issuer, it must prove that it is authorized to do so. The authorization phase answers two questions: **who is the user?** and **what are they entitled to receive?** The output of this phase is an access token -- a bearer credential that grants the wallet permission to call the issuer's credential endpoint.

This document explains the authorization concepts at a high level. Protocol-level details (PKCE, PAR, DPoP) are covered in [Protocol & Standards](./protocol-and-standards.md).

---

## Why Authorization Is Needed

Credential issuance is not an open endpoint. An issuer must verify the identity of the requesting party before creating a credential that bears the issuer's cryptographic signature. Without authorization:

- Any party could request credentials on behalf of any user.
- The issuer could not enforce policies about who receives which credential types.
- There would be no audit trail linking issued credentials to authenticated sessions.

The authorization phase produces two critical artifacts:

1. **Access token** -- Proves the wallet has been authorized for this issuance session. Included as a Bearer (or DPoP-bound) token in the credential request.
2. **`c_nonce`** -- A credential nonce issued alongside the access token. The wallet must include this nonce in its proof-of-possession, binding the proof to this specific session and preventing replay attacks. (See [05 -- Nonce & Replay Protection](../05-nonce-and-replay-protection/) for details.)

---

## Two Grant Types

OID4VCI defines two paths to obtain an access token, corresponding to two different trust models.

### Pre-Authorized Code Grant

```
Issuer                                 Wallet
  |                                      |
  |  Credential Offer                    |
  |  (includes pre-authorized_code)      |
  |  ----------------------------------> |
  |                                      |
  |  Token Request                       |
  |  (grant_type = pre-authorized_code)  |
  |  <---------------------------------- |
  |                                      |
  |  Token Response                      |
  |  (access_token, c_nonce)             |
  |  ----------------------------------> |
  |                                      |
```

In the Pre-Authorized Code Grant, the issuer has already authenticated the user before creating the credential offer. The credential offer itself contains a `pre-authorized_code` -- a one-time code that the wallet exchanges directly at the token endpoint for an access token.

**When it is used:**

- The issuer knows who the user is (e.g., the user completed an identity verification process on the issuer's website before scanning the QR code).
- The user does not need to log in again during the wallet flow.
- The issuance is a continuation of an existing authenticated session.

**Characteristics:**

- Simpler flow -- no browser redirect, no login screen within the wallet.
- The credential offer may include a `tx_code` requirement (a numeric or alphanumeric code the user must enter), providing out-of-band confirmation that the person holding the wallet is the intended recipient.
- The pre-authorized code is single-use and time-limited.

**Example scenario:** A government portal verifies a citizen's identity online, then displays a QR code. The citizen scans the QR code with their wallet. The wallet extracts the pre-authorized code from the offer, exchanges it for an access token, and proceeds to request the credential. The citizen never sees a login screen in the wallet.

### Authorization Code Grant

```
Issuer / Auth Server                   Wallet                    User
  |                                      |                         |
  |  Credential Offer                    |                         |
  |  (no pre-authorized_code)            |                         |
  |  ----------------------------------> |                         |
  |                                      |                         |
  |           Authorization Request      |                         |
  |  <---------------------------------- |                         |
  |                                      |                         |
  |           Browser Redirect           |                         |
  |  --------------------------------------------------------------------->
  |                                      |     User authenticates  |
  |  <---------------------------------------------------------------------
  |           Authorization Code         |                         |
  |  ----------------------------------> |                         |
  |                                      |                         |
  |           Token Request              |                         |
  |  <---------------------------------- |                         |
  |           (authorization_code)       |                         |
  |                                      |                         |
  |           Token Response             |                         |
  |  ----------------------------------> |                         |
  |           (access_token, c_nonce)    |                         |
  |                                      |                         |
```

In the Authorization Code Grant, the user has not yet authenticated with the issuer. The wallet must redirect the user to the issuer's authorization server (typically a browser-based login page), where the user logs in. After successful authentication, the authorization server redirects back to the wallet with an authorization code, which the wallet exchanges for an access token.

**When it is used:**

- The issuer does not know the user at the time of the credential offer.
- The issuance flow begins from a generic entry point (e.g., a public QR code, a link on a website).
- The issuer requires explicit user authentication via its own identity provider.

**Characteristics:**

- More complex flow -- involves browser navigation, redirect handling, and potential user interaction with a login form.
- Supports standard OAuth 2.0 identity providers, including federated login (e.g., national eID systems).
- Enhanced with PKCE (Proof Key for Code Exchange) to prevent authorization code interception.
- May use PAR (Pushed Authorization Requests) to pre-register the authorization request before redirecting the user.

**Example scenario:** A university posts a QR code on its website for alumni to claim a diploma credential. Any alumnus can scan the code, but they must log in with their university credentials before the credential is issued. The wallet opens a browser, the user enters their username and password, and the browser redirects back to the wallet with an authorization code.

---

## How the Grant Type Is Determined

The grant type is not chosen by the wallet. It is declared by the issuer in the credential offer, within the `grants` object:

- If the offer contains a `urn:ietf:params:oauth:grant-type:pre-authorized_code` grant, the wallet uses the Pre-Authorized Code flow.
- If the offer contains an `authorization_code` grant (or no pre-authorized code), the wallet uses the Authorization Code flow.
- If both are present, the wallet may choose either, though most implementations prefer the pre-authorized code path when available because it is simpler.

---

## The Access Token

Regardless of which grant type is used, the end result is the same: the wallet holds an access token. This token is:

- **Short-lived** -- Typically valid for minutes, not hours or days.
- **Scoped** -- Authorized for credential issuance operations only.
- **Potentially sender-constrained** -- If DPoP is used, the token is bound to the wallet's key pair and cannot be used by another party even if intercepted.

The access token is included in the `Authorization` header of the credential request:

```
POST /credential HTTP/1.1
Authorization: Bearer <access_token>
```

Or, if DPoP is used:

```
POST /credential HTTP/1.1
Authorization: DPoP <access_token>
DPoP: <dpop_proof_jwt>
```

---

## Relationship to Other Issuance Steps

Authorization sits at the center of the issuance flow, connecting prior steps to subsequent ones:

- **Input from Credential Offer (Section 02):** The grant type and pre-authorized code (if any) come from the credential offer.
- **Input from Issuer Metadata (Section 03):** The authorization endpoint, token endpoint, and PAR endpoint URLs come from the issuer's metadata.
- **Output to Nonce & Replay Protection (Section 05):** The `c_nonce` from the token response is the starting point for replay protection.
- **Output to Proof Construction (Section 07):** The access token and `c_nonce` are both required to construct the credential request proof.
- **Output to Credential Request (Section 08):** The access token authorizes the credential request itself.

---

**Next**: [Protocol & Standards](./protocol-and-standards.md) -- OAuth 2.0 with PKCE, PAR, DPoP, and the token endpoint in detail.
