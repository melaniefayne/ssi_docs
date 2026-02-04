# Presentation Response — Comparative Analysis

This document compares presentation response handling across EUDI, Procivis, and Affinidi.

---

## Response Generation

| Aspect | EUDI | Procivis | Affinidi |
|--------|------|----------|----------|
| **Generation** | SDK method | Core library | Platform |
| **Format** | Format-specific | Format-specific | Platform |
| **Signing** | SDK handles | Core handles | Platform |

---

## Supported Formats

| Format | EUDI | Procivis | Affinidi |
|--------|------|----------|----------|
| JWT VP | ✓ | ✓ | ✓ |
| SD-JWT + KB | ✓ | ✓ | ✓ |
| mDoc DeviceResponse | ✓ | ✓ | ✗ |
| JSON-LD VP | ✗ | ✓ | ✓ |

---

## Response Delivery

| Mode | EUDI | Procivis | Affinidi |
|------|------|----------|----------|
| `direct_post` | ✓ | ✓ | ✓ |
| `direct_post.jwt` | ✓ | ✗ | ✓ |
| `fragment` | ✗ | ✗ | ✓ |

---

## Redirect Handling

### EUDI

```kotlin
// Check for redirect URI in response
redirectUri?.let {
    NavigationType.Deeplink(it.toString(), initiatorRoute)
} ?: NavigationType.PopTo(DashboardScreens.Dashboard)
```

### Procivis

```typescript
const redirectUri = proof?.redirectUri;
if (redirectUri) {
    Linking.openURL(redirectUri);
}
```

### Affinidi

Platform handles redirect automatically.

---

## Error Handling

| Error Type | EUDI | Procivis | Affinidi |
|------------|------|----------|----------|
| Network failure | State event | Mutation error | Callback |
| Signing failure | State event | Exception | Callback |
| User cancel | State event | Navigation | Callback |

---

## Trade-offs

### EUDI

**Strengths:**
- Native SDK handling
- Platform-optimized
- Full format support

**Weaknesses:**
- SDK dependency
- Less flexibility

### Procivis

**Strengths:**
- Cross-platform
- Flexible selection
- V1/V2 support

**Weaknesses:**
- Core library dependency
- More complex state

### Affinidi

**Strengths:**
- Simple integration
- Managed delivery

**Weaknesses:**
- Platform dependency
- Limited formats

---

## Recommendations

### Choose EUDI-style if:
- Native mobile app
- Full mDoc support needed
- Platform-specific optimization

### Choose Procivis-style if:
- Cross-platform needed
- Multiple format support
- Flexible architecture

### Choose Affinidi-style if:
- Rapid development
- Web-focused
- Simpler requirements
