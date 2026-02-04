# Transport Modes — Comparative Analysis

This document compares transport mode implementations across EUDI, Procivis, and Affinidi.

---

## Transport Support Matrix

| Transport | EUDI Android | EUDI iOS | Procivis | Affinidi |
|-----------|--------------|----------|----------|----------|
| **OID4VP (remote)** | ✓ | ✓ | ✓ | ✓ |
| **BLE proximity** | ✓ | ✓ | ✓ | ✗ |
| **NFC engagement** | ✓ | ✓ | ✓ | ✗ |
| **HTTP transport** | ✓ (via SDK) | ✓ (via SDK) | ✓ | ✓ |
| **MQTT transport** | ✗ | ✗ | ✓ | ✗ |
| **WiFi Aware** | ✗ | ✗ | ✗ | ✗ |

---

## URI Scheme Support

| Scheme | EUDI Android | EUDI iOS | Procivis | Affinidi |
|--------|--------------|----------|----------|----------|
| `openid4vp://` | ✓ | ✓ | ✓ | ✓ |
| `eudi-openid4vp://` | ✓ | ✗ | ✗ | ✗ |
| `mdoc-openid4vp://` | ✓ | ✗ | ✗ | ✗ |
| `haip://` | ✓ | ✓ | ✗ | ✗ |
| Universal Links | ✗ | ✓ | ✓ | ✓ |
| Android App Links | ✓ | N/A | ✓ | ✓ |

---

## Same-Device Flow Comparison

### EUDI

```
Deep link (openid4vp://)
    │
    ▼
DeepLinkHelper routes to PresentationRequest
    │
    ▼
SDK fetches request_uri
    │
    ▼
User consents
    │
    ▼
SDK sends response (direct_post)
    │
    ▼
App receives redirect_uri, navigates
```

### Procivis

```
Deep link or Universal Link
    │
    ▼
parseUniversalLink() if needed
    │
    ▼
useInvitationHandling() navigates to Processing
    │
    ▼
Follow HTTP redirects manually
    │
    ▼
core.handleInvitation()
    │
    ▼
User consents
    │
    ▼
core.holderSubmitProof()
    │
    ▼
Navigate based on redirectUri
```

### Affinidi

```
Deep link
    │
    ▼
Affinidi SDK handles
    │
    ▼
Platform UI for consent
    │
    ▼
Response sent via platform
```

---

## Cross-Device Flow Comparison

| Aspect | EUDI | Procivis | Affinidi |
|--------|------|----------|----------|
| **QR display** | SDK generates | Core generates | Platform |
| **Request fetch** | SDK automatic | Manual + Core | Platform |
| **Session correlation** | SDK handles | state parameter | Platform |
| **Result notification** | Redirect URI | Redirect URI | Callback |

---

## Proximity Flow Comparison

### EUDI

**Strengths:**
- Full ISO 18013-5 compliance
- Native BLE implementation
- NFC engagement support
- QR device engagement

**Implementation:**
```kotlin
// Start proximity
eudiWallet.startProximityPresentation()

// Generate QR for engagement
eudiWallet.startQrEngagement()

// Toggle NFC
eudiWallet.enableNFCEngagement(true)
```

### Procivis

**Strengths:**
- Multiple transport options
- Configurable per deployment
- MQTT for IoT scenarios

**Implementation:**
```typescript
config: {
    transport: {
        BLE: { enabled: true },
        HTTP: { enabled: true },
        MQTT: { enabled: true }
    }
}
```

### Affinidi

**Note:** Proximity/BLE not supported. Focus on remote/network-based verification.

---

## Response Mode Support

| Mode | EUDI | Procivis | Affinidi |
|------|------|----------|----------|
| `fragment` | ✗ | ✗ | ✓ |
| `direct_post` | ✓ | ✓ | ✓ |
| `direct_post.jwt` | ✓ | ✗ | ✓ |

---

## Security Comparison

### Deep Link Security

| Aspect | EUDI | Procivis | Affinidi |
|--------|------|----------|----------|
| Custom schemes | 4 registered | 1 standard | 1 standard |
| Universal Links | iOS only | Both platforms | Both platforms |
| Scheme verification | OS level | OS level | OS level |
| App attestation | Planned | N/A | N/A |

### Proximity Security

| Aspect | EUDI | Procivis | Affinidi |
|--------|------|----------|----------|
| BLE encryption | ✓ | ✓ | N/A |
| Reader authentication | X.509 | Configurable | N/A |
| Session binding | ✓ | ✓ | N/A |
| Relay protection | Physical presence | Physical presence | N/A |

---

## User Experience

### Same-Device UX

| Aspect | EUDI | Procivis | Affinidi |
|--------|------|----------|----------|
| App switch | Automatic | Automatic | Automatic |
| Return flow | Redirect | Redirect | Callback |
| Progress indicator | Loading screen | Loading screen | Platform |
| Error handling | In-app error | Toast/screen | Platform |

### Cross-Device UX

| Aspect | EUDI | Procivis | Affinidi |
|--------|------|----------|----------|
| QR display | SDK provided | Core generated | Platform |
| Scan feedback | Immediate | Immediate | Varies |
| Desktop notification | Polling/redirect | Polling/redirect | Varies |
| Timeout handling | Request expiry | Request expiry | Platform |

### Proximity UX

| Aspect | EUDI | Procivis |
|--------|------|----------|
| Engagement method | QR or NFC | QR or NFC |
| Connection indicator | Status event | State update |
| Transfer progress | Events | Events |
| Completion | Success screen | Success screen |

---

## Trade-offs

### EUDI Approach

**Strengths:**
- Comprehensive transport coverage
- Strong ISO 18013-5 compliance
- Multiple scheme support
- Full proximity support

**Weaknesses:**
- Multiple schemes may confuse
- SDK dependency
- Less flexibility

### Procivis Approach

**Strengths:**
- Transport flexibility
- MQTT for IoT
- Configurable per deployment
- Universal link focus

**Weaknesses:**
- Manual redirect handling
- No encrypted response mode
- Single URI scheme

### Affinidi Approach

**Strengths:**
- Simple integration
- Platform-managed
- Consistent UX

**Weaknesses:**
- No proximity support
- Platform dependency
- Limited customization

---

## Recommendations

### Choose EUDI transport if:
- Proximity verification needed
- EU/eIDAS compliance required
- Multiple URI scheme support wanted
- Native mobile app

### Choose Procivis transport if:
- IoT scenarios (MQTT)
- Flexible deployment needed
- Cross-platform React Native
- Custom transport requirements

### Choose Affinidi transport if:
- Remote-only verification
- Simple integration priority
- Platform management acceptable
- Standard web flows

---

## Future Considerations

| Trend | EUDI | Procivis | Affinidi |
|-------|------|----------|----------|
| **WiFi Aware** | Planned | Possible | Unlikely |
| **5G ProSe** | Research | Unknown | Unknown |
| **Bluetooth Mesh** | Research | Possible | Unknown |
| **NFC HCE** | Possible | Possible | N/A |
