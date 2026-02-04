# Comparative Summary

This section provides executive-level comparison and decision guidance for choosing between EUDI, Procivis ONE, and Affinidi verification implementations.

## Contents

1. [Feature Matrix](./feature-matrix.md) — Side-by-side feature comparison
2. [Decision Framework](./decision-framework.md) — How to choose an implementation
3. [Architectural Trade-offs](./architectural-trade-offs.md) — Deep dive into design decisions

---

## Executive Summary

### EUDI Wallet

**Best for:** European Digital Identity compliance, government-grade security, proximity verification

| Strength | Weakness |
|----------|----------|
| Hardware-backed keys | Two separate codebases (Android/iOS) |
| Full OID4VP v1.0 | Higher development complexity |
| BLE/NFC proximity | Platform-specific expertise needed |
| X.509 verifier trust | |

### Procivis ONE

**Best for:** Cross-platform efficiency, flexible protocol support, enterprise deployments

| Strength | Weakness |
|----------|----------|
| Single codebase | Remote signing dependency |
| PEX v1 + v2 support | React Native overhead |
| RSE security model | Less documented DCQL support |
| Protocol flexibility | |

### Affinidi

**Best for:** Web applications, rapid integration, consent-focused flows

| Strength | Weakness |
|----------|----------|
| Fastest integration | No offline/proximity |
| No mobile app needed | Cloud dependency |
| Portal-based config | Less customization |
| Consent audit logs | Different security model |

---

## Quick Decision Matrix

| If you need... | Choose |
|----------------|--------|
| EU regulatory compliance | EUDI |
| In-person verification (BLE/NFC) | EUDI or Procivis |
| Cross-platform mobile app | Procivis |
| Web-only verification | Affinidi |
| Hardware-backed keys | EUDI |
| PEX v2 support | Procivis |
| Quickest time to market | Affinidi |
| Self-hosted deployment | EUDI or Procivis |

---

## Implementation Complexity

| Aspect | EUDI | Procivis | Affinidi |
|--------|:----:|:--------:|:--------:|
| **Setup Time** | Days | Days | Hours |
| **Code Complexity** | High | Medium | Low |
| **Team Expertise** | Native mobile | React Native | Web/API |
| **Maintenance** | High (2 platforms) | Medium | Low (SaaS) |

---

## Security Model Comparison

| Aspect | EUDI | Procivis | Affinidi |
|--------|------|----------|----------|
| **Key Location** | Device hardware | Remote HSM | Cloud HSM |
| **Key Extraction** | Impossible | Not applicable | Not applicable |
| **Offline Signing** | Yes | No | No |
| **Attestation** | Hardware | Service | Service |
| **Verifier Auth** | X.509 PKI | Protocol-based | OAuth |

---

## Sections

- [Feature Matrix](./feature-matrix.md) — Detailed feature comparison
- [Decision Framework](./decision-framework.md) — Step-by-step selection guide
- [Architectural Trade-offs](./architectural-trade-offs.md) — Technical deep dive
