# Authorization & Token Exchange -- Comparative Analysis

This document compares how the four wallet implementations handle authorization and token exchange. The comparison covers protocol feature support, architectural choices, and the trade-offs each approach entails.

---

## Feature Comparison Matrix

| Feature | EUDI Android | EUDI iOS | Procivis ONE | Affinidi |
|---------|-------------|----------|--------------|----------|
| **Authorization Code Grant** | Yes (SDK-managed) | Yes (SDK-managed) | Yes (app-managed browser) | Not primary |
| **Pre-Authorized Code Grant** | Yes (SDK-managed) | Yes (SDK-managed) | Yes (core engine) | Yes (primary) |
| **PAR (RFC 9126)** | Yes (SDK-internal) | Yes (`usePAR: true`) | Dependent on issuer | Not documented |
| **PKCE (RFC 7636)** | Yes (implicit in auth code flow) | Yes (implicit in auth code flow) | Yes (core engine) | N/A (no auth code flow) |
| **DPoP (RFC 9449)** | Yes (SDK-internal) | Yes (`useDpopIfSupported: true`) | Dependent on issuer | Not documented |
| **Browser integration** | Android Custom Tabs | ASWebAuthenticationSession | `@swan-io/react-native-browser` | N/A |
| **Redirect URI** | Configured in `VciConfig` | `eu.europa.ec.euidi://authorization` | Per build flavor | N/A |
| **Token exchange visibility** | SDK-internal (opaque) | SDK-internal (opaque) | Core engine (opaque) | TDK client library |

---

## PAR Support

### EUDI (Android & iOS)

Both EUDI wallets support PAR. On iOS, this is explicitly configured via `usePAR: true` in `WalletKitConfig`. On Android, the SDK detects PAR support from the authorization server metadata and uses it automatically.

PAR is a significant security advantage for the authorization code flow. It keeps sensitive authorization parameters out of the browser's address bar and allows the authorization server to authenticate the client before user interaction begins.

### Procivis ONE

PAR support depends on the issuer's configuration and the core engine's protocol handling. The application layer does not control PAR behavior directly. If the issuer's authorization server metadata advertises a PAR endpoint, the core engine uses it.

### Affinidi

PAR is not documented in Affinidi's public materials. Since Affinidi primarily uses the Pre-Authorized Code flow, PAR (which applies only to the Authorization Code flow) is not relevant for the primary use case.

---

## DPoP Support

### EUDI iOS

DPoP is explicitly enabled: `useDpopIfSupported: true`. The SDK checks whether the issuer supports DPoP (via the `dpop_signing_alg_values_supported` field in the authorization server metadata) and, if so, generates DPoP proofs using JOSESwift. This is a conditional feature -- if the issuer does not support DPoP, the SDK falls back to Bearer tokens.

### EUDI Android

DPoP is supported and managed internally by the SDK. The application layer does not need to enable or configure DPoP. Like iOS, the SDK checks issuer support and uses DPoP when available.

### Procivis ONE

DPoP support is handled by the core engine. If the issuer requires DPoP, the engine generates proofs. The application layer does not participate in DPoP management.

### Affinidi

DPoP is not documented in Affinidi's public-facing materials. The Pre-Authorized Code flow does not strictly require DPoP (though it can benefit from it), and Bearer tokens appear to be the default.

---

## Auth Code vs. Pre-Auth Code

### Grant Type Preference by Implementation

| Implementation | Primary Grant Type | Reason |
|---------------|-------------------|--------|
| EUDI (both) | Both (issuer-determined) | Reference implementations support the full OID4VCI specification. The grant type is determined by the credential offer. |
| Procivis ONE | Both (issuer-determined) | Multi-protocol architecture supports both. The `handleInvitation()` return type indicates which flow is needed. |
| Affinidi | Pre-Authorized Code | Cloud-based architecture assumes the issuer has already authenticated the user. The issuance service generates offers with pre-authorized codes. |

### Implications

**Pre-Authorized Code advantages:**
- Simpler wallet implementation (no browser management, no redirect handling).
- Better user experience (no login screen in the wallet).
- Lower latency (one fewer round trip).

**Authorization Code advantages:**
- Supports unknown users (user authenticates during the flow).
- Leverages existing identity providers (federated login).
- Stronger security guarantees (user explicitly authenticates with the issuer).
- Required for issuers that cannot pre-authenticate users.

Affinidi's exclusive focus on Pre-Authorized Code simplifies their architecture but limits the deployment scenarios. EUDI and Procivis ONE support both, maximizing flexibility at the cost of implementation complexity.

---

## Browser-Based Authentication

### EUDI Android

Uses Android Custom Tabs, which provide a browser experience within the app context. Custom Tabs share the system browser's cookie jar, enabling SSO with previously authenticated sessions.

### EUDI iOS

Uses `ASWebAuthenticationSession` (or `SFSafariViewController`), the platform-standard mechanism for OAuth browser flows. This provides similar benefits to Custom Tabs on Android -- shared cookies, system browser security.

### Procivis ONE

Uses `@swan-io/react-native-browser`, a React Native bridge to platform-native browser components. This is architecturally necessary because the React Native layer must manage the browser lifecycle (opening, monitoring redirects, closing).

### Affinidi

Does not require browser-based authentication because the Pre-Authorized Code flow does not involve user login in the wallet.

### Comparison

| Aspect | EUDI Android | EUDI iOS | Procivis ONE | Affinidi |
|--------|-------------|----------|--------------|----------|
| Browser component | Custom Tabs | ASWebAuthenticationSession | swan-io bridge | N/A |
| SSO support | Yes (shared cookies) | Yes (shared cookies) | Yes (platform bridge) | N/A |
| Redirect interception | SDK-managed | SDK-managed | App-managed (`useContinueIssuance`) | N/A |
| User-visible browser | Yes (in-app tab) | Yes (modal sheet) | Yes (in-app browser) | N/A |

---

## Redirect URI Approaches

| Implementation | Redirect URI | Configuration |
|---------------|-------------|---------------|
| EUDI Android | Per issuer, in `VciConfig` | Build-time configuration |
| EUDI iOS | `eu.europa.ec.euidi://authorization` | Hardcoded custom scheme |
| Procivis ONE | `requestCredentialRedirectUri` per flavor | Build flavor configuration |
| Affinidi | N/A | N/A |

The EUDI iOS approach uses a fixed custom URI scheme, simplifying configuration but limiting flexibility. The EUDI Android and Procivis ONE approaches support per-issuer or per-flavor redirect URIs, enabling different configurations for different deployment environments (production, staging, development).

---

## Strengths and Weaknesses

### EUDI (Android & iOS)

**Strengths:**
- Full OID4VCI compliance with PAR, PKCE, and DPoP.
- SDK encapsulation protects the application from protocol complexity.
- Security features (PAR, DPoP) are enabled by default, not opt-in.
- Both grant types are supported without application-level branching.

**Weaknesses:**
- Deep SDK encapsulation makes debugging authorization failures difficult. The application has limited visibility into the protocol exchange.
- The application cannot customize authorization request parameters beyond what the SDK exposes.
- Custom authentication flows (e.g., federated identity with non-standard providers) require SDK modifications.

### Procivis ONE

**Strengths:**
- The `AUTHORIZATION_CODE_FLOW` result type from `handleInvitation()` gives the application explicit control over browser-based authentication.
- Build flavor system allows different redirect URIs per deployment environment.
- `useContinueIssuance()` provides a clean hook for redirect handling.
- Support for both grant types enables broad issuer compatibility.

**Weaknesses:**
- Application-layer browser management introduces complexity that the EUDI SDK hides.
- Redirect handling requires careful coordination between the React Native layer and the native core engine.
- The application must handle edge cases (browser dismissed by user, redirect timeout, deep link conflicts) that SDK-managed flows handle internally.

### Affinidi

**Strengths:**
- Pre-Authorized Code focus eliminates browser-based authentication complexity entirely.
- `FIXED_HOLDER` mode provides a clean alternative to TxCode for known users.
- Cloud-managed token exchange simplifies the wallet implementation.

**Weaknesses:**
- Lack of Authorization Code flow support limits deployment to scenarios where the issuer pre-authenticates users.
- No documented PAR or DPoP support reduces the security profile for environments that require sender-constrained tokens.
- Dependency on the Affinidi cloud service introduces a single point of failure for the token exchange.

---

## Decision Framework

| If your scenario requires... | Consider... |
|------------------------------|------------|
| Maximum security (PAR + DPoP + PKCE) | EUDI (Android or iOS) |
| Support for unknown users (auth code flow) | EUDI or Procivis ONE |
| Simplest wallet implementation | Affinidi (pre-auth only) |
| Cross-platform React Native | Procivis ONE |
| Custom browser/redirect handling | Procivis ONE (explicit app control) |
| Cloud-managed issuance | Affinidi |
| Pre-authenticated users only | Affinidi (FIXED_HOLDER mode) |
| EU regulatory compliance | EUDI (designed for eIDAS 2.0) |

---

**Previous**: [Implementation Details](./implementation-details.md)
**Section index**: [README](./README.md)
