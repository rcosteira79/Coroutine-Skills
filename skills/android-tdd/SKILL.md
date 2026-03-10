---
name: android-tdd
description: Use when writing, fixing, or refactoring Android/KMP code in Kotlin — supplements superpowers:test-driven-development with Android's three-tier test model, fake-first strategy, coroutine testing, and Compose UI testing.
---

# Android TDD

## Overview

Extends `superpowers:test-driven-development` with Android-specific patterns.

**REQUIRED BACKGROUND:** You MUST follow `superpowers:test-driven-development`. This skill adds Android context — it does not replace the Iron Law or RED-GREEN-REFACTOR cycle.

## Android's Three Testing Tiers

Choose the lowest tier that can meaningfully test the behaviour:

| Tier | Location | Runs on | Speed | Use for |
|------|----------|---------|-------|---------|
| **Unit** | `src/test/` | JVM | Fast | Pure logic, ViewModels, UseCases, Repositories |
| **Integration** | `src/test/` with Robolectric | JVM (simulated) | Medium | Room DAOs, Context-dependent code, Fragment logic |
| **Instrumented** | `src/androidTest/` | Device/emulator | Slow | Compose UI, real DB, end-to-end flows |

**Rationalization trap:** "Instrumented tests are too slow" is never a reason to skip UI testing for UI code.

## Fakes Over Mocks

**Prefer hand-written fakes to mocking frameworks.**

Fakes implement the real interface with in-memory behaviour. They are:
- Easier to read and reason about
- Reusable across many tests
- Immune to internal implementation changes
- Not tied to call verification (which tests implementation, not behaviour)

Use a mock **only** when:
- The dependency has no meaningful state (pure function delegation)
- Verifying a specific interaction IS the behaviour under test (e.g. analytics events)
- Creating a fake would require duplicating complex external library logic

```kotlin
// PREFER: Fake with real interface
class FakeUserRepository : UserRepository {
    private val users = mutableMapOf<String, User>()
    var saveCallCount: Int = 0
        private set

    override suspend fun findById(id: String): User? = users[id]

    override suspend fun save(user: User) {
        saveCallCount++
        users[user.id] = user
    }
}

// AVOID: Mock with call verification
val mockRepo = mockk<UserRepository>()
every { mockRepo.save(any()) } just Runs
verify { mockRepo.save(expectedUser) } // tests implementation, not behaviour
```

## Coroutine Testing

Use `runTest` for all suspend functions. Never use `runBlocking` in tests.

```kotlin
@Test
fun `given invalid credentials, when logging in, then emits error state`() = runTest {
    // Given
    val fakeRepository = FakeAuthRepository()
    val viewModel = LoginViewModel(fakeRepository)

    // When
    viewModel.login(inputEmail = "bad@email.com", inputPassword = "wrong")
    advanceUntilIdle()

    // Then
    val actualState = viewModel.uiState.value
    assertIs<LoginUiState.Error>(actualState)
}
```

For `StateFlow` / `SharedFlow` collection, use [Turbine](https://github.com/cashapp/turbine):

```kotlin
viewModel.uiState.test {
    assertIs<LoginUiState.Loading>(awaitItem())
    assertIs<LoginUiState.Error>(awaitItem())
    cancelAndIgnoreRemainingEvents()
}
```

Dispatcher injection is required for testability:

```kotlin
class LoginViewModel(
    private val repository: AuthRepository,
    private val dispatcher: CoroutineDispatcher = Dispatchers.Main
) : ViewModel()
```

In tests, inject `UnconfinedTestDispatcher` or `StandardTestDispatcher` from `kotlinx-coroutines-test`.

## Compose UI Testing

Use `createComposeRule()` for component tests, `createAndroidComposeRule()` for integration tests needing Activity.

```kotlin
@get:Rule
val composeTestRule = createComposeRule()

@Test
fun `given loading state, when screen renders, then shows progress indicator`() {
    // Given
    val inputState = LoginUiState.Loading

    // When
    composeTestRule.setContent {
        LoginScreen(uiState = inputState, onLogin = {})
    }

    // Then
    composeTestRule.onNodeWithTag("loading_indicator").assertIsDisplayed()
}
```

Use `semantics` / `testTag` in production Composables to avoid brittle text-based selectors.

## Gradle Commands

```bash
# Run JVM unit tests (fast — run on every change)
./gradlew test

# Run unit tests for a specific module
./gradlew :feature:login:test

# Run a single test class
./gradlew :feature:login:test --tests "com.example.login.LoginViewModelTest"

# Run instrumented tests (requires connected device/emulator)
./gradlew connectedAndroidTest

# Run instrumented tests for a specific module
./gradlew :feature:login:connectedAndroidTest
```

## ViewModel Test Template (Given-When-Then)

```kotlin
class LoginViewModelTest {

    private val fakeRepository = FakeAuthRepository()
    private val viewModel = LoginViewModel(
        repository = fakeRepository,
        dispatcher = UnconfinedTestDispatcher()
    )

    @Test
    fun `given valid credentials, when logging in, then emits success state`() = runTest {
        // Given
        fakeRepository.willSucceedFor(inputEmail = "user@example.com")

        // When
        viewModel.login(inputEmail = "user@example.com", inputPassword = "pass123")

        // Then
        val actualState = viewModel.uiState.value
        assertIs<LoginUiState.Success>(actualState)
    }
}
```

## Common Rationalizations

| Excuse | Reality |
|--------|---------|
| "Instrumented tests are slow, I'll skip" | Slow tests > no tests. Optimise test selection, don't skip. |
| "Mocks are easier to write than fakes" | Fakes are more valuable and readable. Invest once, reuse everywhere. |
| "I can't test this ViewModel, it uses Dispatchers.Main" | Inject the dispatcher. Hard to test = design smell. |
| "This Composable is too simple to test" | Simple UI breaks. A `setContent` + `assertIsDisplayed` takes 2 minutes. |
| "Room DAOs test themselves" | Write DAO tests with in-memory database to verify your queries. |

## Red Flags

- Using `Thread.sleep()` in any test
- Using `runBlocking` instead of `runTest`
- Mocking data classes or simple value types
- No `testTag` on interactive/key UI components
- Instrumented tests for logic that could run on JVM
