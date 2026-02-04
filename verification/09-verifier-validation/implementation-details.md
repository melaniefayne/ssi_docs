# Verifier Validation — Implementation Details

This document provides guidance on implementing verifier-side validation. Note that the EUDI and Procivis repositories analyzed are primarily **wallet** implementations. Verifier validation is typically implemented on backend services.

---

## Validation Architecture

### Typical Verifier Backend

```
┌─────────────────────────────────────────────────────────────────┐
│                        Verifier Backend                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │ Request Handler │  │ VP Validator    │  │ Business Logic  │ │
│  │ (POST /callback)│──│ (signature,     │──│ (application    │ │
│  │                 │  │  trust, status) │  │  rules)         │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘ │
│           │                   │                    │            │
│           │                   ▼                    │            │
│           │         ┌─────────────────┐            │            │
│           │         │ Trust Registry  │            │            │
│           │         │ (issuer list,   │            │            │
│           │         │  revocation)    │            │            │
│           │         └─────────────────┘            │            │
│           │                   │                    │            │
│           ▼                   ▼                    ▼            │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │                    Response to Client                       ││
│  └─────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

---

## EUDI Reference Implementation

While the analyzed EUDI wallet is holder-focused, the EUDI ecosystem includes verifier libraries:

### Android Verifier SDK

**Library:** `eu.europa.ec.eudi:eudi-lib-android-wallet-core`

The SDK provides verification utilities when acting as verifier in proximity scenarios:

```kotlin
// Proximity verification (when wallet acts as verifier)
// Note: Primary use case is wallet-to-wallet or terminal verification

val verificationResult = verifyDeviceResponse(
    deviceResponse = receivedResponse,
    sessionTranscript = transcript,
    trustedCertificates = trustedReaderCerts
)
```

### iOS Verifier Support

**Library:** EudiWalletKit

```swift
// Verification utilities in proximity mode
let verificationResult = try verifyMdocResponse(
    response: deviceResponse,
    trustedCertificates: readerCertificates
)
```

---

## Procivis Verifier Implementation

Procivis ONE includes verifier functionality through the core library:

### Proof Verification

**Location:** The verification happens in the backend core, not the mobile wallet.

The wallet side shows verification results:

```typescript
// Wallet receives verification result
const { data: proof } = useProofDetail(proofId);

// Proof states indicate verification result
switch (proof.state) {
    case ProofStateBindingEnum.ACCEPTED:
        // Verification succeeded
        break;
    case ProofStateBindingEnum.REJECTED:
        // Verification failed
        break;
}
```

### Core Library Verification

The `@procivis/one-core` library handles:

1. VP Token parsing
2. Signature verification
3. Presentation submission validation
4. Trust framework checks

---

## Affinidi Verification Service

Affinidi provides verification through the Iota Framework:

### Verification Flow

```
Verifier App ──request──► Affinidi Iota
                              │
                              ▼
                    Iota validates VP:
                    - Signature check
                    - Issuer trust
                    - Revocation status
                    - Schema validation
                              │
                              ▼
Verifier App ◄──result─── Affinidi Iota
```

### Response Handling

```typescript
// Affinidi SDK callback with validation result
onVerificationComplete: (result) => {
    if (result.valid) {
        // Access verified claims
        const claims = result.verifiedCredentials[0].claims;
    } else {
        // Handle validation failure
        console.error(result.error);
    }
}
```

---

## Generic Validation Implementation

### JWT VP Validation (Pseudocode)

```typescript
async function validateJwtVp(
    vpToken: string,
    expectedNonce: string,
    verifierIdentifier: string,
    trustFramework: TrustFramework
): Promise<ValidationResult> {

    // 1. Decode JWT
    const decoded = decodeJwt(vpToken);
    if (!decoded) {
        return { valid: false, error: "invalid_jwt_structure" };
    }

    // 2. Validate nonce
    if (decoded.payload.nonce !== expectedNonce) {
        return { valid: false, error: "nonce_mismatch" };
    }

    // 3. Validate audience
    if (decoded.payload.aud !== verifierIdentifier) {
        return { valid: false, error: "audience_mismatch" };
    }

    // 4. Resolve holder key
    const holderKey = await resolveKey(decoded.payload.iss);
    if (!holderKey) {
        return { valid: false, error: "holder_key_resolution_failed" };
    }

    // 5. Verify VP signature
    if (!verifySignature(vpToken, holderKey)) {
        return { valid: false, error: "vp_signature_invalid" };
    }

    // 6. Validate embedded VCs
    const vcs = decoded.payload.vp?.verifiableCredential || [];
    for (const vc of vcs) {
        const vcResult = await validateVc(vc, trustFramework);
        if (!vcResult.valid) {
            return vcResult;
        }
    }

    return {
        valid: true,
        holder: decoded.payload.iss,
        credentials: vcs.map(extractClaims)
    };
}
```

### SD-JWT Validation

```typescript
async function validateSdJwt(
    sdJwt: string,
    kbJwt: string,
    expectedNonce: string,
    trustFramework: TrustFramework
): Promise<ValidationResult> {

    // 1. Parse SD-JWT
    const parts = sdJwt.split('~');
    const issuerJwt = parts[0];
    const disclosures = parts.slice(1, -1);

    // 2. Verify issuer JWT
    const issuerDecoded = decodeJwt(issuerJwt);
    const issuerKey = await resolveKey(issuerDecoded.payload.iss);

    if (!verifySignature(issuerJwt, issuerKey)) {
        return { valid: false, error: "issuer_signature_invalid" };
    }

    // 3. Validate issuer trust
    if (!trustFramework.isTrusted(issuerDecoded.payload.iss)) {
        return { valid: false, error: "issuer_not_trusted" };
    }

    // 4. Process disclosures
    const claims = {};
    for (const disclosure of disclosures) {
        const decoded = base64UrlDecode(disclosure);
        const [salt, name, value] = JSON.parse(decoded);

        // Verify disclosure hash is in _sd array
        const hash = sha256(disclosure);
        if (!issuerDecoded.payload._sd?.includes(hash)) {
            return { valid: false, error: "disclosure_hash_mismatch" };
        }

        claims[name] = value;
    }

    // 5. Validate key binding JWT
    const kbDecoded = decodeJwt(kbJwt);

    if (kbDecoded.payload.nonce !== expectedNonce) {
        return { valid: false, error: "kb_nonce_mismatch" };
    }

    // Get holder key from cnf claim
    const holderKey = issuerDecoded.payload.cnf?.jwk;
    if (!verifySignature(kbJwt, holderKey)) {
        return { valid: false, error: "kb_signature_invalid" };
    }

    // 6. Check revocation
    if (issuerDecoded.payload.credentialStatus) {
        const status = await checkRevocation(issuerDecoded.payload.credentialStatus);
        if (status === "revoked") {
            return { valid: false, error: "credential_revoked" };
        }
    }

    return {
        valid: true,
        issuer: issuerDecoded.payload.iss,
        claims: claims
    };
}
```

### Revocation Check

```typescript
async function checkRevocation(
    statusEntry: StatusListEntry
): Promise<"valid" | "revoked" | "suspended" | "unknown"> {

    // Fetch status list credential
    const statusListVc = await fetch(statusEntry.statusListCredential);
    const decoded = await validateVc(statusListVc);

    if (!decoded.valid) {
        return "unknown";
    }

    // Decode status list
    const encodedList = decoded.claims.credentialSubject.encodedList;
    const decompressed = gunzip(base64Decode(encodedList));

    // Extract bit at index
    const index = parseInt(statusEntry.statusListIndex);
    const byteIndex = Math.floor(index / 8);
    const bitIndex = index % 8;
    const bit = (decompressed[byteIndex] >> (7 - bitIndex)) & 1;

    return bit === 1 ? statusEntry.statusPurpose : "valid";
}
```

---

## Trust Framework Integration

### Trust List Example

```typescript
interface TrustFramework {
    isTrusted(issuerId: string): Promise<boolean>;
    getTrustLevel(issuerId: string): Promise<TrustLevel>;
    getCapabilities(issuerId: string): Promise<string[]>;
}

class EudiTrustFramework implements TrustFramework {
    private trustedList: TrustedIssuer[];

    async isTrusted(issuerId: string): Promise<boolean> {
        const issuer = this.trustedList.find(i => i.id === issuerId);
        if (!issuer) return false;
        if (issuer.validUntil < new Date()) return false;
        return true;
    }
}
```

### Certificate Chain Validation

```typescript
async function validateCertificateChain(
    chain: X509Certificate[],
    trustedRoots: X509Certificate[]
): Promise<boolean> {

    // Validate each certificate in chain
    for (let i = 0; i < chain.length - 1; i++) {
        const cert = chain[i];
        const issuer = chain[i + 1];

        // Verify signature
        if (!cert.verify(issuer.publicKey)) {
            return false;
        }

        // Check validity period
        if (cert.notBefore > new Date() || cert.notAfter < new Date()) {
            return false;
        }
    }

    // Verify root is trusted
    const root = chain[chain.length - 1];
    return trustedRoots.some(tr => tr.equals(root));
}
```

---

## Error Handling

### Validation Error Codes

| Code | Meaning | Action |
|------|---------|--------|
| `invalid_structure` | Cannot parse VP | Reject, log |
| `nonce_mismatch` | Replay or wrong request | Reject |
| `signature_invalid` | Cryptographic failure | Reject |
| `issuer_not_trusted` | Unknown issuer | Reject or escalate |
| `credential_revoked` | Credential invalidated | Reject |
| `credential_expired` | Past expiration | Reject |
| `constraint_not_met` | Missing required claim | Reject |

### Logging for Audit

```typescript
function logValidationResult(result: ValidationResult): void {
    const logEntry = {
        timestamp: new Date().toISOString(),
        correlationId: result.correlationId,
        valid: result.valid,
        error: result.error,
        issuer: result.issuer,
        credentialTypes: result.credentialTypes,
        // Do NOT log actual claim values for privacy
    };

    auditLog.write(logEntry);
}
```

---

## Key Libraries

| Platform | Library | Purpose |
|----------|---------|---------|
| Node.js | `jose` | JWT/JWS operations |
| Node.js | `@digitalbazaar/vc` | W3C VC validation |
| Node.js | `@transmute/vc.js` | Multiple VC formats |
| Java | `nimbus-jose-jwt` | JWT/JWS operations |
| Kotlin | `eudi-lib-*` | EUDI-specific validation |
| Swift | `CryptoKit` | Signature verification |
