# Transport Modes — Protocol and Standards

This document details the protocol specifications for different transport modes in OID4VP and ISO 18013-5.

---

## OID4VP Response Modes

### Authorization Response

The response mode determines how the authorization response is delivered:

```
Request:
  response_mode=direct_post
  response_uri=https://verifier.example.com/callback
```

### Fragment Mode

**When:** `response_mode=fragment` or default for same-device

```
Wallet redirects to:
https://verifier.example.com/callback#
  vp_token=eyJhbGciOiJFUzI1NiIsInR5cCI6IkpXVCJ9...&
  presentation_submission=%7B%22id%22%3A%22...%7D&
  state=af0ifjsldkj
```

**Characteristics:**
- Fragment not sent to server in HTTP request
- Client-side JavaScript extracts response
- Same-origin policy applies
- Suitable for SPAs

### Direct POST Mode

**When:** `response_mode=direct_post`

```http
POST /callback HTTP/1.1
Host: verifier.example.com
Content-Type: application/x-www-form-urlencoded

vp_token=eyJhbGciOiJFUzI1NiIsInR5cCI6IkpXVCJ9...&
presentation_submission=%7B%22id%22%3A%22...%22%7D&
state=af0ifjsldkj
```

**Response from verifier:**

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "redirect_uri": "https://verifier.example.com/success?session=xyz"
}
```

The wallet then navigates to `redirect_uri`.

### Direct POST JWT Mode

**When:** `response_mode=direct_post.jwt`

Request specifies encryption:
```json
{
  "response_mode": "direct_post.jwt",
  "response_uri": "https://verifier.example.com/callback",
  "client_metadata": {
    "authorization_encrypted_response_alg": "ECDH-ES",
    "authorization_encrypted_response_enc": "A256GCM",
    "jwks": { "keys": [{ ... verifier encryption key ... }] }
  }
}
```

Response is encrypted JWT:

```http
POST /callback HTTP/1.1
Host: verifier.example.com
Content-Type: application/x-www-form-urlencoded

response=eyJhbGciOiJFQ0RILUVTI...
```

JWT payload after decryption:
```json
{
  "vp_token": "eyJ...",
  "presentation_submission": { ... },
  "state": "af0ifjsldkj"
}
```

---

## Cross-Device Request URI

For cross-device flows, the QR encodes a minimal request:

```
openid4vp://authorize?
  client_id=https://verifier.example.com&
  request_uri=https://verifier.example.com/requests/abc123
```

**QR Code Constraints:**

| Aspect | Constraint |
|--------|------------|
| Practical size | ~2KB data |
| Encoding | URI with percent-encoding |
| Error correction | Medium (M) recommended |
| Version | QR version 10-15 typically |

The wallet:
1. Scans QR code
2. Extracts `request_uri`
3. Fetches full request: `GET https://verifier.example.com/requests/abc123`
4. Processes request

---

## ISO 18013-5 Transport

### Device Engagement

Initiates proximity session:

**QR Device Engagement:**
```
mdoc://engage?
  e=oWNkZW5n... (CBOR-encoded DeviceEngagement)
```

**NFC NDEF Record:**
```
Type: "application/vnd.iso.18013-5.device-engagement"
Payload: [CBOR DeviceEngagement]
```

### DeviceEngagement Structure

```cbor
DeviceEngagement = {
  0: "1.0",                    ; version
  1: Security,                 ; security parameters
  2: [DeviceRetrievalMethod],  ; retrieval methods
  ? 3: ServerRetrievalMethods, ; optional server methods
  ? 4: ProtocolInfo            ; optional protocol info
}

Security = [
  CipherSuiteIdentifier,       ; 1 = EC-P256
  EDeviceKey_Pub               ; ephemeral device public key
]

DeviceRetrievalMethod = [
  Type,     ; 1=NFC, 2=BLE, 3=WiFi
  Options   ; type-specific options
]
```

### BLE Options

```cbor
BleOptions = {
  0: bool,  ; Supports Central Client Mode
  1: bool,  ; Supports Peripheral Server Mode
  ? 10: uuid,  ; Peripheral Server UUID
  ? 11: bytes  ; Central Client UUID
}
```

### Session Establishment

After device engagement:

1. **BLE Connection**: Device connects using UUID from engagement
2. **Session Encryption**: HKDF-derived keys from ECDH
3. **Request**: Reader sends encrypted `DeviceRequest`
4. **Response**: Device sends encrypted `DeviceResponse`

### SessionTranscript

Binds engagement to session:

```cbor
SessionTranscript = [
  DeviceEngagementBytes,
  EReaderKeyBytes,
  Handover
]
```

Used in:
- Session key derivation
- Reader authentication
- Device authentication

---

## BLE Data Transfer

### Characteristics

| Characteristic | UUID | Direction |
|----------------|------|-----------|
| State | `00000001-...` | Notify |
| Client2Server | `00000002-...` | Write |
| Server2Client | `00000003-...` | Read/Notify |

### Message Framing

Large messages split into chunks:

```
First chunk:  [0x01] [length] [data...]
Middle chunk: [0x00] [data...]
Last chunk:   [0x00] [data...] (when complete)
```

### Maximum Data Size

| Method | Max Transfer |
|--------|--------------|
| BLE | ~500 bytes per characteristic |
| NFC | Varies by device |
| WiFi Aware | Higher throughput |

---

## Deep Link Schemes

### OID4VP Schemes

| Scheme | Purpose |
|--------|---------|
| `openid4vp://` | Standard OID4VP |
| `eudi-openid4vp://` | EUDI-specific |
| `mdoc-openid4vp://` | mDoc-specific |
| `haip://` | High Assurance Interoperability |

### Universal Links (iOS)

```json
{
  "applinks": {
    "details": [{
      "appIDs": ["TEAMID.com.example.wallet"],
      "components": [{
        "/": "/verify/*",
        "comment": "Verification deep links"
      }]
    }]
  }
}
```

### Android App Links

```xml
<intent-filter android:autoVerify="true">
  <action android:name="android.intent.action.VIEW" />
  <category android:name="android.intent.category.DEFAULT" />
  <category android:name="android.intent.category.BROWSABLE" />
  <data
    android:scheme="https"
    android:host="verify.example.com"
    android:pathPattern="/verify/.*" />
</intent-filter>
```

---

## Session Management

### Cross-Device Polling

Verifier polls for response:

```http
GET /sessions/abc123/status HTTP/1.1
Host: verifier.example.com

HTTP/1.1 200 OK
{
  "status": "pending"  // or "completed", "expired"
}
```

### WebSocket Notification

Real-time notification:

```javascript
ws.onmessage = (event) => {
  const data = JSON.parse(event.data);
  if (data.status === 'completed') {
    // VP received, process result
  }
};
```

### Request Expiration

```json
{
  "request_uri": "https://verifier.example.com/requests/abc123",
  "expires_in": 300
}
```

Wallet must fetch within expiration window.

---

## Standards Reference

| Standard | Section | Topic |
|----------|---------|-------|
| OID4VP | 6 | Response Mode |
| OID4VP | 7 | Direct POST |
| ISO 18013-5 | 8 | Device Engagement |
| ISO 18013-5 | 8.3 | BLE Transport |
| ISO 18013-5 | 8.2 | NFC Transport |
| RFC 6749 | 4.1 | OAuth 2.0 Flow |
| BLE Core Spec | GATT | Characteristic Operations |
