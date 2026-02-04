# Nonce & Replay Protection -- Conceptual Overview

Nonces are the mechanism by which OID4VCI ensures that every credential request is fresh, unique, and tied to a specific issuance session. Without nonces, an attacker who captures a valid credential request could replay it to obtain duplicate credentials. Transaction codes (TxCodes) complement nonces by adding a human verification layer that confirms the physical user is the intended credential recipient.

This document explains the concepts. Protocol-level encoding and field names are covered in [Protocol & Standards](./protocol-and-standards.md).

---

## What Is a c_nonce?

A `c_nonce` (credential nonce) is a random, single-use string issued by the credential issuer. It serves one purpose: **to bind a proof-of-possession to a specific issuance session**.

When the wallet requests an access token (via either the Pre-Authorized Code or Authorization Code grant), the token response includes a `c_nonce` value. The wallet must then include this nonce in the proof JWT it constructs for the credential request. The issuer verifies that the nonce in the proof matches the nonce it issued, confirming that:

1. The proof was generated after the token was issued (freshness).
2. The proof is intended for this specific session (session binding).
3. The proof has not been copied from a different session (replay prevention).

---

## How Replay Attacks Work

Without nonces, the following attack is possible:

```
Attacker intercepts a valid credential request:

  POST /credential
  Authorization: Bearer <stolen_token>
  Body: { proof: <captured_proof>, ... }

Attacker replays the exact same request to the issuer:

  POST /credential
  Authorization: Bearer <stolen_token>
  Body: { proof: <captured_proof>, ... }

  --> Issuer issues a second credential to the attacker's session.
```

With nonces, this attack fails because:

- The `c_nonce` embedded in the captured proof has already been consumed by the issuer.
- The issuer rejects any proof containing a previously used nonce.
- Even if the attacker obtains a new access token, they cannot construct a valid proof without the wallet's private key and a fresh nonce.

---

## The Nonce Lifecycle

The `c_nonce` follows a specific lifecycle across the issuance protocol:

```
Token Endpoint                     Wallet                      Credential Endpoint
     |                                |                               |
     |  Token Response                |                               |
     |  { access_token,               |                               |
     |    c_nonce: "nonce_1",         |                               |
     |    c_nonce_expires_in: 300 }   |                               |
     |  ----------------------------> |                               |
     |                                |                               |
     |                                |  Construct proof JWT          |
     |                                |  { nonce: "nonce_1", ... }    |
     |                                |                               |
     |                                |  Credential Request           |
     |                                |  { proof: <jwt> }             |
     |                                |  ------------------------------>
     |                                |                               |
     |                                |  Credential Response          |
     |                                |  { credential: <vc>,          |
     |                                |    c_nonce: "nonce_2",        |
     |                                |    c_nonce_expires_in: 300 }  |
     |                                |  <------------------------------
     |                                |                               |
```

### Step 1: Nonce Issuance

The token endpoint includes `c_nonce` and `c_nonce_expires_in` in the token response. The `c_nonce` is a random string; `c_nonce_expires_in` specifies how many seconds the nonce remains valid.

### Step 2: Nonce Consumption

The wallet includes the `c_nonce` value as the `nonce` claim in the proof JWT payload. This proof is submitted as part of the credential request.

### Step 3: Nonce Rotation (Optional)

The credential response may include a new `c_nonce`. If the wallet needs to make additional credential requests (e.g., for batch issuance of multiple credentials under the same offer), it must use this new nonce, not the original one. Each nonce is consumed exactly once.

### Nonce Expiration

The `c_nonce_expires_in` field provides a time window during which the nonce is valid. If the wallet does not submit a credential request within this window, the nonce expires and the wallet must obtain a new one (typically by re-authenticating at the token endpoint).

---

## Transaction Codes (TxCode)

Transaction codes serve a fundamentally different purpose than cryptographic nonces. While `c_nonce` prevents technical replay attacks, TxCodes prevent **social** attacks -- they confirm that the person physically interacting with the wallet is the person the issuer intended to receive the credential.

### How TxCodes Work

1. The issuer includes a `tx_code` requirement in the credential offer's grant object.
2. The issuer delivers the actual transaction code to the user through an out-of-band channel (email, SMS, postal mail, in-person handoff).
3. When the wallet initiates the token exchange, it prompts the user to enter the transaction code.
4. The wallet includes the user-entered code in the token request.
5. The issuer validates the code before issuing the access token.

### TxCode vs. c_nonce

| Property | c_nonce | TxCode |
|----------|---------|--------|
| Purpose | Cryptographic session binding | Human identity verification |
| Generated by | Issuer (token/credential endpoint) | Issuer (out-of-band) |
| Delivered to | Wallet (in protocol response) | User (via email, SMS, etc.) |
| Entered by | Wallet (automatically in proof JWT) | User (manually in wallet UI) |
| Prevents | Replay attacks, proof reuse | Unauthorized credential claiming |
| Format | Opaque string (any characters) | Typically numeric (4-8 digits) |

### TxCode Is Not a Nonce

Although TxCodes are sometimes called "transaction PINs" or "one-time codes," they are not cryptographic nonces in the protocol sense. They do not appear in JWT proofs and they do not provide replay protection. Their value is in confirming user intent through an out-of-band channel, complementing the cryptographic protections provided by `c_nonce` and proof-of-possession.

---

## Why Both Mechanisms Are Needed

Consider an issuance scenario without TxCode: an attacker finds a QR code containing a pre-authorized credential offer (perhaps discarded or photographed). The attacker scans it with their own wallet. The pre-authorized code is valid, so the attacker receives an access token, constructs a valid proof with the `c_nonce`, and obtains the credential. The cryptographic nonce prevented replay, but it did not prevent initial unauthorized claiming.

With a TxCode requirement, the attacker cannot complete the token exchange because they do not know the transaction code that was sent to the legitimate user via a separate channel.

Conversely, a TxCode alone (without `c_nonce`) would not prevent an attacker from intercepting a legitimate wallet's credential request and replaying it.

Both mechanisms address different threat vectors and are complementary.

---

**Next**: [Protocol & Standards](./protocol-and-standards.md) -- The technical encoding of nonces, JWT claims, and TxCode specification parameters.
