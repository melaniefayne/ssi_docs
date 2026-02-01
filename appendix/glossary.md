# Glossary

An alphabetical glossary of all terms, acronyms, and concepts used throughout the SSI documentation.

---

### c_nonce

**Credential Nonce.** A random value provided by the issuer in the OID4VCI token response or credential response. The wallet must include the c_nonce in its proof of possession JWT when requesting a credential. The c_nonce prevents replay attacks by ensuring that each credential request proof is fresh and bound to a specific issuance session. The issuer may rotate the c_nonce with each response.

### CBOR

**Concise Binary Object Representation.** A binary data serialization format defined in RFC 8949. CBOR is designed to be compact, fast to parse, and schema-flexible. It is used as the encoding format for mDoc credentials (ISO 18013-5) and as the foundation for COSE signing. CBOR serves a similar role to JSON but in binary form, making it more suitable for constrained environments and offline transport (BLE, NFC).

### COSE

**CBOR Object Signing and Encryption.** A set of cryptographic operations (signing, encryption, MAC) applied to CBOR-encoded data, defined in RFC 9052. COSE is the CBOR equivalent of JOSE (JSON Object Signing and Encryption). In SSI, COSE_Sign1 is used to sign the Mobile Security Object (MSO) in mDoc credentials and to sign device authentication proofs. COSE uses integer algorithm identifiers (e.g., -7 for ES256) rather than string identifiers.

### Credential Offer

A JSON object provided by the issuer (typically via deep link or QR code) that initiates the OID4VCI credential issuance flow. The credential offer contains the issuer's identifier, the credential configuration IDs (specifying which credentials can be issued), and either a pre-authorized code (for pre-authorized flow) or an authorization endpoint reference (for authorization code flow). The credential offer may also include a transaction code requirement.

### DID

**Decentralized Identifier.** A globally unique identifier that does not require a centralized registration authority, defined in the W3C DID Core specification. A DID resolves to a DID document containing public keys and service endpoints. In SSI, DIDs identify issuers, holders, and verifiers. Common DID methods include did:web (DNS-based), did:key (self-contained public key), and did:ethr (Ethereum-based).

### DPoP

**Demonstration of Proof-of-Possession.** A mechanism defined in RFC 9449 that binds an OAuth 2.0 access token to a specific key pair. The client generates a DPoP key pair and includes a DPoP proof JWT with each request. The authorization server binds the access token to the key's thumbprint. When the token is used, the resource server verifies that the DPoP proof is signed by the same key. This prevents token theft: a stolen access token is useless without the corresponding private key.

### ECDSA

**Elliptic Curve Digital Signature Algorithm.** A digital signature algorithm that uses elliptic curve cryptography. ECDSA provides the same security as RSA with much smaller key sizes. In SSI, ECDSA is used with the P-256 curve (as ES256) for JWT signing, DPoP proofs, and credential signatures. ECDSA with the secp256k1 curve is used by Affinidi for JSON-LD Data Integrity proofs.

### EdDSA

**Edwards-curve Digital Signature Algorithm.** A digital signature algorithm that uses Edwards curves, specifically Ed25519 (Curve25519 in Edwards form). EdDSA provides deterministic signatures (no random nonce required), which eliminates a class of implementation vulnerabilities present in ECDSA. EdDSA is used by Procivis for credential signing. It is faster than ECDSA but not supported by iOS Secure Enclave.

### eIDAS

**electronic IDentification, Authentication and trust Services.** An EU regulation that establishes a legal framework for electronic identification and trust services. eIDAS 2.0 (the revised regulation) introduces the European Digital Identity Wallet, mandating that EU member states offer citizens a digital wallet for identity credentials. eIDAS 2.0 specifies requirements for Wallet Secure Cryptographic Devices (WSCD), qualified electronic signatures, and cross-border interoperability.

### HAIP

**High Assurance Interoperability Profile.** A profile that specifies a constrained set of options from OID4VCI, OID4VP, SD-JWT, and mDoc to ensure interoperability at a high assurance level. HAIP mandates specific algorithms (ES256), proof types (JWT), and security mechanisms (DPoP, PAR, PKCE). It is designed to ensure that wallets and issuers from different vendors can interoperate without negotiation.

### Holder

The entity that possesses and controls a verifiable credential. In SSI, the holder is typically an individual who stores credentials in a wallet application on their device. The holder's private key is bound to the credential, and only the holder can present the credential to a verifier by producing a proof of possession. The holder may also be an organization or a device.

### Issuer

The entity that creates and signs a verifiable credential. The issuer asserts that the claims in the credential are true for the subject. Examples include governments (issuing identity documents), universities (issuing diplomas), and employers (issuing employment attestations). The issuer's public key is published in metadata or a DID document so that verifiers can validate credential signatures.

### JWK

**JSON Web Key.** A JSON format for representing cryptographic keys, defined in RFC 7517. A JWK includes the key type (kty), algorithm-specific parameters (crv, x, y for EC keys), and optional metadata (kid, use). JWKs are used throughout OID4VCI: in DPoP proof headers, in credential proof JWTs, in issuer metadata (JWKS), and in the SD-JWT cnf claim for key binding.

### JWS

**JSON Web Signature.** A standard for signing arbitrary data using JSON-based data structures, defined in RFC 7515. JWS Compact Serialization has the format: BASE64URL(header).BASE64URL(payload).BASE64URL(signature). JWS is the foundation for JWT and is used in DPoP proofs, credential proofs, SD-JWT issuer tokens, and key binding JWTs.

### JWT

**JSON Web Token.** A compact, URL-safe token format for transmitting claims between parties, defined in RFC 7519. A JWT is a JWS with a JSON payload containing claims (iss, sub, exp, iat, etc.). In SSI, JWTs are used for access tokens, DPoP proofs, credential proofs of possession, wallet attestation, key binding proofs, and as a credential format (JWT VC).

### Key Attestation

A cryptographic proof that a key was generated inside genuine security hardware (TEE, Strongbox, Secure Enclave) and has specific properties (non-extractable, hardware-backed, biometric-protected). Key attestation is provided as a certificate chain rooted at the device manufacturer's certificate authority. Issuers use key attestation to verify that the key binding a credential is in secure hardware.

### Key Binding

The cryptographic association between a verifiable credential and a specific key pair controlled by the holder. Key binding ensures that only the holder who controls the private key can present the credential. In SD-JWT, key binding is established via the cnf (confirmation) claim. In mDoc, key binding is established via DeviceKeyInfo in the MSO. During presentation, the holder proves key binding by signing a challenge.

### mDoc

**Mobile Document.** A credential format defined in ISO 18013-5 for mobile driving licenses (mDL) and extended to other document types. mDoc uses CBOR encoding, COSE signing, and a Mobile Security Object (MSO) for integrity. Claims are organized into namespaces and individually hashed in the MSO, enabling selective disclosure. mDoc is designed for offline presentation via BLE and NFC.

### MSO

**Mobile Security Object.** The integrity and authenticity structure within an mDoc credential. The MSO contains SHA-256 hashes of each individual claim (organized by namespace), the holder's device key info, and validity information. The MSO is signed by the issuer using COSE_Sign1. During presentation, the verifier checks that presented claims hash to values in the signed MSO.

### OID4VCI

**OpenID for Verifiable Credential Issuance.** A protocol specification that extends OAuth 2.0 to support the issuance of verifiable credentials. OID4VCI defines how a wallet discovers an issuer's capabilities, authenticates, obtains an access token, and requests credential issuance. It supports pre-authorized and authorization code flows, batch and deferred issuance, DPoP token binding, and multiple credential formats.

### OID4VP

**OpenID for Verifiable Presentations.** A protocol specification that extends OpenID Connect to support the presentation of verifiable credentials. OID4VP defines how a verifier requests specific credentials from a wallet, how the wallet selects and presents credentials, and how the verifier validates the presentation. It supports multiple presentation formats and transport mechanisms.

### PAR

**Pushed Authorization Requests.** A mechanism defined in RFC 9126 that allows an OAuth 2.0 client to pre-register its authorization request directly with the authorization server via a back-channel POST request, rather than sending parameters through the browser redirect. PAR prevents request tampering, avoids URL length limits, and keeps authorization parameters out of browser logs.

### PID

**Person Identification Data.** In the EUDI context, PID is the core identity credential issued to citizens. It contains fundamental identity attributes such as family name, given name, birth date, and nationality. PID is the highest-assurance credential in the EUDI ecosystem and is typically issued with a OneTimeUse credential policy with multiple instances (e.g., 10) to maximize privacy through unlinkability.

### PKCE

**Proof Key for Code Exchange.** A mechanism defined in RFC 7636 that prevents authorization code interception attacks. The client generates a random code verifier, computes a code challenge (SHA-256 hash of the verifier), and sends the challenge with the authorization request. When exchanging the code for a token, the client sends the original verifier. The server checks that SHA-256(verifier) matches the stored challenge. An attacker who intercepts the code cannot exchange it without the verifier.

### Proof of Possession

A cryptographic proof that an entity controls a specific private key. In OID4VCI, the wallet provides a proof of possession (typically a signed JWT) when requesting a credential, demonstrating that it controls the key to which the credential will be bound. The proof includes the issuer's c_nonce to prevent replay. Proof of possession is distinct from presenting the credential -- it occurs during issuance.

### RSE

**Remote Secure Element.** A remote Hardware Security Module (HSM) that provides key generation, storage, and signing operations over a network connection. In Procivis, the UBIQU_RSE option delegates key operations to a remote HSM, enabling centralized key management for enterprise deployments. RSE provides HSM-grade security while allowing mobile wallet UX, at the cost of requiring network connectivity for every signing operation.

### SD-JWT

**Selective Disclosure JWT.** An extension to JWT that enables the holder to selectively disclose individual claims. The issuer creates a JWT with hashed claim references (in _sd arrays) and provides the actual claim values as separate base64url-encoded disclosures. During presentation, the holder includes only the disclosures for claims they wish to reveal. The verifier hashes each disclosure and checks that it matches an entry in the JWT's _sd array.

### Secure Enclave

Apple's hardware-isolated coprocessor for cryptographic key operations. The Secure Enclave has its own boot ROM, AES engine, and protected memory. Keys generated in the Secure Enclave never leave the hardware -- the application processor can request signing operations but cannot access key material. The Secure Enclave supports ECDSA with P-256 (ES256) and ECDH with P-256.

### Strongbox

A discrete secure element chip on Android devices, physically separate from the main system-on-chip (SoC). Strongbox provides a higher assurance level than TEE because it has independent power, clock, and memory, making it resistant to side-channel and fault-injection attacks. Strongbox is required on devices shipping with Android 9+ that include the hardware. It is accessed through the Android Keystore API.

### TEE

**Trusted Execution Environment.** An isolated execution environment on the main system-on-chip (SoC) of Android devices. The TEE has its own processor context, memory space, and storage, and communicates with the main OS through a narrow interface. Keys generated in the TEE cannot be extracted by the main OS. TEE provides a lower assurance level than Strongbox (which is a discrete chip) but is available on more devices.

### TxCode

**Transaction Code.** A short code (typically numeric, 4-8 digits) that the issuer provides to the holder out-of-band (e.g., via email, SMS, or displayed on screen) as a second factor during credential issuance. The wallet prompts the holder to enter the TxCode during the pre-authorized code flow. The TxCode is sent to the token endpoint alongside the pre-authorized code. It provides user authentication and prevents unauthorized credential claim.

### Verifiable Credential

A tamper-evident credential that has been cryptographically signed by its issuer, as defined in the W3C Verifiable Credentials Data Model. A verifiable credential contains claims about a subject, metadata about the credential (issuer, issuance date, expiry), and a cryptographic proof. The credential can be independently verified by any party that has access to the issuer's public key. Specific formats include mDoc, SD-JWT, JSON-LD VC, and JWT VC.

### Verifier

The entity that receives and validates a verifiable presentation from a holder. The verifier checks the cryptographic proofs (issuer signature, holder proof of possession), validates the credential's status (not expired, not revoked), and evaluates whether the presented claims satisfy the verification requirements. Examples include border control agencies, age verification services, and employers.

### VP Token

**Verifiable Presentation Token.** The token containing one or more verifiable credentials presented by the holder to a verifier during an OID4VP flow. The VP token includes the credential(s), the holder's proof of possession, and session-binding information (audience, nonce). The format of the VP token depends on the credential format: a vp_token containing an SD-JWT+KB-JWT, or a DeviceSigned mDoc, or a JSON-LD Verifiable Presentation.

### W3C VC Data Model

**W3C Verifiable Credentials Data Model.** A W3C Recommendation that defines a standard data model for expressing verifiable credentials and verifiable presentations. The data model specifies the JSON-LD structure including @context, type, issuer, credentialSubject, and proof. Version 1.1 (2022) is widely implemented; version 2.0 (2024) adds features including multiple subjects and evidence.

### Wallet Attestation

A cryptographic assertion from the wallet provider (the entity that developed and distributes the wallet application) that a specific wallet instance is genuine, unmodified, and meets security requirements. Wallet attestation is provided as a signed JWT that the wallet presents to the issuer during OID4VCI. The issuer verifies the attestation to ensure it is issuing credentials to a legitimate wallet, not a modified or counterfeit application.

### WSCD

**Wallet Secure Cryptographic Device.** A term from the eIDAS 2.0 regulation referring to the secure hardware component that protects the wallet's cryptographic keys. The WSCD may be a device-local secure element (TEE, Strongbox, Secure Enclave) or a remote secure element (HSM). eIDAS 2.0 requires that certain credentials (e.g., qualified electronic signatures) use keys stored in a certified WSCD.
