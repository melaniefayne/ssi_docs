# 01 — Architecture

This section documents the internal architecture of each wallet implementation studied, focusing on the modules, layers, and design patterns that govern credential issuance. Each sub-document describes a single implementation in depth; the comparative analysis draws structural contrasts across all three.

## Documents

| # | Document | Scope |
|---|----------|-------|
| 1 | [EUDI Wallet Architecture](eudi-architecture.md) | Native Android and iOS reference implementations from the EU Digital Identity initiative. Module hierarchy, MVI/Actor patterns, SDK boundary at `eudi-lib-android-wallet-core` / `eudi-lib-ios-wallet-core`. |
| 2 | [Procivis ONE Architecture](procivis-architecture.md) | React Native cross-platform wallet from Procivis AG. MobX State Tree, native bridge to `@procivis/react-native-one-core`, multi-transport abstraction, build flavor system. |
| 3 | [Affinidi Architecture](affinidi-architecture.md) | Cloud-service architecture from Affinidi. Credential Issuance Service, Affinidi Vault, TDK client libraries, configuration-driven issuance with OID4VCI. |
| 4 | [Comparative Analysis](comparative-analysis.md) | Side-by-side comparison of platform choices, architecture patterns, SDK boundaries, extensibility, and thick-client vs cloud-service trade-offs. |

## Key Themes

- **SDK boundary placement** — Where does application code end and protocol-level SDK code begin? Each implementation draws this line differently, with significant consequences for testability, upgradeability, and vendor lock-in.
- **State management strategy** — MVI with sealed classes (EUDI Android), Actor-based concurrency (EUDI iOS), MobX State Tree (Procivis), and cloud-managed state (Affinidi) represent four distinct approaches to the same problem.
- **Native vs cross-platform** — EUDI maintains two fully native codebases; Procivis uses React Native with a native core bridge; Affinidi sidesteps the question with a cloud service and multi-language TDK.
- **Protocol encapsulation** — All three delegate OID4VCI protocol mechanics to a dedicated component (`OpenId4VciManager`, the Procivis core engine, or the Affinidi Credential Issuance Service), but the depth of encapsulation varies.
