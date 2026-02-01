# Credential Offer -- Implementation Details

This document details how each wallet implementation handles credential offers: URI parsing, offer resolution, validation logic, transport mechanisms, and TxCode handling. Code paths reference actual source files from the analyzed repositories.

---

## EUDI Android

### Offer Resolution

The credential offer flow in the EUDI Android wallet begins in `DocumentOfferViewModel`, which is responsible for resolving the offer URI and managing the UI state for the offer screen.

**Entry point:** `DocumentOfferViewModel.resolveDocumentOffer(offerUri: String)`

This method delegates to the interactor layer:

```
DocumentOfferViewModel
  -> resolveDocumentOffer(offerUri)
    -> DocumentOfferInteractor.resolveDocumentOffer(offerUri)
      -> WalletCoreDocumentsController.resolveDocumentOffer(offerUri)
        -> OpenId4VciManager.resolveCredentialOffer(offerUri)
```

The SDK-level `OpenId4VciManager.resolveCredentialOffer()` handles the OID4VCI protocol mechanics: parsing the URI, extracting the `credential_offer` or `credential_offer_uri` parameter, fetching the offer object if necessary, and resolving the issuer's metadata.

The core offer resolution, PID validation, and TxCode validation logic:

```kotlin
// File: issuance-feature/src/main/java/eu/europa/ec/issuancefeature/interactor/DocumentOfferInteractor.kt

override fun resolveDocumentOffer(offerUri: String): Flow<ResolveDocumentOfferInteractorPartialState> =
    flow {
        val userLocale = resourceProvider.getLocale()
        walletCoreDocumentsController.resolveDocumentOffer(
            offerUri = offerUri
        ).map { response ->
            when (response) {
                is ResolveDocumentOfferPartialState.Failure -> {
                    ResolveDocumentOfferInteractorPartialState.Failure(errorMessage = response.errorMessage)
                }

                is ResolveDocumentOfferPartialState.Success -> {

                    credentialOffers[offerUri] = response.offer

                    val offerHasNoDocuments = response.offer.offeredDocuments.isEmpty()
                    if (offerHasNoDocuments) {
                        ResolveDocumentOfferInteractorPartialState.NoDocument(
                            issuerName = response.offer.getIssuerName(userLocale),
                            issuerLogo = response.offer.getIssuerLogo(userLocale),
                        )
                    } else {

                        val codeMinLength = 4
                        val codeMaxLength = 6

                        safeLet(
                            response.offer.txCodeSpec?.inputMode,
                            response.offer.txCodeSpec?.length
                        ) { inputMode, length ->

                            if ((length !in codeMinLength..codeMaxLength) || inputMode == TxCodeInputMode.TEXT) {
                                return@map ResolveDocumentOfferInteractorPartialState.Failure(
                                    errorMessage = resourceProvider.getString(
                                        R.string.issuance_document_offer_error_invalid_txcode_format,
                                        codeMinLength,
                                        codeMaxLength
                                    )
                                )
                            }
                        }

                        val hasMainPid =
                            walletCoreDocumentsController.getMainPidDocument() != null

                        val hasPidInOffer =
                            response.offer.offeredDocuments.any { offeredDocument ->
                                val id = offeredDocument.documentIdentifier
                                id == DocumentIdentifier.MdocPid || id == DocumentIdentifier.SdJwtPid
                            }

                        if (hasMainPid || hasPidInOffer) {
                            ResolveDocumentOfferInteractorPartialState.Success(
                                documents = response.offer.offeredDocuments.map { offeredDocument ->
                                    DocumentOfferUi(
                                        title = offeredDocument.getName(userLocale).orEmpty(),
                                    )
                                },
                                issuerName = response.offer.getIssuerName(userLocale),
                                issuerLogo = response.offer.getIssuerLogo(userLocale),
                                txCodeLength = response.offer.txCodeSpec?.length
                            )
                        } else {
                            ResolveDocumentOfferInteractorPartialState.Failure(
                                errorMessage = resourceProvider.getString(
                                    R.string.issuance_document_offer_error_missing_pid_text
                                )
                            )
                        }
                    }
                }
            }
        }.collect {
            emit(it)
        }
    }
```

The resolved offer is returned as a domain object containing:
- The list of offered credential types (mapped to `OfferedDocumentDomain`)
- The issuer's display metadata (name, logo)
- Whether a TxCode is required
- TxCode parameters (input mode, length, description)

### PID Requirement Validation

`DocumentOfferInteractor` enforces a regulatory constraint specific to the EU Digital Identity framework: **the wallet must hold a valid PID (Person Identification Data) credential, or the offer must include a PID credential, before any other credential can be issued.**

The validation logic checks:

1. Does the wallet already contain a valid PID document?
2. If not, does the current offer include a PID credential among its `credential_configuration_ids`?

If neither condition is met, the offer is rejected with an error indicating that a PID is required. This is an EU-specific regulatory requirement (eIDAS 2.0) and has no equivalent in the other implementations studied.

### TxCode Validation

When the resolved offer includes a `tx_code` requirement, the EUDI Android wallet:

1. Displays a PIN entry screen to the user.
2. Constrains input to **numeric characters only** (digits 0-9). The implementation does not support the `text` input mode defined by the specification.
3. Enforces a length constraint of **4 to 6 digits**. Input outside this range is rejected.
4. Submits the entered code alongside the pre-authorized code during the token exchange.

This is a deliberate restriction: the EUDI reference wallet targets regulated identity credentials where numeric PINs are the expected transaction code format.

### URI Scheme Handling

The Android wallet registers for the `openid-credential-offer` URI scheme via intent filters. When the system routes an `openid-credential-offer://` URI to the app:

1. The intent is received by the main activity.
2. The `credential_offer` query parameter is extracted from the URI.
3. Navigation routes the user to the `DocumentOfferScreen` with the extracted offer data.

The wallet also supports receiving offers via QR code scanning through its built-in scanner, which produces the same URI string for processing.

### Scoped Document Discovery (Wallet-Initiated)

For wallet-initiated issuance, the EUDI Android wallet provides a discovery mechanism via `WalletCoreDocumentsController.getScopedDocuments()`:

1. Iterates over all configured issuers in the wallet's configuration.
2. For each issuer, calls `OpenId4VciManager.getIssuerMetadata()` to fetch the `/.well-known/openid-credential-issuer` metadata.
3. Maps each entry in `credential_configurations_supported` to a `ScopedDocumentDomain` object containing the configuration ID, display name, format, and whether it is a PID credential.
4. Returns the aggregated list grouped by issuer.

This allows the user to browse available credentials across all configured issuers without receiving an explicit offer.

---

## EUDI iOS

### Offer Resolution

The iOS wallet follows the same architectural pattern as Android, with Swift equivalents of each component.

**Entry point:** `DocumentOfferInteractor.processOfferRequest(offerUri: String)`

The resolution chain:

```
DocumentOfferInteractor
  -> processOfferRequest(offerUri)
    -> WalletKitController.resolveOfferUrlDocTypes(offerUri)
      -> EudiWallet.resolveOfferUrlDocTypes(offerUri)
        -> OpenId4VCIService.resolveCredentialOffer(offerUri)
```

The resolved offer returns the same domain information as the Android implementation: offered document types, issuer display metadata, and TxCode requirements.

The core offer resolution, PID validation, and TxCode validation logic in Swift:

```swift
// File: Modules/feature-issuance/Sources/Interactor/DocumentOfferInteractor.swift

func processOfferRequest(with uri: String) async -> OfferRequestPartialState {
    do {

      let codeMinLength = 4
      let codeMaxLength = 6

      let offer = try await walletController.resolveOfferUrlDocTypes(offerUri: uri)
      let hasPidStored = await !walletController.fetchIssuedDocuments(with: [.mDocPid, .sdJwtPid]).isEmpty

      if let spec = offer.txCodeSpec,
         let codeLength = spec.length,
         !(codeMinLength...codeMaxLength).contains(codeLength) || spec.inputMode == .text {
        return .failure(WalletCoreError.transactionCodeFormat(["\(codeMinLength)", "\(codeMaxLength)"]))
      }

      let hasPidInOffer = offer.docModels.first(
        where: { offer in
          let identifier = DocumentTypeIdentifier(
            rawValue: offer.docType.ifNilOrEmpty {
              offer.vct.ifNilOrEmpty {
                offer.credentialConfigurationIdentifier
              }
            }
          )
          return identifier == .mDocPid || identifier == .sdJwtPid
        }
      ) != nil

      if !hasPidStored && !hasPidInOffer {
        return .failure(WalletCoreError.missingPid)
      }

      return .success(offer.transformToDocumentOfferUi())
    } catch {
      return .failure(error)
    }
  }
```

### URI Scheme Registration

The iOS wallet registers for two URI schemes:

- **`openid-credential-offer`** -- The standard OID4VCI scheme.
- **`haip-vci`** -- The High Assurance Interoperability Profile scheme.

Both schemes are handled identically once received. The URL is routed through the app's URL handling mechanism to the offer resolution flow.

The wallet also supports **Universal Links** for offer delivery. When configured with an associated domain, HTTPS URLs matching the issuer's offer endpoint pattern are intercepted by the app instead of opening in Safari.

### TxCode Input

The iOS wallet presents TxCode input via `OfferCodeView`, which implements:

- A text field configured for numeric input (matching the Android restriction to digits only).
- A **250ms debounce** on input changes. When the user types, the wallet waits 250 milliseconds after the last keystroke before validating the input. This prevents validation from firing on every keystroke and provides a smoother user experience.
- Length enforcement matching the `length` parameter from the TxCode specification.
- Auto-submission when the required number of digits has been entered (if `length` is specified).

### PID Requirement

The iOS wallet enforces the same PID requirement as Android. `DocumentOfferInteractor` validates that either:
- A valid PID exists in the wallet's document store, or
- The current offer includes a PID credential type.

The logic and error handling mirror the Android implementation.

### Scoped Document Discovery

`WalletKitController.getScopedDocuments()` provides the same wallet-initiated discovery as Android:

1. Queries all configured issuers.
2. Fetches metadata from each issuer's `/.well-known/openid-credential-issuer` endpoint.
3. Maps credential configurations to displayable document types.
4. Groups results by issuer for presentation to the user.

---

## Procivis ONE

### Invitation Processing

Procivis ONE uses a unified "invitation" concept that encompasses credential offers, presentation requests, and other protocol interactions. The entry point for credential offers is the invitation processing screen.

**Entry point:** `invitation-process-screen.tsx`

When the user scans a QR code or receives a deep link, the URL is processed through the invitation handler:

```
invitation-process-screen.tsx
  -> useInvitationHandler()
    -> handleInvitation(url)
      -> getInvitationUrlTransports(url)
      -> oneCore.handleInvitation(url, transport)
```

### Transport Detection

`getInvitationUrlTransports(url)` inspects the URL to determine which transports are available for the interaction. The detection logic evaluates the URL scheme and structure:

- **MQTT:** URLs containing MQTT broker identifiers or using MQTT-specific schemes route to the MQTT transport.
- **BLE:** URLs containing BLE-specific parameters or device identifiers route to the Bluetooth Low Energy transport.
- **HTTP:** Standard HTTPS URLs route to the HTTP transport (the default for OID4VCI).

When multiple transports are available, Procivis ONE applies a preference order: **MQTT > BLE > HTTP**. This preference reflects the system's design for enterprise and IoT scenarios where message-broker-mediated or proximity-based communication may be preferred over direct HTTP.

### HTTP Redirect Handling

For HTTP-based offers, the invitation handler follows redirects to resolve the final offer URL. This is handled via `RNBlobUtil` (React Native Blob Util), which performs the HTTP request and follows redirect chains:

```typescript
// Follows redirects to resolve the final URL
const response = await RNBlobUtil.fetch('GET', url, {}, '');
const finalUrl = response.info().redirects?.pop() || url;
```

This is important because credential offer URLs are often short URLs or redirect-based URLs that resolve to the full `openid-credential-offer://` URI.

The actual redirect resolution and invitation handling logic:

```typescript
// File: app/screens/credential/invitation-process-screen.tsx

useEffect(() => {
    if (
      !canHandleInvitation ||
      !availableTransport ||
      redirectState === 'redirecting'
    ) {
      return;
    }

    if (!redirectState && isValidHttpUrl(invitationUrl)) {
      setRedirectState('redirecting');
      RNBlobUtil.config({ followRedirect: false })
        .fetch('GET', invitationUrl)
        .then((response) => {
          setRedirectState('done');
          if (response.respInfo.redirects.length === 0) {
            setRedirectState('done');
            return;
          }
          const headers =
            typeof response.respInfo.headers === 'object'
              ? (response.respInfo.headers as Record<any, any>)
              : undefined;
          const redirectUrl =
            headers && typeof headers === 'object' && 'Location' in headers
              ? (headers['Location'] as string)
              : undefined;
          if (!redirectUrl) {
            setRedirectState('done');
            return;
          }
          const newInvitationUrl =
            parseUniversalLink(redirectUrl) ?? redirectUrl;
          if (
            newInvitationUrl &&
            !isValidHttpUrl(newInvitationUrl) &&
            newInvitationUrl !== invitationUrl
          ) {
            setInvitationSupportedTransports(
              getInvitationUrlTransports(
                newInvitationUrl,
                config.customOpenIdUrlScheme,
              ),
            );
            setInvitationUrl(newInvitationUrl);
          }
          setRedirectState('done');
        })
        .catch(() => {
          setRedirectState('done');
        });
      return;
    }

    const transport = availableTransport.includes(Transport.MQTT)
      ? Transport.MQTT
      : availableTransport[0];
    if (!transport) {
      return;
    }

    handleInvitation({
      redirectUri: config.requestCredentialRedirectUri,
      transport: [transport],
      url: invitationUrl,
    })
      .then((result) => {
        setInvitationResult(result);
      })
      .catch((err: unknown) => {
        setState(LoaderViewState.Warning);
        if (
          err instanceof OneError &&
          err.cause?.includes('BLE adapter not enabled')
        ) {
          setAdapterEnabled(false);
        } else {
          setError(err);
          if (
            err &&
            isInvalidInvitationUrlError(err) &&
            !isValidHttpUrl(invitationUrl)
          ) {
            setState(LoaderViewState.Error);
          }
        }
      });
  }, [
    availableTransport,
    canHandleInvitation,
    handleInvitation,
    invitationUrl,
    managementNavigation,
    redirectState,
  ]);
```

### TxCode Handling

Procivis ONE supports both TxCode input modes defined by the OID4VCI specification:

- **`NUMERIC`** -- Presents a numeric keypad. Input is constrained to digits.
- **`TEXT`** -- Presents a full text keyboard. Any characters are accepted.

The input mode is determined from the offer's `tx_code.input_mode` field and used to configure the input component accordingly. Length enforcement is applied when the `length` parameter is present.

This is a notable difference from the EUDI wallets, which only support numeric input.

### Deep Link Handling

Deep links are processed via `deep-link.ts`, which contains the `parseUniversalLink()` function:

```
deep-link.ts
  -> parseUniversalLink(url)
    -> Extract scheme, path, and query parameters
    -> Route to appropriate handler based on URL structure
```

The parser handles multiple URL formats:
- `openid-credential-offer://` URIs
- Universal link URLs with path-based routing
- Custom scheme URLs for the Procivis ONE app

### Core SDK Delegation

Once the URL is parsed and the transport is selected, the actual OID4VCI protocol handling is delegated to the One Core SDK via `useInvitationHandler()`. The React Native layer does not implement OID4VCI protocol logic directly -- it handles URL parsing, transport selection, and UI presentation, while the native core handles offer resolution, metadata fetching, and the credential exchange.

---

## Affinidi

### Cloud-Based Offer Creation

Affinidi's architecture differs fundamentally from the mobile wallet implementations. Credential offers are created on the server side by the **Credential Issuance Service**, not by the wallet.

The issuer (a service built using the Affinidi TDK) creates a credential offer by calling the Credential Issuance Service API with:

- **User claims:** The actual data to be included in the credential (e.g., name, date of birth, address).
- **Claim mode:** How the holder will claim the credential (see below).
- **Credential configuration:** References a pre-configured credential type, including its schema and signing wallet.
- **Revocability:** Whether the issued credential can be revoked after issuance.

### Claim Modes

Affinidi defines two claim modes that determine how the credential offer is bound to a specific holder:

#### TX_CODE Mode

The service generates a transaction code and includes it in the offer. The issuer shares this code with the intended holder through an out-of-band channel (email, SMS, in-person). When the holder's Affinidi Vault opens the claim link, it prompts for the transaction code.

This is functionally equivalent to the OID4VCI Pre-Authorized Code Grant with a TxCode requirement, but managed entirely by the cloud service rather than the wallet.

#### FIXED_HOLDER Mode

The offer is bound to a specific holder identified by their DID (Decentralized Identifier). Only the holder whose DID matches can claim the credential. No transaction code is needed because the holder's identity is cryptographically verified during the claim process.

This mode has no direct equivalent in the OID4VCI specification's grant types -- it is an Affinidi-specific mechanism that provides stronger binding than TxCode at the cost of requiring the issuer to know the holder's DID in advance.

### Claim Link Delivery

After creating the offer, the Credential Issuance Service returns a **claim link** -- a URL that the issuer delivers to the holder. The delivery mechanism is the issuer's responsibility:

- Email with the claim link
- SMS with the claim link
- QR code encoding the claim link
- In-app notification

When the holder opens the claim link, the Affinidi Vault (the holder's wallet application) resolves the offer URI, presents the credential details, and initiates the claim process.

### Vault-Side Resolution

The Affinidi Vault handles offer resolution:

1. Opens the claim link URL.
2. Resolves the credential offer from the Credential Issuance Service.
3. Displays the credential type and issuer information to the user.
4. If TX_CODE mode: prompts for the transaction code.
5. If FIXED_HOLDER mode: verifies the holder's DID against the offer binding.
6. Proceeds with the OID4VCI token exchange and credential request.

The Vault's internal resolution logic is not exposed in the public TDK -- it is a closed implementation detail of the Affinidi Vault application.

---

## Summary: Resolution Flow by Implementation

| Step | EUDI Android | EUDI iOS | Procivis ONE | Affinidi |
|------|-------------|----------|--------------|----------|
| **Receive offer** | Intent filter / QR scan | URL scheme handler / QR scan | QR scan / deep link / BLE / MQTT | Claim link (email/SMS/QR) |
| **Parse URI** | Extract `credential_offer` param | Extract `credential_offer` param | `parseUniversalLink()` | Vault resolves claim link |
| **Resolve offer** | `OpenId4VciManager.resolveCredentialOffer()` | `OpenId4VCIService.resolveCredentialOffer()` | One Core SDK `handleInvitation()` | Vault internal resolution |
| **Validate** | PID check + TxCode validation | PID check + TxCode validation | Transport selection + TxCode | Claim mode verification |
| **Present to user** | `DocumentOfferScreen` | `DocumentOfferView` | `invitation-process-screen.tsx` | Vault offer screen |
| **TxCode input** | Numeric only, 4-6 digits | Numeric only, 250ms debounce | Numeric or text | TX_CODE mode via Vault |
