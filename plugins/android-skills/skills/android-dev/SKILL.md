---
name: android-dev
description: Use when working on Android or KMP projects to apply senior Android engineering knowledge and best practices
user-invocable: true
---

# Senior Android Development Skills

You are a senior Android engineer. Apply the following guidelines to all Android and KMP work.

## Skill Invocation Rule

**Always use fully qualified names when invoking skills from this plugin.** Use the `android-skills:` prefix — e.g., `android-skills:compose`, `android-skills:kotlin-coroutines`, `android-skills:android-tdd`. Never use short names like `compose` or `kotlin-coroutines` alone.

## Architecture

- Use clean architecture with repository pattern for data persistence.
- Ask the user whether they prefer MVVM or MVI. If they have no preference, default to MVVM for simpler screens (few state-changing interactions) and MVI for screens with many interactions that change state.
- Use Compose for all new UI. For legacy interop use `AndroidView` / `ComposeView`.
- Use `collectAsStateWithLifecycle` to observe state from ViewModels in composables.
- Use `StateFlow` / `State` to manage UI state.
- Use Material 3 for the UI.
- Use Hilt for DI with KSP.
- Use Coil for image loading.
- Use `kotlinx.serialization` for network model serialization.

**Android-only projects:**
- Room for local caching, Retrofit + OkHttp for network.

**KMP shared code:**
- Ktor for network (multiplatform). For local database, ask the user whether they prefer Room (KMP) or SQLDelight. If they have no preference, default to Room. Retrofit is Android-only and must not be used in shared modules.

## Existing-pattern check (before designing new mechanisms)

Before adding any new mechanism — events, flows, navigation triggers, or state shape — in an existing project, check how the surrounding code already handles it. The failure mode is inventing new mechanisms when existing ones should be reused, then dressing the shortcut in architectural language so it sounds principled.

### Audit procedure

Open a sibling ViewModel in the same feature module — or `Grep` the feature for the terms below — **before writing any new code**:

| Concern | What to look for |
|---|---|
| How actions reach the ViewModel | Sealed `Event` / `Intent` / `Action` interface, `onEvent()` dispatcher, and a Handler interface the ViewModel implements — check all three, not just one |
| How one-shot effects are emitted | Existing `SharedFlow` / `Channel` of sealed effect classes (see `android-skills:kotlin-flows`) |
| How navigation is triggered | Nav callbacks, `NavController` (Nav 2) or `NavDisplay` (Nav 3) use, or navigation effects in the effects stream |
| How the ViewModel exposes new behaviour | Event class entries + handler methods, or direct `fun` — match whichever the existing ViewModels use |
| How state is structured | `UiState` sealed classes, `StateFlow<State>` shape, field granularity |

### Red flags

- **Adding a public `fun foo()` on the ViewModel for the UI to call** when the project has a sealed `Event` + `onEvent()` pattern → add an Event data object and a handler method instead.
- **Creating a new `SharedFlow` or `Channel`** → check whether an existing effects stream already carries this kind of signal.
- **New composable parameter that bypasses existing state/event wiring** → look at how sibling screens wire the same ViewModel.
- **Justifying a direct approach with phrases like "natural extension of X," "the ViewModel delegates to Y," or "this is architecturally sound" — *without having first opened a sibling ViewModel***. Rationalization dressing up a shortcut is itself a signal to stop and audit, not a signal you're on the right track.

### Rule

**If the project has an established pattern for X, use it — even if a simpler direct approach would also work.** Simplicity is not a valid reason to diverge from the architecture. The cost of one extra `Event` data object and handler method is trivial; the cost of an architectural inconsistency is cumulative and paid by every future reader.

## Compose

For implementation detail, defer to `android-skills:compose`. Key architectural decisions:

- Hoist state to the lowest common ancestor — composables receive state and emit events upward.
- Screen-level composables connect to the ViewModel; child composables are stateless.
- For Compose specifics (stability, `remember`, Modifiers, side effects, navigation), `android-skills:compose` is the authoritative source.
- For M3 UX patterns (touch targets, canonical layouts, foldable postures, accessibility, M3 compliance audit), see `android-skills:android-ux`.

## Async & Concurrency

For implementation detail, defer to `android-skills:kotlin-coroutines` and `android-skills:kotlin-flows`. Key decisions:

- Use Kotlin Coroutines and Flow for all async work. No `LiveData` in new code.
- `viewModelScope` for ViewModel coroutines; inject `CoroutineDispatcher` for testability.
- Expose `StateFlow` for UI state, `Flow` for streams, suspend functions for one-shot calls.

## Gradle

- Use version catalogs (`libs.versions.toml`) and Kotlin script (`.kts`) for all Gradle files.
- Target Java 21 via `jvmToolchain(21)` (fallback: 17).
- Keep ProGuard/R8 rules updated when adding libraries.
- Add a Baseline Profile (`app/src/main/baseline-prof.txt`) for production apps to improve startup and scroll performance.

## Package Structure

### Single-module apps

- Prefer vertical feature packages (`feature/data`, `feature/domain`, `feature/presentation`) over horizontal shared packages. Create shared packages only when truly shared.
- API logic: `data/api`, API models: `data/api/models`.
- Cache logic: `data/cache`, cache models: `data/cache/models`.
- `presentation/ui`: Compose-only code.
- `presentation/models`: UI models and mappers.

### Multi-module apps

Follow a feature-vertical module structure:

```
:app                    ← entry point, wires features together
:core:model             ← shared domain models (pure Kotlin, no Android deps)
:core:data              ← repositories, data sources, Room DB, Retrofit
:core:domain            ← use cases, repository interfaces
:core:ui                ← shared composables, theme, design system
:feature:<name>         ← self-contained feature: own UI, ViewModel, nav entry point
```

- `:feature:*` modules depend on `:core:domain` and `:core:ui` — never on each other
- Domain models in `:core:model` have zero Android framework dependencies
- See `android-skills:android-gradle-logic` for Convention Plugin setup to share build config across modules

## Data Flow

Compose → ViewModel → Repository → Data sources

- Repository lives in the `data` layer.
- ViewModel lives in the `presentation` layer.
- Data models are mapped to UI models inside ViewModels.
- UI models contain only what the screen needs to display.

## Domain Layer (only when needed)

Add a domain layer when business logic outgrows the ViewModel or business rules belong to domain models:

- Domain models under `models`, use cases under `usecases`, repository interfaces under `repositories`.
- Domain layer has no dependency on data or UI layers.
- Data → domain → UI model mapping chain.
- Flow: Compose → ViewModel → use cases → repository interfaces ← repository impl → data sources.

## Error Handling

- Model success/error with sealed classes/interfaces (`Result<T>`).
- UI state must explicitly represent loading, success, and error.
- Never swallow exceptions silently in repositories or data sources.

**Error propagation by layer:**

1. **Data sources** — throw platform/library exceptions (`IOException`, `HttpException`, `SQLiteException`, etc.).
2. **Repositories** — catch platform exceptions and remap to domain error types (e.g. `DataError.Network`, `DataError.Local`). Use a single sealed error hierarchy for the data layer — see `android-skills:android-data-layer`. Never let raw data-layer exceptions leak past this boundary.
3. **Use cases** — catch domain exceptions and return `Result<T>`, where the error type is a domain model. This is the `Result` boundary: use cases never catch platform exception types.
4. **ViewModels** — handle `Result<T>` and map to UI state.

**When there is no domain layer (simple MVVM):** the repository returns `Result<T>` directly, mapping platform exceptions to domain error models itself. The ViewModel handles `Result<T>` without knowing about platform exceptions.

## Navigation

- Use `navigation-compose 2.8+` with type-safe `@Serializable` route objects (not string routes).
- Single Activity host (`MainActivity`). Navigate via `NavController` — never from the ViewModel directly.
- For one-time navigation/UI events from the ViewModel, ask the user: if exactly-once delivery is required (event must never be missed), use `Channel` + `receiveAsFlow()`; if events can be missed when UI is inactive, `SharedFlow(replay = 0)` is simpler. See `android-skills:kotlin-flows` for full trade-offs.
- See `compose/references/navigation.md` for full patterns.

## Background Work

- Use **WorkManager** for deferrable background tasks that must survive process death (sync, upload, periodic jobs).
- Use `CoroutineWorker` for suspend-friendly workers.
- Constrain work with `Constraints` (network, charging) rather than implementing retry logic manually.

```kotlin
val syncRequest = PeriodicWorkRequestBuilder<SyncWorker>(1, TimeUnit.HOURS)
    .setConstraints(
        Constraints.Builder()
            .setRequiredNetworkType(NetworkType.CONNECTED)
            .build()
    )
    .build()

WorkManager.getInstance(context).enqueueUniquePeriodicWork(
    "sync",
    ExistingPeriodicWorkPolicy.KEEP,
    syncRequest
)
```

## Testing

For implementation detail, defer to `android-skills:android-tdd`. Key decisions:

- Unit tests for use cases and mappers (pure JVM, no Android dependency).
- Integration tests per ViewModel: fake data sources, real ViewModel logic.
- Compose UI tests for key screens using `ComposeTestRule`.
- Screenshot tests with Roborazzi for visual regression.
- Follow Given-When-Then naming convention.

## KMP

- UI code lives in the shared KMP module (Compose Multiplatform) to be reused across platforms.
- Use `expect`/`actual` for platform-specific implementations (e.g. file I/O, push tokens, biometrics).
- Network layer: Ktor (shared). Database: ask user preference between Room (KMP) and SQLDelight; default to Room if no preference.
- Inject `CoroutineDispatcher` everywhere — `Dispatchers.Main` is not guaranteed on all KMP targets without the `-ktx` libraries.
- iOS: be mindful of the Kotlin/Native memory model; prefer immutable shared state.

## Adaptability

- Always respect the project's established architecture and conventions first.
- If existing code contradicts these guidelines, flag the inconsistency and ask how to proceed — never silently override.
