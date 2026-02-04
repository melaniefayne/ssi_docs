# Credential Selection — Protocol and Standards

This document details the algorithms and data structures used for credential matching and selection as defined in DIF Presentation Exchange and related standards.

---

## Presentation Exchange Matching

### Input Descriptor Processing

For each input descriptor in the presentation definition:

```
for each input_descriptor in presentation_definition.input_descriptors:
    matching_credentials = []

    for each credential in wallet.credentials:
        if matches(credential, input_descriptor):
            matching_credentials.append(credential)

    if input_descriptor.group:
        grouped_matches[input_descriptor.group].add(matching_credentials)
    else:
        individual_matches[input_descriptor.id] = matching_credentials
```

### Constraint Matching Algorithm

```
function matches(credential, input_descriptor):
    // Check format compatibility
    if input_descriptor.format:
        if credential.format not in input_descriptor.format:
            return false

    // Check all field constraints
    for each field in input_descriptor.constraints.fields:
        if not field_matches(credential, field):
            if field.optional != true:
                return false

    return true
```

### Field Matching

```
function field_matches(credential, field):
    // Evaluate JSONPath expressions
    for each path in field.path:
        values = jsonpath.query(credential, path)

        if values.is_empty():
            continue  // Try next path

        // Apply filter if present
        if field.filter:
            for each value in values:
                if not filter_matches(value, field.filter):
                    return false

        return true  // At least one path matched

    return false  // No path matched
```

---

## JSONPath Expressions

### Syntax

| Expression | Meaning | Example |
|------------|---------|---------|
| `$` | Root object | `$` |
| `.key` | Object property | `$.name` |
| `[index]` | Array element | `$.items[0]` |
| `[*]` | All array elements | `$.items[*]` |
| `..key` | Recursive descent | `$..name` |

### Common Patterns

**Simple claim:**
```json
{ "path": ["$.credentialSubject.name"] }
```

**Nested claim:**
```json
{ "path": ["$.credentialSubject.address.city"] }
```

**Alternative paths (credential format variations):**
```json
{
  "path": [
    "$.credentialSubject.dateOfBirth",
    "$.vc.credentialSubject.dateOfBirth",
    "$.vct.dateOfBirth"
  ]
}
```

**Array element:**
```json
{ "path": ["$.credentialSubject.degrees[0].type"] }
```

---

## Filter Constraints

### JSON Schema Filters

```json
{
  "path": ["$.credentialSubject.age"],
  "filter": {
    "type": "integer",
    "minimum": 18
  }
}
```

### Supported Filter Types

| Type | Filter Properties | Example |
|------|-------------------|---------|
| `string` | `pattern`, `minLength`, `maxLength`, `enum` | `{"type": "string", "pattern": "^[A-Z]{2}$"}` |
| `integer` | `minimum`, `maximum`, `exclusiveMinimum`, `exclusiveMaximum` | `{"type": "integer", "minimum": 0}` |
| `number` | Same as integer | `{"type": "number", "maximum": 100.0}` |
| `boolean` | `const` | `{"type": "boolean", "const": true}` |
| `array` | `minItems`, `maxItems`, `contains` | `{"type": "array", "minItems": 1}` |

### Pattern Matching

```json
{
  "path": ["$.credentialSubject.nationality"],
  "filter": {
    "type": "string",
    "pattern": "^(DE|FR|IT|ES)$"
  }
}
```

### Enum Constraint

```json
{
  "path": ["$.credentialSubject.documentType"],
  "filter": {
    "type": "string",
    "enum": ["passport", "national_id", "drivers_license"]
  }
}
```

### Const Constraint

```json
{
  "path": ["$.credentialSubject.over18"],
  "filter": {
    "type": "boolean",
    "const": true
  }
}
```

---

## Submission Requirements

### Submission Requirement Rules

When presentation definition has `submission_requirements`:

```json
{
  "submission_requirements": [
    {
      "name": "Identity Verification",
      "rule": "pick",
      "count": 1,
      "from": "identity_credentials"
    }
  ],
  "input_descriptors": [
    { "id": "passport", "group": ["identity_credentials"], ... },
    { "id": "national_id", "group": ["identity_credentials"], ... },
    { "id": "drivers_license", "group": ["identity_credentials"], ... }
  ]
}
```

### Rule Types

| Rule | Meaning | Parameters |
|------|---------|------------|
| `all` | All descriptors in group must be satisfied | - |
| `pick` | Select from group | `count`, `min`, `max` |

### Nested Requirements

```json
{
  "submission_requirements": [
    {
      "rule": "all",
      "from_nested": [
        {
          "rule": "pick",
          "count": 1,
          "from": "group_a"
        },
        {
          "rule": "pick",
          "min": 2,
          "from": "group_b"
        }
      ]
    }
  ]
}
```

---

## Format-Specific Selection

### SD-JWT Selective Disclosure

Claims are selected by path:

```
SD-JWT Structure:
{
  "_sd": ["hash1", "hash2", ...],
  "iss": "https://issuer.example.com",
  "iat": 1683000000,
  ...
}
~disclosure1~disclosure2~...

Selection:
  path: ["given_name"]
  └── Include disclosure for "given_name"

  path: ["family_name"]
  └── Exclude disclosure (not requested)
```

### mDoc Element Selection

Elements selected by namespace and identifier:

```
mDoc Structure:
DeviceResponse {
  documents: [{
    docType: "org.iso.18013.5.1.mDL",
    issuerSigned: {
      nameSpaces: {
        "org.iso.18013.5.1": [
          { elementIdentifier: "given_name", elementValue: "John" },
          { elementIdentifier: "family_name", elementValue: "Doe" },
          ...
        ]
      }
    }
  }]
}

Selection:
  namespace: "org.iso.18013.5.1"
  element: "given_name"
  └── Include in DeviceResponse

  element: "family_name"
  └── Exclude from DeviceResponse
```

---

## Limit Disclosure

Input descriptors can require selective disclosure:

```json
{
  "id": "age_verification",
  "constraints": {
    "limit_disclosure": "required",
    "fields": [
      { "path": ["$.credentialSubject.over18"] }
    ]
  }
}
```

| Value | Meaning |
|-------|---------|
| `required` | Wallet MUST use selective disclosure, only reveal requested |
| `preferred` | Wallet SHOULD use selective disclosure if supported |

---

## Presentation Submission

After selection, wallet constructs a presentation submission:

```json
{
  "id": "submission_uuid",
  "definition_id": "presentation_definition_id",
  "descriptor_map": [
    {
      "id": "age_verification",
      "format": "vc+sd-jwt",
      "path": "$",
      "path_nested": {
        "format": "vc+sd-jwt",
        "path": "$.vp.verifiableCredential[0]"
      }
    }
  ]
}
```

### Descriptor Map Entry

| Field | Description |
|-------|-------------|
| `id` | Input descriptor ID being satisfied |
| `format` | Credential format used |
| `path` | JSONPath to credential in VP |
| `path_nested` | For nested credentials (VP containing VCs) |

---

## Revocation Filtering

Before presenting options, wallet filters by status:

```
function filter_valid_credentials(credentials):
    valid = []
    for credential in credentials:
        status = check_status(credential)
        if status == VALID:
            valid.append(credential)
        elif status == REVOKED:
            // Exclude from options
        elif status == SUSPENDED:
            // May include with warning
        elif status == UNKNOWN:
            // Include with warning
    return valid
```

---

## PEX v2 Changes

Presentation Exchange v2 introduces:

### Credential Queries

```json
{
  "credential_queries": {
    "query_1": {
      "format": "vc+sd-jwt",
      "claims": [
        { "path": ["given_name"] },
        { "path": ["family_name"] }
      ]
    }
  }
}
```

### Credential Sets

```json
{
  "credential_sets": [
    {
      "required": true,
      "options": [["query_1"], ["query_2", "query_3"]]
    }
  ]
}
```

Meaning: Either `query_1` alone OR both `query_2` and `query_3` together.

---

## Standards Reference

| Standard | Section | Topic |
|----------|---------|-------|
| DIF PEX v1.0 | 5 | Input Descriptors |
| DIF PEX v1.0 | 6 | Submission Requirements |
| DIF PEX v1.0 | 7 | Presentation Submission |
| DIF PEX v2.0 | 4 | Credential Queries |
| RFC 9535 | - | JSONPath |
| JSON Schema | - | Filter validation |
| SD-JWT | 5 | Selective Disclosure |
| ISO 18013-5 | 8.3 | mDoc Data Elements |
