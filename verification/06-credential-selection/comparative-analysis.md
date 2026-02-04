# Credential Selection — Comparative Analysis

This document compares how EUDI, Procivis, and Affinidi approach credential matching, user selection, and selective disclosure.

---

## Selection Architecture

| Aspect | EUDI | Procivis | Affinidi |
|--------|------|----------|----------|
| **Matching location** | SDK (native) | Core library | Cloud service |
| **UI integration** | ViewModel + Transformer | React hooks | SDK callbacks |
| **Selection state** | UI model objects | React state | Platform state |
| **Disclosure control** | Per-claim checkboxes | Per-field toggles | Platform UI |

---

## Matching Capabilities

### Presentation Exchange Support

| Feature | EUDI | Procivis | Affinidi |
|---------|------|----------|----------|
| **PEX v1.0** | ✓ | ✓ | ✓ |
| **PEX v2.0** | Partial | ✓ | ✓ |
| **Input descriptors** | ✓ | ✓ | ✓ |
| **Submission requirements** | ✓ | ✓ | ✓ |
| **Nested requirements** | ✗ | ✓ | ✓ |
| **Credential queries** | ✗ | ✓ (V2) | ✓ |

### Constraint Support

| Constraint | EUDI | Procivis | Affinidi |
|------------|------|----------|----------|
| **Path expressions** | ✓ | ✓ | ✓ |
| **Type filters** | ✓ | ✓ | ✓ |
| **Pattern (regex)** | ✓ | ✓ | ✓ |
| **Enum** | ✓ | ✓ | ✓ |
| **Numeric range** | ✓ | ✓ | ✓ |
| **Limit disclosure** | ✓ | ✓ | ✓ |

---

## Pre-selection Strategies

### EUDI

```
SDK determines applicable credentials
    │
    ▼
Transform to UI model
    │
    ▼
Display all applicable (no pre-selection)
    │
    ▼
User selects
```

**Characteristics:**
- No automatic pre-selection
- All matching credentials shown equally
- User makes explicit choice

### Procivis

```
Core returns applicable + inapplicable
    │
    ▼
Pre-selection algorithm:
  1. ACCEPTED + APPLICABLE → Select
  2. ACCEPTED + INAPPLICABLE → Select
  3. First available → Fallback
    │
    ▼
V2: Sort by issuance date (newest first)
    │
    ▼
Display with pre-selected default
```

**Characteristics:**
- Smart pre-selection
- Prioritizes valid credentials
- Newest credential preference (V2)
- User can change selection

### Affinidi

```
Platform matches credentials
    │
    ▼
Platform suggests selection
    │
    ▼
User confirms or modifies
```

**Characteristics:**
- Platform-managed selection
- Simplified UX
- Less user control

---

## Multi-Credential Handling

| Scenario | EUDI | Procivis | Affinidi |
|----------|------|----------|----------|
| **Single credential per descriptor** | ✓ | ✓ | ✓ |
| **Multiple credentials per descriptor** | ✗ | ✓ (V2 `multiple`) | ✓ |
| **Credential sets** | ✗ | ✓ (V2) | ✓ |
| **Alternative credentials** | ✓ | ✓ | ✓ |

### Procivis V2 Multi-select

```typescript
if (credentialQuery.multiple) {
    // Allow multiple selections
    newSelection = uniq(newSelection.concat(credentialId));
} else {
    // Single selection replaces previous
    newSelection = [credentialId];
}
```

---

## Selective Disclosure UI

### EUDI

```
┌─────────────────────────────────────────┐
│ Driver's License                        │
│                                         │
│ ☑ Given Name: John          [Required]  │
│ ☑ Family Name: Doe          [Required]  │
│ ☐ Date of Birth: 1990-01-01 [Optional]  │
│                                         │
└─────────────────────────────────────────┘
```

- Checkboxes for each claim
- Required claims cannot be unchecked
- Optional claims default unchecked

### Procivis

```
┌─────────────────────────────────────────┐
│ National ID                             │
│                                         │
│ ● Given Name        ○ Required          │
│ ● Family Name       ○ Required          │
│ ○ Nationality       ○ Optional          │
│                                         │
│ Fields to share: 2 of 3                 │
└─────────────────────────────────────────┘
```

- Toggle switches
- Clear required/optional indicators
- Sharing summary

### Affinidi

Platform-provided UI with similar functionality but less customization.

---

## Revocation Handling

| Aspect | EUDI | Procivis | Affinidi |
|--------|------|----------|----------|
| **Check timing** | Before display | Before display | Real-time |
| **Revoked credentials** | Filtered out | Filtered out | Excluded |
| **Suspended credentials** | Depends on config | Warning shown | Configurable |
| **User notification** | Implicit (not shown) | Explicit warning | Platform message |

### EUDI Revocation Filter

```kotlin
.filterNot {
    walletCoreDocumentsController.isDocumentRevoked(it.docId)
}
```

### Procivis Revocation Check

```typescript
const credentialIds = new Set<string>(/* ... */);
await checkRevocation(Array.from(credentialIds));
```

---

## Path Handling Comparison

### EUDI: ClaimPathDomain

```kotlin
data class ClaimPathDomain(val path: List<String>)

// Prefix matching for nested claims
fun isPrefixOf(other: ClaimPathDomain): Boolean
```

- Path as list of strings
- Prefix matching for hierarchical claims
- Format-agnostic abstraction

### Procivis: Field IDs vs Paths

**V1:** Field IDs (opaque identifiers)
```typescript
submitClaims: string[]  // Array of field IDs
```

**V2:** JSON paths
```typescript
userSelections: string[]  // Array of JSON paths
```

### Format Mapping

| Format | EUDI | Procivis |
|--------|------|----------|
| **SD-JWT** | `["given_name"]` | `"given_name"` or `["given_name"]` |
| **mDoc** | `["namespace", "element"]` | `namespace:element` |

---

## Trade-off Analysis

### EUDI

**Strengths:**
- Clean SDK abstraction
- Type-safe transformers
- Consistent across platforms

**Weaknesses:**
- No smart pre-selection
- Limited PEX v2 support
- No multi-credential per descriptor

### Procivis

**Strengths:**
- Full PEX v1 and v2 support
- Smart pre-selection
- Multi-credential support
- Flexible state management

**Weaknesses:**
- More complex state management
- React-specific patterns
- Two code paths (V1/V2)

### Affinidi

**Strengths:**
- Simplified integration
- Platform-managed complexity
- Consistent UX

**Weaknesses:**
- Less customization
- Platform dependency
- Limited advanced features

---

## User Experience Comparison

| UX Aspect | EUDI | Procivis | Affinidi |
|-----------|------|----------|----------|
| **Initial display** | All options | Pre-selected | Suggested |
| **Selection effort** | High (user chooses) | Low (defaults) | Low |
| **Disclosure clarity** | Good | Very good | Good |
| **Error handling** | State events | Toast/screen | Platform |
| **Missing credential** | Error screen | Inapplicable list | Error |

---

## Recommendations

### Choose EUDI-style if:
- Building native mobile apps
- EU regulatory compliance needed
- Clean architecture priority
- SDK abstraction preferred

### Choose Procivis-style if:
- React Native development
- Full PEX v2 features needed
- Multi-credential scenarios
- Smart pre-selection valued

### Choose Affinidi-style if:
- Rapid development needed
- Simplified integration wanted
- Platform management acceptable
- Standard scenarios only

---

## Implementation Complexity

| Aspect | EUDI | Procivis | Affinidi |
|--------|------|----------|----------|
| **Matching logic** | SDK handles | Core + app logic | Platform |
| **State management** | ViewModel | React hooks + context | SDK |
| **UI customization** | Full control | Full control | Limited |
| **Testing** | Unit + UI tests | Jest + component | Platform |
| **Maintenance** | Platform-specific | Cross-platform | Managed |
