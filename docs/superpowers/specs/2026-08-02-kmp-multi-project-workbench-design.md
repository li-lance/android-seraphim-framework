# KMP Multi-Project Workbench Design

**Status:** Approved

**Date:** 2026-08-02

## 1. Summary

Replace the current repository with a clean Kotlin Multiplatform workbench for creating and
validating independent multi-platform products. The existing applications, modules, build logic,
and compatibility boundaries may be discarded. Git history and this approved design remain the
starting point for the replacement.

The workbench uses a manifest-driven hybrid model:

- It maintains build conventions, composable project templates, a generator, and a real reference
  product.
- A generated product may stay inside the workbench or be exported as a standalone repository.
- Each product selects only the platforms and data strategy it needs.
- UI code is not shared between platforms.
- Kotlin Multiplatform shares domain logic, application use cases, data contracts, and optional
  synchronization logic.
- An optional Ktor/JVM server shares compatible domain rules and protocol models without sharing
  client persistence implementations.

The first reference product is a personal daily task board targeting Android, iOS, Desktop, and
Web, with a Ktor/PostgreSQL service added in a second delivery phase.

## 2. Goals

1. Generate a product from one versioned `project.yaml` file.
2. Allow every product to select Android, iOS, Desktop, Web/Wasm, and Ktor independently.
3. Support `local-only`, `offline-first`, and `remote-first` data strategies as optional template
   capabilities.
4. Provide both a Gradle entry point for in-workbench creation and a Kotlin CLI for standalone
   repository creation.
5. Keep generated projects conventional and editable without the generator.
6. Continuously prove supported templates by generating and building clean projects in CI.
7. Use the daily task board to validate the full platform and offline-first paths.

## 3. Non-goals

- Sharing UI, navigation, lifecycle abstractions, or platform view models.
- Building a low-code application platform or a universal feature DSL.
- Requiring every product to enable every platform.
- Bidirectionally synchronizing generated source code with evolving templates.
- Providing real-time multi-user collaboration, presence, or a generic CRDT framework.
- Providing end-to-end encrypted synchronization in the first two delivery phases.
- Building a cross-product identity platform.
- Preserving source or binary compatibility with the current repository.

## 4. Official Kotlin Multiplatform Alignment

The design follows the current Kotlin Multiplatform recommendations as of 2026-08-02:

- Application entry points are separate from shared code modules.
- KMP modules that expose an Android target use the Android-KMP library plugin required with AGP 9.
- Source sets use the default hierarchy template. The workbench does not manually reproduce the
  standard Apple or Native hierarchy.
- Shared code lives in `commonMain`, with platform integrations in platform or intermediate source
  sets and corresponding tests in `*Test` source sets.
- Platform entry points remain dedicated even when they consume the same shared logic.

References:

- [Recommended Kotlin Multiplatform project structure](https://kotlinlang.org/docs/multiplatform/multiplatform-project-recommended-structure.html)
- [Updating multiplatform projects with Android apps to use AGP 9](https://kotlinlang.org/docs/multiplatform/multiplatform-project-agp-9-migration.html)
- [The basics of Kotlin Multiplatform project structure](https://kotlinlang.org/docs/multiplatform/multiplatform-discover-project.html)
- [Hierarchical project structure](https://kotlinlang.org/docs/multiplatform/multiplatform-hierarchy.html)

Exact Kotlin, AGP, Gradle, Xcode, Node.js, and library versions are implementation-plan decisions.
They must be selected from a mutually compatible, stable toolchain and locked in the repository.

## 5. Workbench Architecture

The workbench has four top-level responsibilities:

```text
seraphim-workbench/
├── platform-kit/              # Included-build convention plugins and compatibility policy
├── templates/                 # Composable source templates and template metadata
├── tooling/                   # Manifest schema, generator engine, Gradle task, and Kotlin CLI
├── products/
│   └── daily-board/           # Full reference product
├── docs/                      # Architecture, product, and contributor documentation
└── .github/workflows/         # Build and template verification matrices
```

### 5.1 `platform-kit`

`platform-kit` owns build configuration, not product architecture. It provides a small set of
cohesive convention plugins:

- KMP library
- Android application
- Android library
- Desktop application
- Ktor service
- Kotlin/Wasm library
- Quality and test conventions

It also owns the supported toolchain matrix and a version catalog. Product modules keep product
identifiers, target choices, and product dependencies in their own build files.

### 5.2 `templates`

Templates are composable capabilities rather than one giant project snapshot. The supported
template units are:

- shared domain
- shared application
- shared data
- optional shared synchronization
- Android Jetpack Compose shell
- iOS SwiftUI shell and Xcode project
- Desktop Compose shell
- framework-independent TypeScript Web shell integration
- optional Ktor service
- local-only, offline-first, and remote-first data adapters

Templates contain metadata declaring requirements, incompatibilities, produced modules, and their
dependencies. A template is promoted as supported only after CI generates and builds its declared
combinations.

### 5.3 `tooling`

`tooling` contains one Kotlin generator engine. Both public entry points call this engine:

- `./gradlew createProduct` creates a product under `products/`.
- `seraphim new` creates a standalone repository in a caller-selected empty directory.

The CLI initially wraps the same engine used by Gradle; it is not a separate implementation.

### 5.4 `products/daily-board`

The daily task board is a maintained reference product, not disposable sample code. It proves the
full Android, iOS, Desktop, Web/Wasm, Ktor, and offline-first paths. Workbench capabilities are not
extracted from speculative needs: a capability becomes a reusable template only after the
reference product or a second real product demonstrates the boundary.

## 6. Product Manifest

`project.yaml` is declarative and schema-versioned. It expresses stable project choices and cannot
embed arbitrary Gradle or shell code.

The daily task board begins with this logical manifest:

```yaml
schema: 1
product:
  id: daily-board
  package: com.seraphim.dailyboard
platforms:
  android: true
  ios: true
  desktop: true
  web: true
server:
  ktor: true
data:
  strategy: offline-first
features:
  authentication: phase-2
  sync: phase-2
web:
  ui: external-typescript
```

Schema validation covers:

- product identifier and package naming
- selected target combination
- data strategy
- optional server and capabilities
- capability compatibility
- schema version

`phase-2` is reference-product delivery metadata. Generic generated products use `enabled` or
`disabled`; they do not inherit the daily board roadmap.

## 7. Generation Semantics

The generator performs six ordered steps:

1. Parse the manifest.
2. Validate the schema and capability combinations before writing product files.
3. Resolve a deterministic module and template graph.
4. Render into a new temporary directory.
5. Verify the generated structure and configured platform builds appropriate to the command mode.
6. Move the verified directory to its final destination.

Creation is allowed only when the destination does not exist or is empty. A failed generation does
not leave a partial product at the destination.

Re-running `new` or `createProduct` never overwrites an existing product. A separate, explicit
`migrate` command may upgrade manifest or template-owned infrastructure. Migration first produces a
change report and requires confirmation before applying changes. Product source code is never
silently replaced.

Generated products contain ordinary Gradle, Xcode, and Web project files and may evolve without the
workbench. A standalone product pins the platform-kit and template versions used to create it.

## 8. Generated Product Architecture

A full product has the following logical shape:

```text
product/
├── apps/
│   ├── android/               # Jetpack Compose UI and Android entry point
│   ├── ios/                   # SwiftUI UI and Xcode entry point
│   ├── desktop/               # Compose Desktop UI and JVM entry point
│   └── web/                   # Framework-selected TypeScript UI
├── services/
│   └── api/                   # Optional Ktor/JVM service
└── shared/
    ├── domain/                # Pure entities, value objects, rules, and repository ports
    ├── application/           # Use cases, commands, queries, and observable application state
    ├── data/                  # Repository implementations, DTO mapping, and data adapters
    └── sync/                  # Optional outbox, cursor, merge, and conflict logic
```

Products that do not need a target or capability omit that module entirely.

### 8.1 Dependency rules

- `shared:domain` depends only on Kotlin and deliberately selected common libraries.
- `shared:application` depends on `shared:domain`.
- `shared:data` implements ports defined by `shared:domain`; it does not expose persistence or HTTP
  models to UI modules.
- `shared:sync` depends on domain and data contracts, not on platform UI.
- Platform UI modules call application use cases and assemble platform adapters.
- Platform UI modules never depend on one another.
- The Ktor service may reuse compatible domain rules and protocol models, but never imports client
  database or client repository implementations.
- Cross-product `core` modules are created only after real reuse is demonstrated.

### 8.2 Source sets

Shared modules use the minimum target set required by the product. For the daily task board:

- `commonMain` and `commonTest`: domain, use cases, ports, shared data behavior, and most tests.
- `androidMain` and `androidUnitTest`: Android storage, files, clocks, and integration tests.
- default Apple hierarchy source sets: Native integrations shared by supported Apple targets.
- JVM source sets: Desktop and server-compatible infrastructure where the module contract allows
  reuse.
- `wasmJsMain` and its tests: Web exports and browser adapters.

The implementation plan must verify the exact source-set names produced by the chosen current
Kotlin Gradle plugin. It must use the default hierarchy rather than hard-coding a parallel graph.

## 9. Platform UI Boundaries

- Android uses Jetpack Compose.
- iOS uses SwiftUI.
- Desktop uses Compose Desktop on JVM.
- Web uses a product-selected TypeScript framework. The workbench does not require React, Vue, or
  another specific UI framework.

The Kotlin/Wasm boundary exposes a coarse-grained, JavaScript-friendly facade. It does not leak
Kotlin-specific collection, coroutine, exception, or generic types into TypeScript. The generated
Web integration includes TypeScript declarations and explicit lifecycle/disposal functions.

Each platform shell owns navigation, lifecycle, theme, accessibility, system permissions,
notifications, background execution, and platform dependency assembly. Platform adapters convert
shared state and results into native presentation state. Shared modules contain no platform view
model or UI component types.

## 10. Server and Protocol

The default optional server uses Ktor on JVM with PostgreSQL. It is Kotlin-based but remains
replaceable. OpenAPI is the stable client/server contract so a product may later implement the
service with Node.js, Go, a BaaS, or another stack without changing shared domain interfaces.

Generated API types are protocol models. Explicit mappers isolate them from domain entities and
database records. Authentication is an optional service capability and is introduced for the daily
task board in phase 2.

## 11. Daily Task Board Data Flow

### 11.1 Phase 1: complete local product

Each user command follows this flow:

1. A platform UI calls an application use case.
2. The use case validates domain rules.
3. The repository commits business data locally.
4. Observable application state publishes the committed change.
5. The UI updates without waiting for network availability.

Phase 1 supports task and board creation, editing, movement, completion, scheduling, filtering, and
local persistence on all four clients. It does not require an account or server.

### 11.2 Phase 2: offline-first synchronization

When synchronization is enabled, a local business mutation and its outbox operation are committed
in one transaction. Each operation has a stable operation identifier, aggregate identifier, base
version, logical timestamp, and payload.

The background synchronization loop:

1. Sends pending operations to the Ktor service.
2. The server accepts operations idempotently.
3. The server returns accepted, rejected, or conflicted results plus remote changes after the
   client's cursor.
4. The client confirms outbox entries, applies remote changes, records conflicts, and advances the
   cursor in one local transaction.
5. The client publishes the new observable state only after the transaction succeeds.

Retries cannot duplicate business operations. A sync cursor is never advanced before all returned
changes are durably applied.

### 11.3 Conflict policy

The daily board uses domain-specific conflict rules instead of a generic CRDT:

- Concurrent changes to different fields merge by field.
- Concurrent changes to the same scalar field use last-write-wins with retained conflict audit
  metadata.
- Delete wins over update; deleted records remain restorable from a recycle bin.
- Reordering and movement use stable ranks, with server-side rank normalization.
- Unsafe automatic merges create a `ConflictRecord` for explicit user resolution.

## 12. Error Handling

Shared application operations return a typed product error model:

- validation errors are displayed immediately and never enter the outbox;
- storage errors roll back the transaction and retain the previously committed UI state;
- network errors retain pending operations and retry with bounded exponential backoff and jitter;
- authentication errors pause synchronization without deleting local product data;
- protocol incompatibility stops the affected sync batch and records actionable diagnostics;
- unresolved merge conflicts remain queryable until explicitly resolved.

Platform shells map typed errors to native UI. Unexpected exceptions are captured at platform
boundaries with sensitive values removed from diagnostics.

## 13. Testing and Quality Gates

CI is layered so fast checks run before platform matrices:

### 13.1 Fast checks

- formatting and static analysis
- manifest schema tests
- generator graph and rendering unit tests
- forbidden dependency checks

### 13.2 Shared core checks

- `commonTest` domain and use-case tests
- JVM and Android integration tests
- repository contract tests run against every selected data adapter
- synchronization state-machine and protocol contract tests

### 13.3 Platform checks

- Android assemble and focused UI smoke test
- iOS simulator build and focused XCTest smoke test
- Desktop build/package and focused UI smoke test
- Wasm production build and browser smoke test
- Ktor service test and database migration test when the server is enabled

### 13.4 Template checks

CI generates at least:

- the default Android+iOS local-only product;
- a minimal single-platform product for each supported platform;
- the full daily-board platform combination with offline-first data and Ktor;
- a standalone exported repository.

Each generated fixture is built from a clean directory. A template combination is not documented
as supported until this clean-generation check passes.

## 14. Versioning

The workbench, manifest schema, and template catalog have separate semantic versions. The repository
records a compatibility table for Kotlin, AGP, Gradle, JDK, Xcode, Node.js, Ktor, and the Web build
toolchain.

Dependency update automation opens reviewable changes and runs the complete affected platform and
template matrix. Generated projects pin their toolchain and template inputs; they do not silently
follow workbench mainline changes.

## 15. Delivery Phases

### Phase 0: workbench foundation

- Replace the legacy repository contents with the new top-level structure.
- Establish the locked toolchain and focused convention plugins.
- Implement schema v1 and the generator engine.
- Provide the Gradle creation entry point.
- Generate and build a default Android+iOS local-only product in CI.

### Phase 1: local daily task board and standalone CLI

- Build the daily-board domain, application, and local data modules.
- Provide independently implemented Android, iOS, Desktop, and Web UI shells.
- Complete local task-board workflows on all four clients.
- Add the Kotlin CLI and standalone repository export.
- Add full platform clean-generation checks.

### Phase 2: account and cloud synchronization

- Add Ktor, PostgreSQL, OpenAPI, database migrations, and local container development.
- Add authentication, transactional outbox, incremental cursor sync, and conflict records.
- Add offline, retry, idempotency, and multi-device end-to-end tests.

### Phase 3: template productization

- Promote proven daily-board capabilities into stable optional templates.
- Implement explicit manifest and template migrations with change reports.
- Publish a supported compatibility matrix.
- Create a second real product to test that workbench abstractions are genuinely reusable.

Each phase produces independently buildable, testable software. Phase 1 remains a useful product
without phase 2.

## 16. Acceptance Criteria

The redesigned workbench is successful when:

1. A contributor can create a default product without copying or editing convention Gradle logic.
2. Unselected platforms are absent from the generated module graph and are not configured by
   Gradle.
3. The daily task board builds for Android, iOS, Desktop, and Web from a clean checkout.
4. The local task board remains fully usable without a server or network.
5. A generated standalone product builds after being moved outside the workbench repository.
6. Invalid manifest combinations fail before the destination is modified.
7. Template verification detects a broken supported combination before release.
8. Phase 2 synchronization retries idempotently and never advances a cursor before local commit.
9. UI code is not shared across platform UI modules.
10. The legacy project has no compatibility or migration requirement beyond retained Git history.
