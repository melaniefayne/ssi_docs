# Trust Establishment and Attack Prevention

This page analyzes the security architecture of SSI systems: how trust is established, how specific attacks are prevented, and where residual risks remain.

---

## Trust Establishment

Trust in an SSI system is not a single relationship but a chain of trust assertions. Each link in the chain must be cryptographically verifiable.

### Trust Chain

```
Certificate Authority
    |
    +-- Issuer Certificate (X.509 or trust registry entry)
          |
          +-- Issuer Signs Credential (with issuer's private key)
                |
                +-- Credential Binds to Holder Key (cnf / DeviceKeyInfo)
                      |
                      +-- Holder Key is in Hardware (TEE / Secure Enclave)
                            |
                            +-- Hardware Produces Key Attestation
                                  |
                                  +-- Wallet Attestation (wallet app integrity)
```

### Trust Components

#### Issuer Trust

The verifier must trust that the issuer is a legitimate authority for the credential type. This is established through:

| Mechanism | How It Works |
|-----------|-------------|
| X.509 certificate chain | The issuer's signing key is certified by a trusted CA. The verifier validates the chain up to a trusted root. Used in mDoc (IssuerAuth contains x5chain). |
| Trust registry | A curated list of trusted issuers, maintained by a governance authority. The verifier checks that the issuer's DID or identifier is in the registry. |
| DID resolution | The verifier resolves the issuer's DID to obtain the public key. Trust in the DID method itself provides the foundation. |
| Issuer metadata | The issuer publishes metadata at a well-known endpoint. The verifier fetches and validates the metadata, including signing keys. |

#### Wallet Provider Trust

The issuer must trust that the wallet is a legitimate, unmodified application capable of securely managing credentials. This is established through:

| Mechanism | How It Works |
|-----------|-------------|
| Wallet attestation | The wallet provider (app developer) issues an attestation JWT asserting the wallet's identity and integrity. The issuer verifies this attestation during OID4VCI. |
| App attestation (platform) | Android (Play Integrity) and iOS (App Attest) provide platform-level attestation that the app is genuine, unmodified, and running on a real device. |

#### Key Trust

The issuer must trust that the key presented during credential issuance is genuinely hardware-backed and controlled by the holder's device. This is established through:

| Mechanism | How It Works |
|-----------|-------------|
| Key attestation | The device's security hardware produces an attestation certificate chain proving the key was generated inside the TEE/Secure Enclave and has specific properties (non-extractable, biometric-protected). |
| Key binding (cnf) | The credential's `cnf` (confirmation) claim or mDoc's `DeviceKeyInfo` cryptographically binds the credential to a specific public key. |

#### Holder Trust

The verifier must trust that the entity presenting the credential is the legitimate holder. This is established through:

| Mechanism | How It Works |
|-----------|-------------|
| Proof of possession | The holder signs a challenge (nonce) with the private key bound to the credential. Since the key is in hardware, only the device that holds the key can produce the signature. |
| Biometric gating | The key operation requires biometric authentication (Face ID, fingerprint), adding a local authentication layer. |

---

## Attack Analysis

### 1. Replay Attacks

**Threat:** An attacker captures a valid credential presentation and replays it to a verifier at a later time, impersonating the holder.

**How c_nonce Prevents It:**

The c_nonce (credential nonce) is a fresh random value provided by the issuer or verifier for each interaction. The wallet must include this nonce in its proof JWT, and the verifier checks that the nonce matches the one it issued.

```
Legitimate flow:
  Verifier --> Wallet: "Present credential. Nonce: abc123"
  Wallet --> Verifier: Proof JWT { nonce: "abc123", signed by holder key }
  Verifier: nonce matches, signature valid --> ACCEPT

Replay attack:
  Attacker captures the Proof JWT from above
  Later, attacker --> Verifier: Proof JWT { nonce: "abc123", signed by holder key }
  Verifier: nonce "abc123" was already used / is expired --> REJECT
```

**Defense layers:**
- c_nonce is single-use: the verifier never accepts the same nonce twice
- c_nonce has a short lifetime: even if not tracked, it expires
- The proof JWT includes `iat` (issued at), enabling time-based rejection

**Nonce binding in JWT proofs:**
```json
{
  "iss": "wallet-client-id",
  "aud": "https://issuer.example.com",
  "iat": 1700001000,
  "nonce": "issuer-provided-c_nonce"
}
```

The nonce is part of the signed payload. An attacker cannot modify it without invalidating the signature.

---

### 2. Token Theft

**Threat:** An attacker intercepts an OAuth 2.0 access token and uses it to request credential issuance from a different device.

**How DPoP Prevents It:**

DPoP (Demonstration of Proof-of-Possession) binds the access token to a specific key pair. The token can only be used by the entity that controls the corresponding private key.

```
Legitimate flow:
  Wallet: Generate DPoP key pair (private in hardware, public in DPoP proof)
  Wallet --> Issuer: Token request + DPoP proof (signed by DPoP key)
  Issuer: Binds access token to DPoP key thumbprint (jkt)
  Wallet --> Issuer: Credential request + access token + new DPoP proof (with ath)
  Issuer: Verifies DPoP key matches token's jkt --> ACCEPT

Token theft:
  Attacker: Intercepts access token
  Attacker: Does NOT have wallet's DPoP private key
  Attacker --> Issuer: Credential request + stolen token + DPoP proof (signed by attacker's key)
  Issuer: DPoP key thumbprint != token's jkt --> REJECT
```

**Sender-constrained tokens:**

The access token's `cnf` claim contains the JWK thumbprint of the wallet's DPoP key:

```json
{
  "cnf": {
    "jkt": "SHA-256-thumbprint-of-wallet-DPoP-key"
  }
}
```

Every request using this token must include a DPoP proof signed by the key whose thumbprint matches `jkt`. Since the wallet's DPoP key is generated in hardware (TEE/Secure Enclave), an attacker cannot forge the proof.

---

### 3. Substitution Attacks

**Threat:** An attacker obtains a valid credential (e.g., through a compromised issuer or social engineering) and presents it as their own to a verifier.

**How Key Binding Prevents It:**

Every credential is cryptographically bound to a specific key pair controlled by the legitimate holder. The credential includes the holder's public key (via `cnf` in SD-JWT or `DeviceKeyInfo` in mDoc). During presentation, the holder must produce a proof signed by the corresponding private key.

```
Credential binding:
  SD-JWT contains: cnf.jwk = { holder's public key }
  mDoc contains: DeviceKeyInfo = { holder's public key }

Legitimate presentation:
  Holder: Signs KB-JWT / DeviceAuth with private key
  Verifier: Checks signature against cnf.jwk / DeviceKeyInfo --> MATCH --> ACCEPT

Substitution attack:
  Attacker: Has a copy of the credential
  Attacker: Does NOT have holder's private key (it is in holder's hardware)
  Attacker: Signs KB-JWT / DeviceAuth with attacker's key
  Verifier: Checks signature against cnf.jwk / DeviceKeyInfo --> MISMATCH --> REJECT
```

**Critical dependency:** This defense relies entirely on the private key being non-extractable. If the key can be copied (e.g., software-only storage), substitution attacks become possible through key theft.

---

### 4. Impersonation

**Threat:** An attacker creates a fake wallet or uses a modified wallet to obtain credentials from an issuer, then presents them as a legitimate holder.

**How Wallet Attestation Prevents It:**

Wallet attestation allows the issuer to verify that the credential request comes from a genuine, unmodified wallet application.

```
Wallet attestation flow:
  1. Wallet Provider signs attestation JWT for the wallet instance
  2. Platform (Android/iOS) provides app integrity attestation
  3. Wallet sends both attestations to issuer during OID4VCI
  4. Issuer verifies:
     - Wallet attestation signature is valid
     - Wallet provider is in trusted list
     - App integrity attestation confirms genuine app
     - Key attestation confirms hardware-backed keys
```

**How Key Attestation Prevents It:**

Key attestation proves that the key used for credential binding was generated in genuine security hardware, not in a software emulator or modified app.

```
Key attestation chain:
  Root: Device manufacturer's root certificate
    +-- Intermediate: Security hardware attestation key
          +-- Leaf: Attestation for the wallet's credential key
                +-- Properties: non-extractable, hardware-backed, TEE/Strongbox

Issuer verifies:
  - Certificate chain is valid up to a trusted root
  - Key properties meet minimum security requirements
  - Key is genuinely hardware-backed (not software-emulated)
```

---

### 5. Man-in-the-Middle (MITM)

**Threat:** An attacker positions themselves between the wallet and the issuer (or verifier), intercepting and potentially modifying communications.

**Defense Layers:**

| Layer | Mechanism | What It Protects |
|-------|-----------|-----------------|
| Transport | TLS 1.3 | Encrypts all communication, authenticates the server via certificate |
| Token binding | DPoP | Even if TLS is compromised, the token is bound to the wallet's key |
| Request integrity | PAR | Authorization request is sent directly to server, not through browser |
| Code binding | PKCE | Authorization code is bound to the original client |
| Key attestation | Hardware attestation chains | Proves keys are in genuine hardware, not intercepted |
| Nonce freshness | c_nonce | Prevents replay of intercepted messages |

**MITM scenario analysis:**

```
Attacker intercepts TLS (compromised CA or network):
  - DPoP prevents token use (attacker lacks private key)
  - PKCE prevents code exchange (attacker lacks code verifier)
  - Key attestation detects emulated hardware

Attacker intercepts BLE/NFC (proximity presentation):
  - DeviceAuth requires holder's private key (in hardware)
  - Session encryption (mDoc) prevents eavesdropping
  - Reader authentication (optional) verifies the verifier
```

---

## Attack/Defense Matrix

| Attack Vector | Description | c_nonce | DPoP | PKCE | PAR | Key Binding | Wallet Attestation | Key Attestation | TLS |
|--------------|-------------|---------|------|------|-----|-------------|-------------------|-----------------|-----|
| **Replay** | Reuse captured presentation | DEFENDS | -- | -- | -- | -- | -- | -- | -- |
| **Token theft** | Steal access token | -- | DEFENDS | -- | -- | -- | -- | -- | -- |
| **Code interception** | Steal authorization code | -- | -- | DEFENDS | -- | -- | -- | -- | -- |
| **Request tampering** | Modify auth request | -- | -- | -- | DEFENDS | -- | -- | -- | -- |
| **Credential transfer** | Use someone else's credential | -- | -- | -- | -- | DEFENDS | -- | -- | -- |
| **Fake wallet** | Modified or counterfeit wallet | -- | -- | -- | -- | -- | DEFENDS | -- | -- |
| **Software key emulation** | Fake hardware-backed keys | -- | -- | -- | -- | -- | -- | DEFENDS | -- |
| **Network eavesdropping** | Read communications | -- | -- | -- | -- | -- | -- | -- | DEFENDS |
| **Session hijacking** | Take over active session | DEFENDS | DEFENDS | DEFENDS | -- | -- | -- | -- | DEFENDS |
| **Impersonation (holder)** | Pretend to be holder | -- | -- | -- | -- | DEFENDS | -- | DEFENDS | -- |
| **Impersonation (issuer)** | Pretend to be issuer | -- | -- | -- | -- | -- | -- | -- | DEFENDS |

Legend:
- **DEFENDS**: This mechanism is a primary defense against this attack
- **--**: This mechanism does not directly address this attack

---

## Residual Risks

No system is perfectly secure. The following risks remain even with all defenses in place:

### Device Compromise (Root/Jailbreak)

If the device operating system is compromised (rooted Android, jailbroken iOS), the hardware isolation guarantees weaken. While TEE/Secure Enclave keys remain in hardware, the application layer that interacts with them may be manipulated.

**Mitigation:** App integrity attestation (Play Integrity, App Attest) can detect some compromised environments, but sophisticated attackers may bypass these checks.

### Side-Channel Attacks

Hardware security processors may be vulnerable to side-channel attacks (power analysis, electromagnetic emissions, timing attacks). These require physical access and specialized equipment but are not theoretical.

**Mitigation:** Strongbox (discrete secure element) and Secure Enclave have countermeasures against known side-channel attacks. Software countermeasures (constant-time operations) are implemented in cryptographic libraries.

### Social Engineering

An attacker may trick a legitimate holder into presenting credentials to a fraudulent verifier, or trick an issuer into issuing credentials to the wrong person. Cryptography cannot prevent human error in trust decisions.

**Mitigation:** Verifier authentication (display verifier identity in wallet UI), user education, and credential policies (one-time use limits exposure).

### Quantum Computing (Future)

Current elliptic curve algorithms (ES256, Ed25519, secp256k1) are vulnerable to quantum attacks via Shor's algorithm. A sufficiently powerful quantum computer could derive private keys from public keys.

**Mitigation:** Procivis supports CRYSTALS-DILITHIUM 3 for post-quantum signatures. Hybrid classical+PQC approaches are recommended during the transition period. Credentials issued today with classical algorithms may need re-issuance once quantum computers become practical.

### Cloud Key Management

For implementations that use cloud-based key storage (Affinidi, Procivis RSE), the cloud provider becomes a single point of trust. A compromise of the cloud infrastructure could expose all managed keys.

**Mitigation:** HSM-backed cloud key storage, strict access controls, audit logging, and geographic/jurisdictional controls on key storage locations.

---

## Trust Model Comparison

| Trust Property | EUDI | Procivis (Local) | Procivis (RSE) | Affinidi |
|---------------|------|-------------------|----------------|----------|
| Key non-extractability | Hardware-guaranteed | Hardware-guaranteed | HSM-guaranteed | Cloud provider SLA |
| Wallet attestation | Platform + wallet provider | Platform + wallet provider | Wallet provider | Cloud authentication |
| Key attestation | Hardware attestation chain | Hardware attestation chain | HSM attestation | N/A |
| Issuer trust | Trust registry + X.509 | DID resolution + trust config | DID resolution + trust config | DID resolution |
| Offline presentation security | Full (device-bound keys) | Full (device-bound keys) | Degraded (needs network for signing) | Not supported |
| Post-quantum readiness | No | Yes (DILITHIUM) | Yes (DILITHIUM) | No |
| Regulatory compliance (eIDAS 2.0) | Designed for it | Supported | Supported | Not primary target |
