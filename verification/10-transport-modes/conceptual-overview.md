# Transport Modes — Conceptual Overview

Transport modes determine how presentation requests reach wallets and how responses return to verifiers. The choice of transport affects security, user experience, and deployment scenarios.

---

## Transport Classification

```
┌─────────────────────────────────────────────────────────────────┐
│                      TRANSPORT MODES                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  BY DEVICE RELATIONSHIP:                                         │
│  ┌───────────────────┐    ┌───────────────────┐                 │
│  │   Same-Device     │    │   Cross-Device    │                 │
│  │   (wallet & web   │    │   (wallet on      │                 │
│  │    on same phone) │    │    phone, verifier│                 │
│  │                   │    │    on desktop)    │                 │
│  └───────────────────┘    └───────────────────┘                 │
│                                                                  │
│  BY CHANNEL:                                                     │
│  ┌───────────────────┐    ┌───────────────────┐                 │
│  │   Remote          │    │   Proximity       │                 │
│  │   (over network)  │    │   (physical       │                 │
│  │                   │    │    presence)      │                 │
│  └───────────────────┘    └───────────────────┘                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Same-Device Flow

Verifier and wallet on the same device (e.g., mobile browser and wallet app):

```
┌─────────────────────────────────────────────────────────────────┐
│                         MOBILE DEVICE                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────┐                    ┌─────────────────┐     │
│  │  Mobile Browser │                    │   Wallet App    │     │
│  │                 │   1. Deep link     │                 │     │
│  │  Verifier page  │ ──────────────────►│  Receives       │     │
│  │                 │   openid4vp://     │  request        │     │
│  │                 │                    │                 │     │
│  │                 │   4. Redirect      │  2. User        │     │
│  │  Receives       │ ◄──────────────────│  consents       │     │
│  │  result         │   back to app      │                 │     │
│  │                 │                    │  3. Sends VP    │     │
│  └─────────────────┘                    └─────────────────┘     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Characteristics

| Aspect | Value |
|--------|-------|
| Request delivery | Deep link / app-to-app intent |
| Response delivery | Redirect or direct_post |
| User action | App switch (may be seamless) |
| Network | Shared device network context |
| Security | App verification via OS |

### Use Cases

- Mobile web login
- In-app verification
- Mobile-first services

---

## Cross-Device Flow

Verifier on one device, wallet on another (e.g., desktop browser and mobile wallet):

```
┌───────────────────┐                    ┌───────────────────┐
│  Desktop Browser  │                    │  Mobile Device    │
│                   │                    │                   │
│  ┌─────────────┐  │                    │  ┌─────────────┐  │
│  │   Verifier  │  │   1. Display QR    │  │   Wallet    │  │
│  │   Website   │  │ ─────────────────► │  │   App       │  │
│  │             │  │      (User scans)  │  │             │  │
│  │  █████████  │  │                    │  │  Camera     │  │
│  │  █ QR    █  │  │                    │  │  scans QR   │  │
│  │  █████████  │  │                    │  │             │  │
│  │             │  │   2. Fetch request │  │             │  │
│  │             │  │ ◄──(HTTPS)──────── │  │             │  │
│  │             │  │                    │  │             │  │
│  │             │  │   3. User consents │  │             │  │
│  │             │  │                    │  │  ┌───────┐  │  │
│  │             │  │   4. POST VP       │  │  │Share? │  │  │
│  │  Waiting... │  │ ◄──(direct_post)── │  │  └───────┘  │  │
│  │             │  │                    │  │             │  │
│  │  ✓ Verified │  │   5. Poll/push     │  │  ✓ Sent    │  │
│  └─────────────┘  │      result        │  └─────────────┘  │
│                   │                    │                   │
└───────────────────┘                    └───────────────────┘
```

### Characteristics

| Aspect | Value |
|--------|-------|
| Request delivery | QR code → request_uri fetch |
| Response delivery | direct_post to verifier backend |
| User action | Scan QR, approve on phone |
| Network | Separate network contexts |
| Security | QR limits payload; HTTPS for data |

### Use Cases

- Desktop web login
- Kiosk verification
- POS terminals
- Access control

---

## Proximity Flow (BLE)

Physical presence verification using Bluetooth Low Energy:

```
┌───────────────────┐                    ┌───────────────────┐
│  Verifier Device  │                    │  Holder Device    │
│  (Terminal/Phone) │                    │  (Phone)          │
│                   │                    │                   │
│  ┌─────────────┐  │   1. NFC tap or    │  ┌─────────────┐  │
│  │   Reader    │  │      QR scan       │  │   Wallet    │  │
│  │   App       │  │ ─────────────────► │  │   App       │  │
│  │             │  │   (Device engage)  │  │             │  │
│  │             │  │                    │  │             │  │
│  │             │  │   2. BLE connect   │  │             │  │
│  │             │  │ ◄────────────────► │  │             │  │
│  │             │  │                    │  │             │  │
│  │             │  │   3. Request       │  │             │  │
│  │             │  │ ──────(BLE)──────► │  │             │  │
│  │             │  │                    │  │             │  │
│  │             │  │   4. User consent  │  │  ┌───────┐  │  │
│  │             │  │                    │  │  │Share? │  │  │
│  │             │  │   5. Response      │  │  └───────┘  │  │
│  │  ✓ Verified │  │ ◄─────(BLE)─────── │  │             │  │
│  └─────────────┘  │                    │  └─────────────┘  │
│                   │                    │                   │
└───────────────────┘                    └───────────────────┘
```

### ISO 18013-5 Device Engagement

The proximity flow starts with device engagement:

```
Device Engagement (QR or NFC):
{
  "version": "1.0",
  "security": {
    "cipherSuiteIdentifier": 1,
    "deviceEngagementKey": { ... }
  },
  "deviceRetrievalMethods": [{
    "type": 2,  // BLE
    "options": {
      "centralClientMode": true,
      "peripheralServerUUID": "..."
    }
  }]
}
```

### Characteristics

| Aspect | Value |
|--------|-------|
| Request delivery | BLE after engagement |
| Response delivery | BLE |
| User action | Tap/scan, then approve |
| Network | No internet required |
| Security | Physical presence, encrypted BLE |

### Use Cases

- ID verification (age check, identity)
- Border control
- Building access
- Healthcare identification

---

## NFC Proximity

Near Field Communication for very short range:

```
┌───────────────────┐
│                   │
│  ┌─────────────┐  │
│  │   Reader    │  │
│  │             │  │         ┌─────────────────┐
│  │    [NFC]    │◄───tap────►│   📱 Phone     │
│  │             │  │         │   with Wallet   │
│  └─────────────┘  │         └─────────────────┘
│                   │
└───────────────────┘
```

### Characteristics

| Aspect | Value |
|--------|-------|
| Range | ~4cm |
| Data rate | Low (NDEF record) |
| Use | Device engagement trigger |
| Follow-up | Usually BLE for data transfer |

---

## Response Modes (OID4VP)

### `fragment`

Response in URL fragment (same-device):

```
App receives:
myapp://callback#vp_token=eyJ...&presentation_submission={...}
```

- Fragment never sent to server
- Wallet directly returns to verifier app
- Same-device only

### `direct_post`

HTTP POST to verifier backend:

```http
POST /callback HTTP/1.1
Host: verifier.example.com
Content-Type: application/x-www-form-urlencoded

vp_token=eyJ...&presentation_submission=%7B...%7D&state=abc123
```

- Crosses network boundary
- Works for cross-device
- Verifier backend receives directly

### `direct_post.jwt`

Encrypted/signed POST:

```http
POST /callback HTTP/1.1
Host: verifier.example.com
Content-Type: application/x-www-form-urlencoded

response=eyJ...
```

- JWT-wrapped response
- Can be encrypted to verifier's key
- Higher security

---

## Security Considerations

### Same-Device

| Risk | Mitigation |
|------|------------|
| App impersonation | OS app verification, custom schemes |
| Intent interception | Verified deep links |
| Replay in browser | Nonce binding |

### Cross-Device

| Risk | Mitigation |
|------|------------|
| QR screenshot | Short-lived request_uri |
| Session hijacking | State parameter |
| Network eavesdrop | HTTPS everywhere |

### Proximity

| Risk | Mitigation |
|------|------------|
| Relay attack | Physical presence check, timing |
| Eavesdropping | BLE encryption |
| Unauthorized reader | Reader authentication |

---

## Choosing a Transport

| Factor | Same-Device | Cross-Device | Proximity |
|--------|-------------|--------------|-----------|
| **Network required** | For response | Yes | No |
| **Physical presence** | No | No | Yes |
| **User experience** | Seamless | Manual scan | Tap/scan |
| **Security level** | Medium | Medium | High |
| **Use case** | Mobile web | Desktop web | In-person |
| **Offline capable** | Partial | No | Yes |
