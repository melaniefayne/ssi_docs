# Credential Selection — Conceptual Overview

Credential selection is the wallet-side process of finding credentials that match verifier requirements and enabling the user to make informed sharing decisions.

---

## The Selection Challenge

When a wallet receives a presentation request, it must:

```
┌─────────────────────────────────────────────────────────────────┐
│  Presentation Request                                            │
│  "Need proof of age (over 18) and address"                       │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  Wallet Storage                                                  │
│                                                                  │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │ Driver's License│  │ National ID     │  │ Utility Bill    │ │
│  │ ✓ Age          │  │ ✓ Age          │  │ ✓ Address       │ │
│  │ ✓ Address      │  │ ✓ Address      │  │ ✗ Age           │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘ │
│                                                                  │
│  Which credential(s) should satisfy the request?                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Selection Process Overview

```
1. MATCHING
   Presentation Definition ──► Match against stored credentials
                                    │
                                    ▼
2. FILTERING
   Remove revoked, expired, or incompatible credentials
                                    │
                                    ▼
3. RANKING (optional)
   Order by relevance, freshness, or user preference
                                    │
                                    ▼
4. USER SELECTION
   Display options to user ──► User chooses credential(s)
                                    │
                                    ▼
5. CLAIM SELECTION
   Display claims ──► User selects which claims to share
                                    │
                                    ▼
6. CONSENT
   User confirms sharing decision
```

---

## Matching Mechanics

### Input Descriptors to Credentials

Each input descriptor in the presentation definition describes a required credential:

```
Input Descriptor:
{
  "id": "age_proof",
  "constraints": {
    "fields": [
      {
        "path": ["$.credentialSubject.dateOfBirth"],
        "filter": { "type": "string" }
      }
    ]
  }
}
```

The wallet checks each stored credential:

```
Credential A (Driver's License):
  └── Has dateOfBirth? ✓
  └── Field type matches? ✓
  └── Format compatible? ✓
  └── MATCH ✓

Credential B (Membership Card):
  └── Has dateOfBirth? ✗
  └── NO MATCH
```

### Applicable vs Inapplicable

Credentials fall into categories:

| Category | Definition | Example |
|----------|------------|---------|
| **Applicable** | Matches all required constraints | License with valid age claim |
| **Inapplicable** | Credential exists but doesn't fully match | License lacking required field |
| **Missing** | No credential of required type | No credential at all |

---

## User Selection Interface

### Single Credential Required

```
┌─────────────────────────────────────────────────────────────────┐
│  Select credential to share                                      │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ ○ Driver's License                                          ││
│  │   Issued: Jan 2024 · Expires: Jan 2029                      ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ ● National ID (Recommended)                                 ││
│  │   Issued: Mar 2023 · Expires: Mar 2033                      ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
│  [Continue]                                                      │
└─────────────────────────────────────────────────────────────────┘
```

### Multiple Credentials Required

```
┌─────────────────────────────────────────────────────────────────┐
│  Share credentials                                               │
│                                                                  │
│  For age verification:                                           │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ ● Driver's License                                          ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
│  For address verification:                                       │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ ○ Driver's License (same as above)                          ││
│  │ ● Utility Bill                                              ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
│  [Continue]                                                      │
└─────────────────────────────────────────────────────────────────┘
```

---

## Claim Selection (Selective Disclosure)

After credential selection, users choose which claims to share:

```
┌─────────────────────────────────────────────────────────────────┐
│  Review information to share                                     │
│                                                                  │
│  Driver's License                                                │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ Required:                                                   ││
│  │   ☑ Date of Birth: 1990-05-15                              ││
│  │   ☑ Address: 123 Main St                                   ││
│  │                                                             ││
│  │ Optional (verifier requested):                              ││
│  │   ☐ Full Name: John Doe                                    ││
│  │   ☐ License Number: DL123456                               ││
│  │                                                             ││
│  │ Not requested (will not be shared):                         ││
│  │   • Eye Color                                               ││
│  │   • Height                                                  ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
│  [Share Selected Information]                                    │
└─────────────────────────────────────────────────────────────────┘
```

### Disclosure Levels

| Level | Description | User Control |
|-------|-------------|--------------|
| **Required** | Verifier mandates, cannot deselect | None |
| **Optional** | Verifier requests, user can deselect | Yes |
| **Not requested** | Available but not asked for | Auto-excluded |

---

## Pre-selection Strategies

Wallets may pre-select credentials to streamline UX:

### Most Recent

```python
# Prefer newest credential of matching type
candidates.sort(by=issuance_date, descending=True)
return candidates[0]
```

### Most Minimal

```python
# Prefer credential that exposes least extra data
candidates.sort(by=extra_claims_count, ascending=True)
return candidates[0]
```

### User Preference

```python
# Check if user has set a default
if user.default_credential_for(type):
    return user.default_credential_for(type)
```

---

## Handling Edge Cases

### No Matching Credentials

```
┌─────────────────────────────────────────────────────────────────┐
│  Cannot complete request                                         │
│                                                                  │
│  ⚠ Missing required credential                                  │
│                                                                  │
│  The verifier requires:                                          │
│  • Proof of employment                                           │
│                                                                  │
│  You don't have a matching credential.                           │
│                                                                  │
│  [Cancel] [Get Credential]                                       │
└─────────────────────────────────────────────────────────────────┘
```

### Revoked Credential

```
┌─────────────────────────────────────────────────────────────────┐
│  Credential unavailable                                          │
│                                                                  │
│  ⚠ Your Driver's License has been revoked                       │
│                                                                  │
│  This credential can no longer be used for verification.         │
│                                                                  │
│  [Use Different Credential] [Cancel]                             │
└─────────────────────────────────────────────────────────────────┘
```

### Expired Credential

Some verifiers accept expired credentials with awareness:

```
┌─────────────────────────────────────────────────────────────────┐
│  Credential expired                                              │
│                                                                  │
│  ⚠ This credential expired on Jan 15, 2024                      │
│                                                                  │
│  The verifier may or may not accept it.                          │
│                                                                  │
│  [Share Anyway] [Use Different Credential]                       │
└─────────────────────────────────────────────────────────────────┘
```

---

## Privacy Considerations

### Correlation Risk

Sharing the same credential with multiple verifiers enables tracking:

```
Verifier A ──sees credential ID──► Can correlate
                                        │
Verifier B ──sees credential ID──► ◄────┘
```

Mitigation strategies:
- Wallet warns about repeated use
- Use credentials with unlinkability features (BBS+, ZKP)
- Issue multiple credentials for different contexts

### Over-sharing Prevention

Wallet should:
1. Clearly show what will be shared
2. Distinguish required vs optional
3. Exclude non-requested claims by default
4. Warn about sensitive data (government IDs, etc.)

---

## Consent Requirements

Before sharing, user must:

| Requirement | Implementation |
|-------------|----------------|
| See verifier identity | Display verifier name, trust status |
| See requested claims | List all claims to be shared |
| Understand purpose | Show verifier's stated purpose (if provided) |
| Explicit action | Require tap/click to share |
| Authentication | Biometric/PIN to authorize |

---

## Format-Specific Selection

### SD-JWT

Selection based on JSON paths:

```
Credential claims:
├── given_name
├── family_name
├── address
│   ├── street
│   ├── city
│   └── country
└── age_over_18

Verifier requests: ["age_over_18", "address.city"]

User selects: ✓ age_over_18, ✓ address.city
Disclosed: Only those two claims
```

### mDoc (ISO 18013-5)

Selection based on namespace + element:

```
Credential namespaces:
├── org.iso.18013.5.1
│   ├── given_name
│   ├── family_name
│   └── age_over_18
└── org.iso.18013.5.1.aamva
    └── domestic_driving_privileges

Verifier requests: org.iso.18013.5.1:age_over_18

User selects: ✓ age_over_18
Disclosed: Only that element
```

---

## Relationship to Key Binding

After selection, the wallet must prove the holder controls the credential's bound key:

```
Selection Complete
      │
      ▼
Retrieve private key for selected credential
      │
      ▼
Sign nonce/challenge with that key
      │
      ▼
Include proof in Verifiable Presentation
```

This is covered in detail in [Section 07: Holder Binding](../07-holder-binding/).
