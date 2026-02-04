# Session Security

This document covers cryptographic session security for verification flows, including nonce handling, channel security, and replay prevention.

---

## Session Lifecycle

```
┌─────────────────────────────────────────────────────────────────┐
│                     SESSION LIFECYCLE                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. SESSION CREATION                                             │
│     ├── Generate session ID                                      │
│     ├── Generate nonce                                           │
│     ├── Store session state                                      │
│     └── Set expiration                                           │
│                                                                  │
│  2. REQUEST DELIVERY                                             │
│     ├── Include nonce in request                                 │
│     ├── Sign request (optional)                                  │
│     └── Track session state                                      │
│                                                                  │
│  3. RESPONSE PROCESSING                                          │
│     ├── Validate nonce match                                     │
│     ├── Consume nonce (single-use)                               │
│     ├── Validate session not expired                             │
│     └── Process response                                         │
│                                                                  │
│  4. SESSION CLEANUP                                              │
│     ├── Mark session complete/failed                             │
│     └── Remove sensitive data                                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Nonce Generation

### Requirements

| Property | Specification |
|----------|---------------|
| **Entropy** | ≥ 128 bits (256 recommended) |
| **Source** | CSPRNG only |
| **Uniqueness** | Never reused |
| **Format** | URL-safe (base64url) |

### Implementation

```python
import secrets
import base64

def generate_nonce():
    # Generate 256 bits of randomness
    random_bytes = secrets.token_bytes(32)

    # Encode as URL-safe string
    nonce = base64.urlsafe_b64encode(random_bytes).decode('ascii')

    # Remove padding for cleaner URLs
    return nonce.rstrip('=')

# Example: "dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk"
```

### Storage

```python
class NonceStore:
    def __init__(self, ttl_seconds=300):
        self.ttl = ttl_seconds
        self.store = {}  # nonce → (session_id, created_at)

    def create(self, session_id):
        nonce = generate_nonce()
        self.store[nonce] = (session_id, time.time())
        return nonce

    def validate_and_consume(self, nonce, session_id):
        if nonce not in self.store:
            raise InvalidNonce("Unknown nonce")

        stored_session, created_at = self.store[nonce]

        if stored_session != session_id:
            raise InvalidNonce("Session mismatch")

        if time.time() - created_at > self.ttl:
            del self.store[nonce]
            raise InvalidNonce("Nonce expired")

        # Consume nonce (single-use)
        del self.store[nonce]
        return True
```

---

## State Parameter

The `state` parameter correlates request and response:

### Purpose

```
Verifier:
  state = generate_unique_state()
  store(state → user_session)
  send_request(state=state)

Wallet:
  // Include state in response
  response(state=state, vp_token=...)

Verifier:
  session = lookup(response.state)
  // Continue user's session
```

### Security Properties

| Attack | Prevention |
|--------|------------|
| CSRF | State binds response to session |
| Session fixation | Unpredictable state value |
| Response injection | State must match |

### Implementation

```python
def create_request(session_id):
    state = secrets.token_urlsafe(32)
    nonce = generate_nonce()

    # Store mapping
    session_store[state] = {
        'session_id': session_id,
        'nonce': nonce,
        'created_at': time.time()
    }

    return {
        'state': state,
        'nonce': nonce,
        # ... other request params
    }

def process_response(state, vp_token):
    if state not in session_store:
        raise InvalidState("Unknown state")

    session = session_store.pop(state)  # Consume state

    # Validate nonce in VP
    vp_nonce = extract_nonce(vp_token)
    if vp_nonce != session['nonce']:
        raise InvalidNonce()

    return session['session_id']
```

---

## Channel Security

### HTTPS Requirements

| Aspect | Requirement |
|--------|-------------|
| Protocol | TLS 1.2+ (TLS 1.3 recommended) |
| Certificates | Valid, not self-signed |
| Cipher suites | AEAD (AES-GCM, ChaCha20-Poly1305) |
| Key exchange | ECDHE |

### Direct POST Security

```http
POST /callback HTTP/1.1
Host: verifier.example.com
Content-Type: application/x-www-form-urlencoded
```

- Response travels server-to-server
- HTTPS encrypts in transit
- No browser exposure

### Response Encryption

For additional protection, use `direct_post.jwt`:

```python
def encrypt_response(vp_token, verifier_public_key):
    # JWE encryption
    header = {
        "alg": "ECDH-ES+A256KW",
        "enc": "A256GCM",
        "kid": verifier_public_key['kid']
    }

    # Generate ephemeral key pair
    ephemeral = generate_ec_keypair()

    # ECDH key agreement
    shared_secret = ecdh(ephemeral.private, verifier_public_key)

    # Derive content encryption key
    cek = hkdf(shared_secret, ...)

    # Encrypt VP token
    ciphertext = aes_gcm_encrypt(cek, vp_token)

    return jwe(header, ciphertext, ephemeral.public)
```

---

## Proximity Session Security

### ISO 18013-5 Session Encryption

```
Session Key Derivation:

1. Device Engagement
   - Device generates EDeviceKey (ephemeral EC key pair)
   - Device publishes EDeviceKey.public in QR/NFC

2. Session Establishment
   - Reader generates EReaderKey (ephemeral EC key pair)
   - Reader sends EReaderKey.public

3. Shared Secret
   ZAB = ECDH(EDeviceKey.private, EReaderKey.public)
       = ECDH(EReaderKey.private, EDeviceKey.public)

4. Key Derivation
   SessionTranscript = [DeviceEngagementBytes, EReaderKeyBytes, Handover]
   salt = SHA-256(SessionTranscript)

   SKDevice = HKDF-SHA256(salt, ZAB, "SKDevice", 32)
   SKReader = HKDF-SHA256(salt, ZAB, "SKReader", 32)

5. Encrypted Communication
   Device → Reader: AES-GCM(SKDevice, message)
   Reader → Device: AES-GCM(SKReader, message)
```

### BLE Security

| Layer | Protection |
|-------|------------|
| BLE pairing | None (device engagement handles trust) |
| Application | ISO 18013-5 session encryption |
| Anti-relay | Session transcript binding |

---

## Replay Prevention

### Attack Scenario

```
Time T1:
  Legitimate holder presents to Verifier A
  Attacker captures network traffic

Time T2:
  Attacker replays captured VP to Verifier A
  Attacker replays captured VP to Verifier B
```

### Prevention Mechanisms

**1. Nonce (Session Binding)**
```
VP must contain nonce matching current request
Old VP has old nonce → rejected
```

**2. Audience (Verifier Binding)**
```
VP contains aud = "https://verifier-a.example.com"
Replay to Verifier B → aud mismatch → rejected
```

**3. Issued At (Time Binding)**
```
VP contains iat = 1683000000
Verifier checks: iat within acceptable window
Old VP has old iat → rejected
```

**4. Single-Use Nonce**
```
Verifier consumes nonce after use
Replay with same nonce → nonce not found → rejected
```

### Combined Defense

```python
def validate_replay_protection(vp, expected_nonce, verifier_id, max_age=300):
    # Extract VP claims
    nonce = vp.nonce
    aud = vp.aud
    iat = vp.iat

    # Check nonce
    if nonce != expected_nonce:
        raise ReplayError("Nonce mismatch")

    # Check audience
    if aud != verifier_id:
        raise ReplayError("Audience mismatch")

    # Check freshness
    now = time.time()
    if iat > now + 60:  # Clock skew tolerance
        raise ReplayError("VP from future")
    if now - iat > max_age:
        raise ReplayError("VP too old")

    return True
```

---

## Request Signing (JAR)

### Why Sign Requests

| Risk | Mitigation |
|------|------------|
| Request tampering | Signature integrity |
| Verifier impersonation | Verifier authentication |
| Request age attacks | `iat`/`exp` claims |

### Signed Request Structure

```
JWT Request:
{
  header: {
    "alg": "ES256",
    "typ": "oauth-authz-req+jwt",
    "x5c": ["cert_chain..."]  // For x509_san_dns
  },
  payload: {
    "iss": "https://verifier.example.com",
    "aud": "https://self-issued.me/v2",
    "iat": 1683000000,
    "exp": 1683000300,  // 5 minute window
    "client_id": "https://verifier.example.com",
    "nonce": "n-0S6_WzA2Mj",
    "presentation_definition": { ... }
  }
}
```

### Wallet Validation

```python
def validate_signed_request(request_jwt, trust_anchors):
    # Decode JWT
    header, payload, signature = decode_jwt(request_jwt)

    # Check request freshness
    now = time.time()
    if payload['iat'] > now + 60:
        raise InvalidRequest("Request from future")
    if payload['exp'] < now:
        raise InvalidRequest("Request expired")

    # Validate based on client_id_scheme
    if payload['client_id_scheme'] == 'x509_san_dns':
        # Extract and validate certificate chain
        certs = decode_x5c(header['x5c'])
        validate_cert_chain(certs, trust_anchors)

        # Check SAN matches client_id
        san = extract_san_dns(certs[0])
        if san != payload['client_id']:
            raise InvalidRequest("SAN mismatch")

        # Verify signature
        verify_signature(request_jwt, certs[0].public_key)

    return payload
```

---

## Session Timeout Handling

### Timeout Values

| Phase | Recommended Timeout |
|-------|---------------------|
| Request creation → scan | 5 minutes |
| Scan → user consent | 5 minutes |
| Consent → response | 2 minutes |
| Total session | 10 minutes |

### Graceful Expiration

```python
class Session:
    def __init__(self, total_timeout=600):
        self.created_at = time.time()
        self.timeout = total_timeout
        self.state = 'pending'

    def is_expired(self):
        return time.time() - self.created_at > self.timeout

    def check_or_expire(self):
        if self.is_expired():
            self.state = 'expired'
            self.cleanup_sensitive_data()
            raise SessionExpired()
```

### Client Notification

For cross-device flows, notify user of expiration:

```javascript
// Polling with timeout awareness
function pollForResult(sessionId, maxWait) {
    const startTime = Date.now();

    const poll = async () => {
        if (Date.now() - startTime > maxWait) {
            showTimeout("Session expired. Please try again.");
            return;
        }

        const result = await fetch(`/session/${sessionId}/status`);

        if (result.status === 'completed') {
            handleSuccess(result);
        } else if (result.status === 'expired') {
            showTimeout("Session expired.");
        } else {
            setTimeout(poll, 2000);
        }
    };

    poll();
}
```
