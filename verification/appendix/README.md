# Appendix

Reference materials for the SSI Verification documentation.

## Contents

1. [Glossary](./glossary.md) — Terms and definitions
2. [Standards Index](./standards-index.md) — Referenced specifications

---

## Quick Reference

### Key Acronyms

| Acronym | Full Name |
|---------|-----------|
| OID4VP | OpenID for Verifiable Presentations |
| PEX | Presentation Exchange |
| DCQL | Digital Credentials Query Language |
| VP | Verifiable Presentation |
| VC | Verifiable Credential |
| DID | Decentralized Identifier |
| SD-JWT | Selective Disclosure JWT |
| mDL | Mobile Driving License |
| BLE | Bluetooth Low Energy |
| NFC | Near Field Communication |
| RSE | Remote Secure Element |
| HSM | Hardware Security Module |

### Protocol Flow Summary

```
1. Verifier creates Authorization Request
   ├── presentation_definition (what to request)
   ├── client_id (who is asking)
   └── nonce (freshness)

2. Wallet receives request
   ├── Via QR code, deep link, or redirect

3. Wallet matches credentials
   ├── Against presentation_definition
   └── Filters revoked credentials

4. User consents
   ├── Reviews requested claims
   └── Selects optional claims

5. Wallet constructs VP Token
   ├── Selected credentials
   ├── Selective disclosure
   └── Holder binding proof

6. Wallet sends response
   ├── Via direct_post or redirect
   └── Includes presentation_submission

7. Verifier validates response
   ├── Signature verification
   ├── Holder binding check
   └── Revocation check
```
