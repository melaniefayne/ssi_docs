# What Is SSI Credential Verification?

Credential verification is the process by which a Holder proves claims about themselves to a Verifier by presenting credentials that were previously issued by a trusted Issuer. In Self-Sovereign Identity (SSI), this process is designed so that the Holder controls what information is shared, with whom, and when — without requiring real-time involvement from the Issuer.

This document explains the conceptual model. No implementation details, no code, no protocol-level specifics. Those come in sections 01 through 12.

---

## The Trust Triangle Revisited

SSI verification operates within the same three-party trust model as issuance, but the data flow direction is different:

```
           +---------+
           | Issuer  |
           +----+----+
                │
        issued credential (past)
                │
                ▼
           +---------+        presents credential (now)       +-----------+
           | Holder  | ────────────────────────────────────►  | Verifier  |
           +---------+                                        +-----------+
                                                                    │
                                                           validates against
                                                                    │
                                                                    ▼
                                                            Trust Framework
                                                         (Issuer public keys,
                                                          revocation lists)
```

### Issuer (in verification context)

The Issuer is not an active participant in each verification transaction. However, the Issuer's role is foundational:

- The Issuer's public key (or DID) must be available for the Verifier to validate signatures
- The Issuer may publish revocation status lists that Verifiers check
- The Issuer's reputation and the trust framework determine whether a Verifier accepts credentials they issued

The Issuer does not learn when, where, or to whom the Holder presents credentials — this is a critical privacy property of SSI.

### Holder

The Holder possesses credentials in their wallet and decides when to present them. During verification, the Holder:

1. Receives a presentation request from a Verifier (directly or via QR code/deep link)
2. Reviews what information is being requested and why
3. Selects which credentials and claims to share
4. Constructs a cryptographically signed presentation proving they control the credentials
5. Transmits the presentation to the Verifier

The Holder has agency: they can refuse requests, share only partial information (selective disclosure), or choose which of multiple matching credentials to use.

### Verifier

The Verifier (also called Relying Party) needs to confirm claims about the Holder. The Verifier:

1. Constructs a presentation request specifying what credentials and claims are needed
2. Transmits the request to the Holder's wallet
3. Receives the Verifiable Presentation from the Holder
4. Validates the presentation:
   - Credential signatures are valid and from trusted Issuers
   - Credentials have not been revoked
   - The Holder proved possession of the credential binding keys
   - The presentation is fresh (not a replay)
5. Extracts the verified claims for business logic

---

## What Is a Verifiable Presentation?

A **Verifiable Presentation (VP)** is a container that wraps one or more Verifiable Credentials for transmission to a Verifier. The VP adds:

- **Holder signature** — Proof that the Holder authorized this presentation
- **Audience binding** — The presentation is targeted at a specific Verifier
- **Freshness** — A nonce ties the presentation to a specific request, preventing replay
- **Selective disclosure** — Only the requested claims are revealed (format-dependent)

### VP Structure (Conceptual)

```json
{
  "@context": ["https://www.w3.org/2018/credentials/v1"],
  "type": ["VerifiablePresentation"],
  "holder": "did:example:holder123",
  "verifiableCredential": [
    { /* VC 1 - potentially with selective disclosure */ },
    { /* VC 2 */ }
  ],
  "proof": {
    "type": "Ed25519Signature2020",
    "verificationMethod": "did:example:holder123#key-1",
    "challenge": "n-0S6_WzA2Mj",
    "domain": "https://verifier.example.com",
    "proofPurpose": "authentication",
    "proofValue": "z58DAdFfa9SkqZMVPxAQp..."
  }
}
```

### Key Properties

| Property | Purpose |
|----------|---------|
| `holder` | Identifies the presenter; Verifier checks this matches the credential subject |
| `verifiableCredential` | Array of VCs being presented; may be full credentials or selectively disclosed |
| `proof.challenge` | Nonce from the Verifier's request; prevents replay attacks |
| `proof.domain` | Intended Verifier; prevents presentations being redirected to other parties |
| `proof.proofValue` | Holder's signature over the presentation |

---

## Verification vs. Issuance: Key Differences

| Aspect | Issuance | Verification |
|--------|----------|--------------|
| **Protocol** | OID4VCI | OID4VP |
| **Initiator** | Issuer (offers credential) | Verifier (requests presentation) |
| **Data flow** | Issuer → Holder | Holder → Verifier |
| **Trust anchor** | Holder trusts Issuer | Verifier trusts Issuer (indirectly) |
| **Holder action** | Accept credential | Consent to share |
| **Key ceremony** | Holder generates key, Issuer binds it | Holder proves possession of bound key |
| **Privacy concern** | Issuer learns Holder identity | Issuer should NOT learn about presentation |

---

## The Verification Flow at a High Level

The following sequence represents the generalized flow used by OID4VP (OpenID for Verifiable Presentations), which all three implementations in this documentation follow in some form:

```
  Verifier                              Wallet (Holder)
    │                                        │
    │  1. Authorization Request              │
    │  ◄──────────────────────────────────── │
    │     (QR code, deep link, or redirect)  │
    │  ────────────────────────────────────► │
    │     (request + presentation_definition)│
    │                                        │
    │                                        │  2. Credential Matching
    │                                        │     [find matching credentials]
    │                                        │
    │                                        │  3. User Consent
    │                                        │     [display request, get approval]
    │                                        │
    │                                        │  4. Presentation Construction
    │                                        │     [build VP with selected claims]
    │                                        │     [sign with holder key]
    │                                        │
    │  5. Authorization Response             │
    │  ◄──────────────────────────────────── │
    │     (vp_token via redirect/POST)       │
    │                                        │
    │  6. Response Validation                │
    │     [verify signatures]                │
    │     [check revocation]                 │
    │     [validate holder binding]          │
    │                                        │
```

### Step 1 — Authorization Request

The Verifier initiates the flow by creating a presentation request. This request specifies what credentials and claims are needed, typically using a Presentation Definition (DIF PEX) or DCQL query. The request is delivered via QR code, deep link, or OAuth redirect.

### Step 2 — Credential Matching

The wallet searches its stored credentials to find those that satisfy the Verifier's request. Multiple credentials may match; the wallet may need to apply filtering (e.g., excluding revoked credentials).

### Step 3 — User Consent

The wallet displays the request to the user, showing:
- Who is requesting the information (Verifier identity)
- What information is being requested
- Why it is being requested (if provided)
- Whether the Verifier is trusted (certificate validation)

The user must explicitly consent before any data is shared.

### Step 4 — Presentation Construction

The wallet constructs a Verifiable Presentation containing:
- The selected credentials (potentially with selective disclosure)
- A holder binding proof signed with the key bound in the credentials
- The nonce from the Verifier's request

For SD-JWT credentials, this means revealing only selected claims. For mDoc credentials, this means constructing device-signed response. For JSON-LD credentials with BBS+, this means creating a derived proof.

### Step 5 — Authorization Response

The wallet transmits the VP to the Verifier. Transmission modes include:
- **Redirect** — Browser redirect with VP in URL fragment
- **Direct POST** — HTTP POST to Verifier's response endpoint
- **Proximity** — BLE or NFC for in-person verification

### Step 6 — Response Validation

The Verifier validates the received presentation:
- Credential signatures are valid
- Credentials were issued by trusted Issuers
- Credentials have not been revoked
- Holder binding proof is valid
- Nonce matches the request (freshness)
- Credential claims satisfy the original request

---

## Why Verification Matters

Verification is the moment of truth in the SSI ecosystem. It is where the value of a credential is realized:

- **Security**: A poorly implemented verification flow can accept forged credentials, miss revocation, or be susceptible to replay attacks
- **Privacy**: Improper verification can leak more data than necessary or enable tracking
- **User experience**: Complex consent flows or slow verification degrades adoption
- **Interoperability**: Verification must work across different wallet implementations and credential formats

The design decisions made in the verification flow directly impact all of these outcomes.

---

## Verification Trust Models

Different deployment scenarios require different trust assumptions:

### High-Trust, Regulated Environment (EUDI)

- Verifiers are registered and audited
- Verifier certificates come from trusted PKI
- Wallet validates Verifier identity before presenting
- Trust lists are maintained by governance authorities

### Open Ecosystem (Web3 / Decentralized)

- Verifiers may be unknown entities
- DID-based identification without central PKI
- User makes trust decisions based on context
- Reputation systems may inform trust

### Enterprise / B2B (Procivis)

- Verifiers are pre-registered business partners
- Trust is established through business relationships
- Credentials flow within defined organizational boundaries

### Consumer / Identity (Affinidi)

- Users control data sharing through consent
- Presentation through user's vault application
- Trust established through OAuth-like consent flows

---

**Next**: [Reading Guide](./reading-guide.md) — How to navigate this documentation based on your role.
