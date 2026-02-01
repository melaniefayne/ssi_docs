# Authorization & Token Exchange -- Implementation Details

This document describes how each wallet implementation handles the authorization and token exchange phase of OID4VCI. The implementations differ significantly in how much of the authorization flow is exposed to the application layer versus encapsulated within the SDK or core engine.

---

## 1. EUDI Android

### SDK Encapsulation

On Android, the entire authorization and token exchange flow is handled internally by `OpenId4VciManager` in the `eudi-lib-android-wallet-core` SDK. The application layer does not construct authorization requests, manage PKCE parameters, or call the token endpoint directly.

The wallet application interacts with the SDK through `WalletCoreDocumentsController`, which exposes high-level operations:

```kotlin
// File: core-logic/src/main/java/eu/europa/ec/corelogic/controller/WalletCoreDocumentsController.kt

override fun issueDocumentsByOffer(
    offer: Offer,
    txCode: String?,
): Flow<IssueDocumentsPartialState> =
    callbackFlow {

        val issuerId = offer
            .credentialOffer
            .credentialIssuerIdentifier
            .toString()

        val manager = openId4VciManagers[issuerId]
            ?: openId4VciManagers.values.firstOrNull()

        require(manager != null) { documentErrorMessage }

        manager.issueDocumentByOffer(
            offer = offer,
            onIssueEvent = issuanceCallback(),
            txCode = txCode,
        )
        awaitClose()
    }
```

The SDK resolves the grant type from the credential offer, selects the appropriate flow (authorization code or pre-authorized code), and executes it internally.

### Configuration

Each issuer's VCI configuration is provided through `WalletCoreConfig`, which includes per-issuer settings:

```kotlin
// Simplified from WalletCoreConfig
data class VciConfig(
    val issuerUrl: String,
    val clientId: String,
    val redirectUri: String,
    // Additional per-issuer configuration
)
```

The `vciConfig` is loaded from the wallet's configuration module and passed to the SDK when initializing the `EudiWallet` instance. The SDK uses this configuration to construct authorization requests with the correct `client_id` and `redirect_uri`.

### Authorization Code Flow

When the credential offer specifies an authorization code grant:

1. `OpenId4VciManager` constructs the authorization request, including PKCE `code_challenge` and `code_challenge_method`.
2. If PAR is supported by the issuer, the SDK sends a pushed authorization request to the PAR endpoint and receives a `request_uri`.
3. The SDK triggers a browser-based authentication flow (via Android Custom Tabs or the system browser).
4. After the user authenticates, the browser redirects back to the wallet via the configured `redirect_uri`.
5. `OpenId4VciManager` intercepts the redirect, extracts the authorization code, and exchanges it at the token endpoint.
6. The token response (including `access_token` and `c_nonce`) is stored internally for the subsequent credential request.

### Wallet Attestation Provider

The wallet provides cryptographic attestation through `WalletCoreAttestationProvider`:

```kotlin
interface WalletCoreAttestationProvider {
    suspend fun getWalletAttestation(keyInfo: KeyInfo): String
    suspend fun getKeyAttestation(keys: List<Key>, nonce: String): String
}
```

During the authorization flow, the attestation provider may be invoked to include wallet attestation in the token request. The `nonce` parameter in `getKeyAttestation` is passed from the issuer to bind the attestation to the session.

### DPoP

DPoP support is handled internally by the SDK. When the issuer's authorization server metadata indicates DPoP support, the SDK generates a DPoP key pair, constructs DPoP proof JWTs, and includes them in token and credential requests. The application layer does not participate in DPoP token management.

---

## 2. EUDI iOS

### SDK Encapsulation

The iOS implementation mirrors the Android architecture. `WalletKitController` delegates authorization to the `eudi-lib-ios-wallet-core` SDK, which manages the flow internally.

### Configuration

`WalletKitConfig` explicitly enables security features:

```swift
// File: Modules/logic-core/Sources/Config/WalletKitConfig.swift

.init(
    credentialIssuerURL: "https://issuer.eudiw.dev",
    clientId: "wallet-dev",
    keyAttestationsConfig: .init(walletAttestationsProvider: walletKitAttestationProvider),
    authFlowRedirectionURI: URL(string: "eu.europa.ec.euidi://authorization")!,
    usePAR: true,
    useDpopIfSupported: true,
    cacheIssuerMetadata: true
)
```

Key configuration values:

| Parameter | Value | Description |
|-----------|-------|-------------|
| `usePAR` | `true` | Pushed Authorization Requests are enabled. The SDK sends authorization parameters to the PAR endpoint before redirecting the user. |
| `useDpopIfSupported` | `true` | If the issuer supports DPoP, the SDK generates DPoP proofs and binds access tokens to the wallet's key pair. |
| `authFlowRedirectionURI` | `"eu.europa.ec.euidi://authorization"` | The custom URI scheme used for OAuth redirect callbacks. Registered in the app's Info.plist. |

### Authorization Code Flow with PKCE

The authorization code flow on iOS follows the same logical sequence as Android:

1. The SDK constructs the authorization request with PKCE parameters.
2. If `usePAR` is `true`, the SDK pushes the authorization request to the PAR endpoint.
3. The SDK opens the authorization URL in the system browser (ASWebAuthenticationSession or SFSafariViewController).
4. After authentication, the browser redirects to `eu.europa.ec.euidi://authorization` with the authorization code.
5. The SDK extracts the code and exchanges it for an access token.

### DPoP Token Generation

DPoP proof JWTs are generated using the `JOSESwift` library:

```swift
// Conceptual flow inside the SDK
let dpopKey = try P256.Signing.PrivateKey()
let header = JWSHeader(algorithm: .ES256)
header.type = "dpop+jwt"
header.jwk = dpopKey.publicKey.jwk

let payload = DPoPPayload(
    jti: UUID().uuidString,
    htm: "POST",
    htu: tokenEndpointURL,
    iat: Date()
)

let dpopProof = try JWS(header: header, payload: payload, signer: dpopKey)
```

When `useDpopIfSupported` is `true` and the issuer advertises DPoP support in its metadata, the SDK includes the DPoP proof in the `DPoP` header of the token request and uses the `DPoP` authorization scheme for subsequent requests.

### Pre-Authorized Code Flow

The pre-authorized code flow is used when the credential offer includes a `pre-authorized_code` grant. The SDK bypasses the browser-based flow entirely and sends the pre-authorized code directly to the token endpoint. If a TxCode is required, the application collects it from the user and passes it to the SDK.

---

## 3. Procivis ONE

### Architecture

Procivis ONE exposes the authorization flow at a higher level than the EUDI implementations. The native core engine (`@procivis/react-native-one-core`) handles protocol mechanics, but the React Native application layer manages browser-based authentication and redirect handling.

### Invitation Handling

When the wallet processes a credential offer, the `handleInvitation()` function returns a result that indicates which flow is required:

```typescript
// File: app/screens/credential/invitation-process-screen.tsx

useEffect(() => {
    if (!invitationResult) {
      return;
    }
    if (invitationResult.type_ === 'AUTHORIZATION_CODE_FLOW') {
      openBrowser(invitationResult.authorizationCodeFlowUrl);
    } else if (invitationResult.type_ === 'PROOF_REQUEST') {
      managementNavigation.replace('ShareCredential', {
        params: { request: invitationResult },
        screen: 'ProofRequest',
      });
    } else {
      if (isLoadingWU) {
        return;
      }
      if (invitationResult.txCode) {
        managementNavigation.replace('IssueCredential', {
          params: { invitationResult: invitationResult },
          screen: 'CredentialConfirmationCode',
        });
      } else {
        const needsRSESetup =
          invitationResult.keyStorageSecurityLevels?.includes(
            KeyStorageSecurityBindingEnum.HIGH,
          ) && !isRSESetup;
        managementNavigation.replace('IssueCredential', {
          params: { invitationResult: invitationResult },
          screen: needsRSESetup ? 'RSEInfo' : 'CredentialOffer',
        });
      }
    }
  }, [invitationResult, isRSESetup, isLoadingWU, managementNavigation, rootNavigation]);
```

The `AUTHORIZATION_CODE_FLOW` result type signals that the wallet must open a browser for user authentication.

### Browser-Based Authentication

Procivis ONE uses `@swan-io/react-native-browser` to open an in-app browser for the authorization code flow:

```typescript
import { openBrowser } from '@swan-io/react-native-browser';

const openBrowserForAuth = async (authUrl: string) => {
  const result = await openBrowser(authUrl, {
    dismissButtonStyle: 'cancel',
    // Browser opens within the app context
  });
};
```

The browser presents the issuer's login page. After the user authenticates, the issuer redirects to the wallet's redirect URI.

### Redirect Handling

The redirect URI is configured per build flavor (e.g., production, staging, development):

```typescript
// Configuration per flavor
const config = {
  requestCredentialRedirectUri: 'procivis-one://credential-redirect',
  // Other flavor-specific settings
};
```

When the browser redirects to the configured URI, the app intercepts the redirect and passes the authorization code back to the core engine. The `useContinueIssuance()` hook handles the redirect callback:

```typescript
// File: app/screens/credential/invitation-process-screen.tsx

const { mutateAsync: continueIssuance } = useContinueIssuance();

const handleContinueIssuance = useCallback(
    async (url: string) => {
      if (
        config.requestCredentialRedirectUri &&
        url.startsWith(config.requestCredentialRedirectUri)
      ) {
        closeBrowser();
        const result = await continueIssuance(url);
        managementNavigation.replace('IssueCredential', {
          params: {
            invitationResult: {
              type_: 'CREDENTIAL_ISSUANCE',
              ...result,
            },
          },
          screen: 'CredentialOffer',
        });
      }
    },
    [continueIssuance, managementNavigation],
  );
```

### Pre-Authorized Code Flow

When the credential offer uses the pre-authorized code grant, the flow is transparent to the application layer. The core engine handles the token exchange internally, and the application proceeds directly to the credential acceptance screen.

### Supported Grant Types

Procivis ONE supports both grant types through its multi-protocol architecture:

| Grant Type | Application Involvement | Core Engine Role |
|------------|------------------------|-----------------|
| Pre-Authorized Code | Minimal -- TxCode collection only | Full protocol handling |
| Authorization Code | Browser management, redirect handling | Token exchange, credential request |

---

## 4. Affinidi

### Pre-Authorized Code Flow (Primary)

Affinidi's credential issuance architecture is built around the Pre-Authorized Code flow. The credential offer is generated by the Affinidi Credential Issuance Service and includes a pre-authorization code:

```json
{
  "credential_issuer": "https://oid4vci.affinidi.com/issuance",
  "credential_configuration_ids": ["VerifiedEmail"],
  "grants": {
    "urn:ietf:params:oauth:grant-type:pre-authorized_code": {
      "pre-authorized_code": "code_value_here",
      "tx_code": {
        "input_mode": "numeric",
        "length": 6
      }
    }
  }
}
```

### Token Exchange

The Affinidi Vault (the wallet application) exchanges the pre-authorized code and the user-entered transaction code for an access token:

```
Affinidi Vault                    Affinidi Issuance Service
     |                                    |
     |  POST /token                       |
     |  grant_type=pre-authorized_code    |
     |  pre-authorized_code=...           |
     |  tx_code=493536                    |
     |  --------------------------------> |
     |                                    |
     |  { access_token, c_nonce }         |
     |  <-------------------------------- |
     |                                    |
```

### FIXED_HOLDER Mode

Affinidi supports a `FIXED_HOLDER` claim mode where the credential is pre-assigned to a specific holder identified by DID. In this mode, the transaction code may not be required because the holder's identity is validated through DID-based authentication:

```
Configuration: claimMode = FIXED_HOLDER
                holderDid = "did:key:z6Mkh..."

Flow:
  Vault presents DID --> Issuance Service validates DID matches holder
                         --> Access token issued without tx_code
```

This mode is used when the issuer knows the holder's DID in advance (e.g., the holder has previously registered with the issuer). DID validation replaces the transaction code as the holder verification mechanism.

### Authorization Code Flow

Affinidi's primary documentation and public APIs focus on the Pre-Authorized Code flow. The Authorization Code flow is not prominently featured in public-facing materials, though the underlying OID4VCI implementation may support it for specific deployment configurations.

### Cloud-Managed Token Exchange

Unlike the mobile-first EUDI and Procivis implementations, Affinidi's token exchange involves the cloud-based Credential Issuance Service. The Vault communicates with the service through the Affinidi TDK (Trust Development Kit), which handles HTTP transport and token management:

```
Affinidi Vault App
       |
       v
Affinidi TDK Client Library
       |
       v
Affinidi Credential Issuance Service (cloud)
       |
       +--> Token endpoint
       +--> Credential endpoint
       +--> Configuration management
```

---

## Summary: Authorization Flow by Implementation

| Implementation | Primary Grant Type | Auth Code Flow | Browser Integration | Redirect URI | Token Exchange |
|---------------|-------------------|----------------|--------------------|--------------|----|
| EUDI Android | Both | SDK-managed | Android Custom Tabs | Configured in `VciConfig` | SDK-internal |
| EUDI iOS | Both | SDK-managed | ASWebAuthenticationSession | `eu.europa.ec.euidi://authorization` | SDK-internal |
| Procivis ONE | Both | App manages browser | `@swan-io/react-native-browser` | Per build flavor | Core engine |
| Affinidi | Pre-Authorized Code | Not primary | N/A | N/A | TDK client library |

---

**Next**: [Comparative Analysis](./comparative-analysis.md) -- Strengths, weaknesses, and trade-offs across implementations.
