# Presentation Definition — Conceptual Overview

A **Presentation Definition** is a structured query that describes the credentials and claims a Verifier requires. It serves as the contract between the Verifier's needs and the Holder's wallet, enabling automated credential matching and selective disclosure.

---

## Structure Overview

```
┌─────────────────────────────────────────────────────────────┐
│  Presentation Definition                                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  id: "example-verification"                                  │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Input Descriptor #1                                    │ │
│  │  ┌──────────────────────────────────────────────────┐  │ │
│  │  │  id: "identity-card"                              │  │ │
│  │  │  name: "Government ID"                            │  │ │
│  │  │  purpose: "Verify your identity"                  │  │ │
│  │  │                                                    │  │ │
│  │  │  format: { jwt_vc_json: { alg: ["ES256"] } }      │  │ │
│  │  │                                                    │  │ │
│  │  │  constraints:                                      │  │ │
│  │  │    fields:                                         │  │ │
│  │  │      - path: ["$.credentialSubject.given_name"]   │  │ │
│  │  │      - path: ["$.credentialSubject.family_name"]  │  │ │
│  │  │      - path: ["$.credentialSubject.birthdate"]    │  │ │
│  │  └──────────────────────────────────────────────────┘  │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Input Descriptor #2                                    │ │
│  │  ┌──────────────────────────────────────────────────┐  │ │
│  │  │  id: "proof-of-address"                           │  │ │
│  │  │  name: "Address Verification"                     │  │ │
│  │  │  purpose: "Confirm your residence"                │  │ │
│  │  │                                                    │  │ │
│  │  │  constraints:                                      │  │ │
│  │  │    fields:                                         │  │ │
│  │  │      - path: ["$.credentialSubject.address"]      │  │ │
│  │  └──────────────────────────────────────────────────┘  │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  submission_requirements: (optional logical grouping)        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Input Descriptors

Each input descriptor defines a single credential requirement:

### Identity

```json
{
  "id": "identity-card",
  "name": "Government ID",
  "purpose": "Verify your identity for account creation"
}
```

- **id**: Unique identifier, referenced in submission
- **name**: Human-readable name for user display
- **purpose**: Explains why this credential is needed

### Format Requirements

```json
{
  "format": {
    "jwt_vc_json": {
      "alg": ["ES256", "ES384"]
    },
    "mso_mdoc": {
      "alg": ["ES256"]
    }
  }
}
```

Specifies acceptable credential formats:
- `jwt_vc_json` — JWT-encoded VC
- `jwt_vc` — JWT VC (legacy)
- `ldp_vc` — JSON-LD VC with linked data proof
- `mso_mdoc` — ISO 18013-5 mobile document
- `dc+sd-jwt` — SD-JWT Verifiable Credential

### Constraints

The heart of the query — specifies required fields and filters:

```json
{
  "constraints": {
    "limit_disclosure": "required",
    "fields": [
      {
        "path": ["$.credentialSubject.given_name"],
        "filter": {
          "type": "string"
        }
      },
      {
        "path": ["$.credentialSubject.birthdate"],
        "filter": {
          "type": "string",
          "format": "date"
        }
      }
    ]
  }
}
```

---

## Field Specifications

### Path Syntax

JSONPath expressions locate claim values:

```json
// Simple path
{ "path": ["$.credentialSubject.name"] }

// Nested path
{ "path": ["$.credentialSubject.address.city"] }

// Array path
{ "path": ["$.credentialSubject.qualifications[0].title"] }

// Alternative paths (try each until match)
{ "path": [
    "$.credentialSubject.given_name",
    "$.vc.credentialSubject.given_name",
    "$.given_name"
  ]
}
```

### Filters

JSON Schema filters constrain acceptable values:

```json
// String type
{ "filter": { "type": "string" } }

// Specific value
{ "filter": { "type": "string", "const": "adult" } }

// Pattern match
{ "filter": { "type": "string", "pattern": "^[A-Z]{2}$" } }

// Numeric range
{ "filter": { "type": "number", "minimum": 18 } }

// Enum values
{ "filter": { "type": "string", "enum": ["verified", "confirmed"] } }

// Date format
{ "filter": { "type": "string", "format": "date" } }

// Array contains
{
  "filter": {
    "type": "array",
    "contains": {
      "type": "string",
      "pattern": "^IdentityCredential$"
    }
  }
}
```

### Optional vs Required Fields

```json
{
  "fields": [
    {
      "path": ["$.credentialSubject.given_name"],
      "optional": false  // Default: required
    },
    {
      "path": ["$.credentialSubject.middle_name"],
      "optional": true   // Can be missing
    }
  ]
}
```

### Intent to Retain

Signals data retention plans:

```json
{
  "path": ["$.credentialSubject.email"],
  "intent_to_retain": true  // Verifier will store this
}
```

---

## Submission Requirements

For complex queries with logical combinations:

### All Required (AND)

```json
{
  "submission_requirements": [
    {
      "name": "Identity Documents",
      "rule": "all",
      "from": "A"
    }
  ],
  "input_descriptors": [
    { "id": "passport", "group": ["A"] },
    { "id": "utility-bill", "group": ["A"] }
  ]
}
```

Holder must provide both passport AND utility bill.

### Pick One (OR)

```json
{
  "submission_requirements": [
    {
      "name": "Identity Document",
      "rule": "pick",
      "count": 1,
      "from": "A"
    }
  ],
  "input_descriptors": [
    { "id": "passport", "group": ["A"] },
    { "id": "drivers-license", "group": ["A"] },
    { "id": "national-id", "group": ["A"] }
  ]
}
```

Holder provides any one of the three.

### Minimum Count

```json
{
  "submission_requirements": [
    {
      "rule": "pick",
      "min": 2,
      "from": "A"
    }
  ]
}
```

Holder must provide at least 2 credentials from group A.

---

## Selective Disclosure Control

### limit_disclosure

Controls whether selective disclosure is required:

```json
{
  "constraints": {
    "limit_disclosure": "required",  // Must hide non-requested
    "fields": [...]
  }
}
```

| Value | Behavior |
|-------|----------|
| `required` | Only requested fields may be disclosed |
| `preferred` | Prefer selective disclosure if supported |
| (absent) | Full credential may be sent |

### Implications by Format

| Format | Selective Disclosure Support |
|--------|------------------------------|
| SD-JWT-VC | Full support (claim-level) |
| MSO-MDOC | Full support (element-level) |
| JWT-VC (plain) | No support (all or nothing) |
| JSON-LD + BBS+ | Full support (claim-level) |

---

## DCQL: Alternative Query Language

Digital Credentials Query Language is a simpler alternative:

```json
{
  "credentials": [
    {
      "id": "pid",
      "format": "dc+sd-jwt",
      "meta": {
        "vct_values": ["https://example.com/credentials/pid"]
      },
      "claims": [
        { "path": ["given_name"] },
        { "path": ["family_name"] },
        { "path": ["birthdate"], "optional": true }
      ]
    }
  ],
  "credential_sets": [
    {
      "options": [["pid"]],
      "required": true
    }
  ]
}
```

### DCQL vs PEX

| Aspect | PEX | DCQL |
|--------|-----|------|
| Complexity | Higher | Lower |
| Path syntax | JSONPath | Simple array |
| Format metadata | Generic | Format-specific |
| Logical grouping | submission_requirements | credential_sets |
| Industry adoption | Wide | Emerging |

---

## Credential Matching Process

When a wallet receives a Presentation Definition:

```
1. Parse presentation definition

2. For each input descriptor:
   ├─ Identify format requirements
   ├─ For each stored credential:
   │   ├─ Check format match
   │   ├─ For each required field:
   │   │   ├─ Apply path expression
   │   │   ├─ Check filter constraints
   │   │   └─ Record match/no-match
   │   └─ If all required fields match → credential eligible
   └─ Collect eligible credentials

3. Apply submission requirements:
   ├─ Evaluate logical rules (all, pick)
   ├─ Check minimum/maximum counts
   └─ Determine valid combinations

4. Present options to user:
   ├─ Show matching credentials
   ├─ Indicate required vs optional
   ├─ Allow selection if multiple match
   └─ Display field-level choices
```

---

## User Display

The wallet should present the definition meaningfully:

```
┌─────────────────────────────────────────────┐
│  Request from: Example Bank                  │
│  ✓ Verified                                  │
├─────────────────────────────────────────────┤
│                                              │
│  Required: Government ID                     │
│  "Verify your identity for account opening"  │
│                                              │
│  ┌───────────────────────────────────────┐  │
│  │  ☑ Given Name                         │  │
│  │  ☑ Family Name                        │  │
│  │  ☑ Date of Birth                      │  │
│  │  ☐ Address (optional)                 │  │
│  └───────────────────────────────────────┘  │
│                                              │
│  [Cancel]              [Share Selected]      │
└─────────────────────────────────────────────┘
```

Key display elements:
- **name**: Credential category name
- **purpose**: Why it's needed
- **fields**: Individual claims with selection
- **optional**: Clearly marked optional fields
