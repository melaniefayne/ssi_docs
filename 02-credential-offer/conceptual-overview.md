# Credential Offer -- Conceptual Overview

A **Credential Offer** is the entry point for the OID4VCI issuance flow. It is a structured object that an issuer creates and delivers to a wallet, signaling that one or more credentials are available for issuance and providing the information the wallet needs to begin the protocol exchange.

---

## The OID4VCI Credential Offer Object

The Credential Offer is a JSON object with a well-defined structure specified by the OID4VCI specification. It contains everything the wallet needs to identify the issuer, understand what credentials are being offered, and determine how to obtain them.

### Core Structure

```json
{
  "credential_issuer": "https://issuer.example.com",
  "credential_configuration_ids": [
    "org.iso.18013.5.1.mDL",
    "eu.europa.ec.eudi.pid.1"
  ],
  "grants": {
    "urn:ietf:params:oauth:grant-type:pre-authorized_code": {
      "pre-authorized_code": "SplxlOBeZQQYbYS6WxSbIA",
      "tx_code": {
        "input_mode": "numeric",
        "length": 6,
        "description": "Enter the code sent to your email"
      }
    },
    "authorization_code": {
      "issuer_state": "eyJhbGciOiJSU0..."
    }
  }
}
```

### Fields

| Field | Required | Description |
|-------|----------|-------------|
| `credential_issuer` | Yes | The URL identifying the credential issuer. The wallet uses this to retrieve issuer metadata from the `/.well-known/openid-credential-issuer` endpoint. |
| `credential_configuration_ids` | Yes | An array of identifiers referencing specific credential types the issuer is offering. These IDs correspond to entries in the issuer's `credential_configurations_supported` metadata. |
| `grants` | No | An object specifying one or more grant types the wallet can use to obtain an access token. If omitted, the wallet must determine the grant type from the issuer's metadata. |

### Delivery Parameters

The Credential Offer reaches the wallet via one of two URI parameters:

- **`credential_offer`** -- The offer object is included directly as a URL-encoded JSON value in the URI query string. Used when the offer is small enough to fit in a URI (typical for QR codes and deep links).
- **`credential_offer_uri`** -- A URL pointing to where the full offer object can be fetched. Used when the offer is too large for inline delivery or when the issuer wants to control offer lifecycle (e.g., single-use offers that expire after retrieval).

---

## Issuer-Initiated vs Wallet-Initiated Flows

The OID4VCI specification supports two initiation models, which differ in who triggers the issuance process.

### Issuer-Initiated Flow

The issuer creates a Credential Offer and delivers it to the wallet. This is the dominant model across all implementations studied.

```
Issuer                              Wallet
  |                                   |
  |  1. Create Credential Offer       |
  |  2. Deliver via QR/deep link/etc  |
  |---------------------------------->|
  |                                   |  3. Parse offer
  |                                   |  4. Resolve issuer metadata
  |                                   |  5. Begin token exchange
  |<----------------------------------|
  |                                   |
```

The issuer controls what credentials are offered, which grant types are available, and whether a transaction code is required. The user's role is to scan or tap to accept the offer.

### Wallet-Initiated Flow

The wallet discovers available credentials from an issuer's metadata and requests issuance without a prior offer from the issuer.

```
Wallet                              Issuer
  |                                   |
  |  1. Discover issuer metadata      |
  |---------------------------------->|
  |  2. Return metadata               |
  |<----------------------------------|
  |  3. Select credential type        |
  |  4. Begin authorization           |
  |---------------------------------->|
  |                                   |
```

In the wallet-initiated flow, the wallet queries the issuer's `/.well-known/openid-credential-issuer` endpoint to discover what credential types are supported, then initiates the authorization and issuance process. This flow always uses the Authorization Code Grant since there is no pre-authorized context.

**Implementation note:** The EUDI wallets support wallet-initiated issuance through their scoped document discovery mechanism (`getScopedDocuments()`), which fetches metadata from all configured issuers and presents available credentials to the user. Procivis ONE and Affinidi primarily focus on issuer-initiated flows.

---

## Delivery Mechanisms

The Credential Offer must be transported from the issuer to the wallet. The delivery mechanism determines the user experience, the deployment context, and the security properties of the offer exchange.

### QR Codes

The most common delivery mechanism. The issuer renders the Credential Offer URI as a QR code, which the user scans with the wallet application.

- **Format:** The QR code encodes the full `openid-credential-offer://` URI, including either the `credential_offer` or `credential_offer_uri` parameter.
- **Context:** Typically displayed on a web page after the user completes an application or identity verification process, or on a physical document.
- **Limitations:** QR codes have a practical size limit (~2,953 bytes for alphanumeric content at error correction level L). Large offers should use `credential_offer_uri` to keep the QR code scannable.

### Deep Links

A deep link is a URI that, when opened on a device, launches the wallet application directly. This mechanism works when the offer originates from a digital context (email, SMS, web page) rather than a physical one.

- **Format:** Same `openid-credential-offer://` URI scheme as QR codes, but delivered as a clickable link.
- **Context:** The user receives the link via email, SMS, or in-app notification and taps it to open the wallet.
- **Advantage:** No camera or scanning required. Works on the same device where the wallet is installed.

### Universal Links / App Links

Platform-specific mechanisms (iOS Universal Links, Android App Links) that map HTTPS URLs to specific applications. The issuer hosts a standard HTTPS URL that, when opened on a device with the wallet installed, routes to the wallet application instead of the browser.

- **Advantage:** Falls back gracefully to a web page if the wallet is not installed. No custom URI scheme registration required.
- **Requirement:** Requires the issuer to configure association files (`.well-known/apple-app-site-association` on iOS, `/.well-known/assetlinks.json` on Android).

### NFC

Near Field Communication allows the offer to be delivered via physical proximity. The user taps their device against an NFC tag or terminal that contains the Credential Offer URI.

- **Context:** Physical issuance scenarios -- government offices, service counters, event registration.
- **Advantage:** Works without internet connectivity at the point of offer delivery (though the wallet still needs connectivity to complete issuance).

### BLE and MQTT (Procivis ONE)

Procivis ONE extends beyond standard HTTP delivery to support Bluetooth Low Energy (BLE) and MQTT as transport mechanisms for credential offers.

- **BLE:** Enables offer delivery in close-proximity, offline-capable scenarios.
- **MQTT:** Enables asynchronous, message-broker-mediated offer delivery. Useful for IoT and enterprise contexts.
- **Transport detection:** The wallet inspects the URL scheme and structure to determine which transport to use: MQTT URLs route to the MQTT transport, BLE-specific URLs route to Bluetooth, and standard HTTPS URLs route to HTTP.

---

## Grant Types in the Offer

The `grants` object in the Credential Offer specifies how the wallet should obtain an access token. Two grant types are defined by OID4VCI.

### Pre-Authorized Code Grant

Used when the issuer has already verified the holder's identity or entitlement before creating the offer. The offer includes a `pre-authorized_code` that the wallet exchanges directly for an access token at the token endpoint, bypassing the authorization endpoint entirely.

This grant type may optionally include a `tx_code` requirement, which adds a second-factor verification step: the user must enter a transaction code (received out-of-band from the issuer) before the wallet can exchange the pre-authorized code for a token.

**When used:** The issuer has completed identity verification (e.g., in-person ID check, existing account verification) and wants to issue a credential without requiring the user to authenticate again.

### Authorization Code Grant

Used when the wallet needs to authenticate the user through an OAuth 2.0 authorization server before issuance. The offer may include an `issuer_state` parameter that binds the authorization request to the specific offer context.

**When used:** The issuer requires the user to authenticate or consent at issuance time, or the issuer delegates identity verification to an external identity provider.

### When Both Are Present

An offer may include both grant types, giving the wallet the choice of which flow to follow. The wallet typically prefers the Pre-Authorized Code Grant when available, as it requires fewer round trips.

---

## What Happens After the Offer

Once the wallet receives and parses a Credential Offer, the issuance flow proceeds through several stages:

1. **Issuer Metadata Resolution** -- The wallet fetches the issuer's metadata to understand its capabilities, supported credential formats, and cryptographic requirements (see [03 -- Issuer Metadata](../03-issuer-metadata/)).
2. **Authorization / Token Exchange** -- The wallet obtains an access token using the grant type specified in the offer (see [04 -- Authorization & Token](../04-authorization-and-token/)).
3. **Credential Request** -- The wallet constructs and sends a credential request to the issuer's credential endpoint (see [08 -- Credential Request](../08-credential-request/)).
4. **Secure Storage** -- The issued credential is stored securely on the device (see [10 -- Secure Storage](../10-secure-storage/)).

The Credential Offer is the trigger for this entire chain. Its structure determines the path the wallet takes through the remaining protocol steps.
