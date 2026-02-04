# Presentation Definition — Protocol & Standards

This document details the DIF Presentation Exchange (PEX) and DCQL specifications used for defining credential requirements in OID4VP.

---

## DIF Presentation Exchange (PEX)

### Specification Reference

- **PEX v1.0**: https://identity.foundation/presentation-exchange/spec/v1.0.0/
- **PEX v2.0**: https://identity.foundation/presentation-exchange/spec/v2.0.0/

### Root Structure

```json
{
  "id": "32f54163-7166-48f1-93d8-ff217bdb0653",
  "name": "Identity Verification",
  "purpose": "We need to verify your identity to complete registration",
  "format": { ... },
  "input_descriptors": [ ... ],
  "submission_requirements": [ ... ]
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `id` | Yes | Unique identifier (UUID recommended) |
| `name` | No | Human-readable name |
| `purpose` | No | Explanation of why credentials are requested |
| `format` | No | Global format requirements |
| `input_descriptors` | Yes | Array of credential requirements |
| `submission_requirements` | No | Logical grouping rules |

### Input Descriptor Specification

```json
{
  "id": "identity_credential",
  "name": "Identity Credential",
  "purpose": "Verify your name and date of birth",
  "group": ["A"],
  "format": {
    "jwt_vc_json": {
      "alg": ["ES256", "ES384", "ES512"]
    }
  },
  "constraints": {
    "limit_disclosure": "required",
    "fields": [ ... ]
  }
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `id` | Yes | Unique within presentation definition |
| `name` | No | Display name |
| `purpose` | No | Why this specific credential is needed |
| `group` | No | Group memberships for submission requirements |
| `format` | No | Format requirements (overrides global) |
| `constraints` | No | Field requirements and filters |

### Field Constraint Specification

```json
{
  "path": ["$.credentialSubject.given_name", "$.vc.credentialSubject.given_name"],
  "id": "given_name_field",
  "name": "Given Name",
  "purpose": "Your first name as it appears on official documents",
  "filter": {
    "type": "string",
    "minLength": 1
  },
  "optional": false,
  "intent_to_retain": false
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `path` | Yes | JSONPath expressions to locate the claim |
| `id` | No | Identifier for field |
| `name` | No | Display name |
| `purpose` | No | Why this field is needed |
| `filter` | No | JSON Schema filter for value validation |
| `optional` | No | If true, field may be absent (default: false) |
| `intent_to_retain` | No | Whether verifier will store this value |

### JSONPath Syntax

PEX uses JSONPath for locating claims:

```
$                  Root object
$.property         Property access
$['property']      Bracket notation
$.array[0]         Array index
$.array[*]         All array elements
$.object.*         All object values
$..property        Recursive descent
```

**Examples:**

```json
// Direct property
"$.credentialSubject.name"

// Nested property
"$.credentialSubject.address.streetAddress"

// Array element
"$.credentialSubject.qualifications[0].name"

// Type array check
"$.type"

// Alternative paths (try in order)
["$.credentialSubject.given_name", "$.given_name", "$.vc.credentialSubject.given_name"]
```

### JSON Schema Filter

Filters use JSON Schema vocabulary:

```json
// String type
{ "type": "string" }

// String with constraints
{
  "type": "string",
  "minLength": 1,
  "maxLength": 100,
  "pattern": "^[A-Za-z ]+$"
}

// Specific value
{ "const": "verified" }

// Enumeration
{ "enum": ["active", "pending", "verified"] }

// Number range
{
  "type": "number",
  "minimum": 0,
  "maximum": 150
}

// Date format
{
  "type": "string",
  "format": "date"
}

// Array contains
{
  "type": "array",
  "contains": {
    "const": "IdentityCredential"
  }
}
```

### Submission Requirements

Logical grouping of input descriptors:

```json
{
  "submission_requirements": [
    {
      "name": "Identity Documents",
      "purpose": "Prove your identity",
      "rule": "pick",
      "count": 1,
      "from": "A"
    },
    {
      "name": "Proof of Address",
      "rule": "all",
      "from": "B"
    }
  ]
}
```

**Rules:**

| Rule | Parameters | Behavior |
|------|------------|----------|
| `all` | `from` | All descriptors in group required |
| `pick` | `from`, `count` | Exactly `count` from group |
| `pick` | `from`, `min`, `max` | Between min and max from group |

### Format Definitions

```json
{
  "format": {
    "jwt_vc_json": {
      "alg": ["ES256", "ES384", "EdDSA"]
    },
    "jwt_vc": {
      "alg": ["ES256"]
    },
    "ldp_vc": {
      "proof_type": ["Ed25519Signature2018", "BbsBlsSignature2020"]
    },
    "mso_mdoc": {
      "alg": ["ES256"]
    },
    "dc+sd-jwt": {
      "kb-jwt": {
        "alg": ["ES256"]
      }
    }
  }
}
```

---

## Presentation Definition v2 Differences

PEX v2.0 introduces:

### Frame Object (for JSON-LD)

```json
{
  "constraints": {
    "fields": [...],
    "subject_is_issuer": "preferred",
    "is_holder": [{
      "field_id": ["given_name_field"],
      "directive": "required"
    }]
  }
}
```

### Statuses

```json
{
  "constraints": {
    "statuses": {
      "active": {
        "directive": "required"
      },
      "revoked": {
        "directive": "disallowed"
      }
    }
  }
}
```

---

## DCQL (Digital Credentials Query Language)

### Specification Reference

- Defined in OID4VP v1.0 specification, Section 5.5

### Root Structure

```json
{
  "credentials": [ ... ],
  "credential_sets": [ ... ]
}
```

### Credential Query

```json
{
  "id": "pid",
  "format": "dc+sd-jwt",
  "meta": {
    "vct_values": ["https://credentials.example.com/identity"]
  },
  "claims": [
    { "path": ["given_name"] },
    { "path": ["family_name"] },
    {
      "path": ["birthdate"],
      "optional": true
    }
  ]
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `id` | Yes | Unique credential query identifier |
| `format` | Yes | Credential format |
| `meta` | No | Format-specific metadata |
| `claims` | No | Required/optional claims |
| `claim_sets` | No | Alternative claim groupings |

### Format-Specific Meta

**SD-JWT:**
```json
{
  "format": "dc+sd-jwt",
  "meta": {
    "vct_values": ["https://example.com/credentials/pid"]
  }
}
```

**mDoc:**
```json
{
  "format": "mso_mdoc",
  "meta": {
    "doctype_value": "org.iso.18013.5.1.mDL"
  }
}
```

### Claims Path

DCQL uses simple path arrays instead of JSONPath:

```json
// Simple claim
{ "path": ["given_name"] }

// Nested claim
{ "path": ["address", "locality"] }

// With namespace (mDoc)
{ "namespace": "org.iso.18013.5.1", "claim_name": "given_name" }
```

### Credential Sets

Logical grouping in DCQL:

```json
{
  "credential_sets": [
    {
      "options": [["pid"]],
      "required": true
    },
    {
      "options": [["address_proof"], ["utility_bill"]],
      "required": false
    }
  ]
}
```

Each `options` entry is an array of credential query IDs that together satisfy the set.

---

## Presentation Submission

The response includes a Presentation Submission mapping:

### PEX Presentation Submission

```json
{
  "id": "submission-1",
  "definition_id": "32f54163-7166-48f1-93d8-ff217bdb0653",
  "descriptor_map": [
    {
      "id": "identity_credential",
      "format": "jwt_vc_json",
      "path": "$",
      "path_nested": {
        "format": "jwt_vc_json",
        "path": "$.verifiableCredential[0]"
      }
    }
  ]
}
```

### DCQL Presentation Submission

```json
{
  "vp_token": [ ... ],
  "presentation_submission": {
    "pid": "credential-id-123"
  }
}
```

---

## Implementation Support Matrix

| Feature | EUDI | Procivis | Affinidi |
|---------|------|----------|----------|
| **PEX v1** | Via DCQL | ✓ | ✓ |
| **PEX v2** | Via DCQL | ✓ | Not documented |
| **DCQL** | ✓ | Not documented | Not documented |
| **limit_disclosure** | ✓ | ✓ | ✓ |
| **submission_requirements** | ✓ | ✓ | ✓ |
| **JSONPath** | ✓ | ✓ | ✓ |
| **JSON Schema filters** | ✓ | ✓ | ✓ |

---

## Standards References

| Standard | Section | Topic |
|----------|---------|-------|
| PEX v1.0 | § 4 | Presentation Definition |
| PEX v1.0 | § 5 | Input Descriptor |
| PEX v1.0 | § 6 | Submission Requirements |
| PEX v2.0 | § 4 | Updated Presentation Definition |
| OID4VP | § 5.4 | Presentation Definition usage |
| OID4VP | § 5.5 | DCQL specification |
| JSON Schema | Draft-07 | Filter vocabulary |
| JSONPath | RFC 9535 | Path expression syntax |
