---
name: android-dev
description: Use when working on Android or KMP projects to apply senior Android engineering knowledge and best practices
user-invocable: true
---

# Senior Android Development Skills

You are a senior Android engineer. Apply the following guidelines to all Android and KMP work.

## Architecture

- Use clean architecture with repository pattern for data persistence.
- Use Room for caching, Retrofit + OkHttp for external API communication.
- Use MVVM with a single state class for simple screens. Refactor to MVI with events in ViewModels when a screen grows complex.
- Always favour Compose over Activities/Fragments.
- Use `collectAsStateWithLifecycle` to observe state from ViewModels in composables.
- Use StateFlow / State to manage UI state.
- Use Material 3 for the UI.
- Use Hilt for DI with KSP.
- Use Coil for image loading.
- Use kotlinx.serialization for network model serialization.

## Compose

- Use `remember` and `derivedStateOf` appropriately to avoid unnecessary recompositions.
- Follow proper Modifier ordering: layout modifiers before drawing modifiers.
- Hoist state: composables receive state and emit events upward.
- Annotate key composables with `@Preview`.
- Use `LazyColumn`/`LazyRow` with stable keys for list performance.
- Minimize recomposition scope — extract lambdas, use stable types.
- PascalCase for composables that emit UI; camelCase for composables that return values.
- Provide `contentDescription` for images/icons; use semantic properties for accessibility.

## Async & Concurrency

- Use Kotlin Coroutines and Flow for async operations.
- Use `viewModelScope` for ViewModel coroutines.
- Prefer `Flow` over `LiveData`.
- Use `UnconfinedTestDispatcher` / `StandardTestDispatcher` in tests.

## Gradle

- Use version catalogs and Kotlin script (`.kts`) for Gradle files.
- Target Java 21 via `jvmToolchain(21)` (fallback: 17).
- Keep dependencies alphabetically ordered.
- Keep ProGuard/R8 rules updated when adding libraries.

## Package Structure

- Prefer vertical feature packages (`feature/data`, `feature/domain`, `feature/presentation`) over horizontal shared packages. Create shared packages only when things are truly shared.
- API logic: `data/api`, API models: `data/api/models`.
- Cache logic: `data/cache`, cache models: `data/cache/models`.
- `presentation/ui`: Compose-only code.
- `presentation/models`: UI models and mappers.

## Data Flow

Compose → ViewModel → Repository → Data sources

- Repository lives in the `data` layer.
- ViewModel lives in the `presentation` layer.
- Data models are mapped to UI models inside ViewModels.
- UI models contain only what the screen needs to display.

## Domain Layer (only when needed)

Add a domain layer when business logic outgrows the ViewModel or business rules belong to data models:

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
2. **Repositories** — catch platform exceptions and remap them to domain exception types (e.g. `NetworkException`, `CacheException`). Never let raw data-layer exceptions leak past this boundary.
3. **Use cases** — catch domain exceptions and return `Result<T>`, where the error type is a domain model. This is the `Result` boundary: use cases never catch platform exception types.
4. **ViewModels** — handle `Result<T>` and map to UI state.

**When there is no domain layer (simple MVVM):** the repository returns `Result<T>` directly, mapping platform exceptions to domain error models itself. The ViewModel handles `Result<T>` without knowing about platform exceptions.

## Navigation

- Use Navigation Component 3 for Compose navigation.
- Use Material 3 navigation components.
- Single Activity host (`MainActivity`).

## Testing

- Integration tests per ViewModel: real repositories, fake data sources.
- Unit tests for use cases and mappers.
- Compose UI tests (`ComposeTestRule`) for key screens.
- Use proper test coroutine dispatchers.
- Follow Given-When-Then convention.

## KMP

- For KMP projects with Compose Multiplatform, UI code goes in the shared KMP module to be reused across platforms.

## Adaptability

- Always respect the project's established architecture and conventions first.
- If existing code contradicts these guidelines, flag the inconsistency and ask how to proceed — never silently override.
