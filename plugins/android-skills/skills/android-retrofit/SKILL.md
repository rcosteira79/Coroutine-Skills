---
name: android-retrofit
description: Use when setting up or working with Retrofit in Android — service interface definitions, coroutines integration, OkHttp configuration, Hilt module setup, and error handling in the repository layer.
---

# Android Networking with Retrofit

Modern Retrofit setup for Android using coroutines, `kotlinx.serialization`, and Hilt.

## Service Interface

Declare all endpoints as `suspend` functions. Use `Response<T>` when you need access to status codes or error bodies; use the body type directly when 2xx is the only expected success case.

```kotlin
interface GitHubService {

    // Direct body — throws HttpException on non-2xx
    @GET("users/{user}/repos")
    suspend fun listRepos(@Path("user") user: String): List<Repo>

    // Response wrapper — gives access to code, headers, error body
    @GET("users/{user}")
    suspend fun getUser(@Path("user") user: String): Response<User>
}
```

### URL Parameters

```kotlin
interface SearchService {

    @GET("search/users")
    suspend fun searchUsers(
        @Query("q") query: String,
        @Query("sort") sort: String? = null,
        @QueryMap options: Map<String, String> = emptyMap()
    ): SearchResult<User>

    @GET("orgs/{org}/members")
    suspend fun orgMembers(@Path("org") org: String): List<User>
}
```

### Request Bodies

```kotlin
interface UserService {

    @POST("users")
    suspend fun createUser(@Body user: CreateUserRequest): User

    @FormUrlEncoded
    @POST("user/edit")
    suspend fun updateUser(
        @Field("first_name") firstName: String,
        @Field("last_name") lastName: String
    ): User

    @Multipart
    @PUT("user/photo")
    suspend fun uploadPhoto(
        @Part("description") description: RequestBody,
        @Part photo: MultipartBody.Part
    ): User
}
```

### Headers

```kotlin
interface AuthService {
    // Static header
    @Headers("Cache-Control: no-cache")
    @GET("auth/refresh")
    suspend fun refreshToken(): TokenResponse

    // Dynamic header
    @GET("user/profile")
    suspend fun getProfile(@Header("Authorization") token: String): Profile
}
```

---

## Hilt Module Setup

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {

    @Provides
    @Singleton
    fun provideJson(): Json = Json {
        ignoreUnknownKeys = true
        coerceInputValues = true
        isLenient = true
    }

    @Provides
    @Singleton
    fun provideOkHttpClient(): OkHttpClient = OkHttpClient.Builder()
        .addInterceptor(
            HttpLoggingInterceptor().apply {
                level = if (BuildConfig.DEBUG) {
                    HttpLoggingInterceptor.Level.BODY
                } else {
                    HttpLoggingInterceptor.Level.NONE
                }
            }
        )
        .connectTimeout(30, TimeUnit.SECONDS)
        .readTimeout(30, TimeUnit.SECONDS)
        .build()

    @Provides
    @Singleton
    fun provideRetrofit(okHttpClient: OkHttpClient, json: Json): Retrofit = Retrofit.Builder()
        .baseUrl("https://api.example.com/")
        .client(okHttpClient)
        .addConverterFactory(json.asConverterFactory("application/json".toMediaType()))
        .build()

    @Provides
    @Singleton
    fun provideGitHubService(retrofit: Retrofit): GitHubService =
        retrofit.create(GitHubService::class.java)
}
```

---

## Error Handling in Repositories

Catch network exceptions at the repository layer. Never let `HttpException`, `IOException`, or `UnknownHostException` leak into the ViewModel. See `android-skills:android-data-layer` for the full repository pattern including offline-first strategies.

Use the project's existing domain error types. If none exist, use a unified sealed class for all data-layer errors (not separate hierarchies per data source):

```kotlin
class GitHubRepository @Inject constructor(
    private val service: GitHubService
) {
    suspend fun listRepos(user: String): Result<List<Repo>> = try {
        Result.success(service.listRepos(user))
    } catch (e: HttpException) {
        Result.failure(DataError.Server(e.code(), e.message()))
    } catch (e: IOException) {
        Result.failure(DataError.Network(e))
    }
}

// Reuse the same error hierarchy across the data layer — see android-skills:android-data-layer
sealed class DataError(message: String, cause: Throwable? = null) : Exception(message, cause) {
    class Network(cause: Throwable) : DataError("Network error", cause)
    class Server(val code: Int, message: String?) : DataError("Server error $code: $message")
    class Local(cause: Throwable) : DataError("Local storage error", cause)
}
```

---

## Authentication Interceptor

Add auth tokens via an `Interceptor` rather than individual `@Header` parameters:

```kotlin
class AuthInterceptor @Inject constructor(
    private val tokenProvider: TokenProvider
) : Interceptor {

    override fun intercept(chain: Interceptor.Chain): Response {
        val token = tokenProvider.getToken() ?: return chain.proceed(chain.request())
        val request = chain.request().newBuilder()
            .header("Authorization", "Bearer $token")
            .build()
        return chain.proceed(request)
    }
}
```

Inject it into `OkHttpClient` via the Hilt module.

---

## RIGHT vs WRONG Patterns

### `Response<T>` — use only when needed

```kotlin
// WRONG — wrapping every endpoint in Response<T> "just in case"
// Forces callers to check isSuccessful and handle nullable body, even when only the body matters
@GET("users/{user}/repos")
suspend fun listRepos(@Path("user") user: String): Response<List<Repo>>

// In repository — verbose and error-prone:
val response = service.listRepos(user)
if (response.isSuccessful) {
    Result.success(response.body()!!) // nullable body needs !! or ?. handling
} else {
    Result.failure(DataError.Server(response.code(), response.message()))
}

// RIGHT — direct return; Retrofit throws HttpException on non-2xx automatically
@GET("users/{user}/repos")
suspend fun listRepos(@Path("user") user: String): List<Repo>

// In repository — clean:
try {
    Result.success(service.listRepos(user)) // non-null, direct
} catch (e: HttpException) {
    Result.failure(DataError.Server(e.code(), e.message()))
}
```

RIGHT because direct return types are non-null and throw `HttpException` on non-2xx, giving you a clean try/catch at the repository level. Use `Response<T>` only when you need the error body content (e.g., validation messages) or response headers.

### Network exceptions leaking to ViewModel

```kotlin
// WRONG — ViewModel catches HttpException directly; couples UI layer to network internals
class RepoViewModel(private val service: GitHubService) : ViewModel() {
    fun loadRepos(user: String) {
        viewModelScope.launch {
            try {
                _uiState.value = UiState.Success(service.listRepos(user))
            } catch (e: HttpException) { // ViewModel knows about HTTP
                _uiState.value = UiState.Error("Server error: ${e.code()}")
            } catch (e: IOException) { // ViewModel knows about IO
                _uiState.value = UiState.Error("Network error")
            }
        }
    }
}

// RIGHT — repository maps to domain errors; ViewModel handles domain types only
class RepoViewModel(private val repository: GitHubRepository) : ViewModel() {
    fun loadRepos(user: String) {
        viewModelScope.launch {
            repository.listRepos(user)
                .onSuccess { repos -> _uiState.value = UiState.Success(repos) }
                .onFailure { error ->
                    _uiState.value = when (error) {
                        is DataError.Network -> UiState.Error("Check your connection")
                        is DataError.Server -> UiState.Error("Something went wrong")
                        else -> UiState.Error("Unknown error")
                    }
                }
        }
    }
}
```

WRONG because the ViewModel directly depends on Retrofit/OkHttp exception types. If you later swap Retrofit for Ktor, or add a cache layer, every ViewModel must change. The repository is the boundary — it maps network exceptions to domain error types that the ViewModel can handle without knowing the network implementation.

### Auth token: interceptor vs `@Header`

```kotlin
// WRONG — token parameter on every endpoint; easy to forget, duplicates logic
@GET("user/profile")
suspend fun getProfile(@Header("Authorization") token: String): Profile

@GET("user/settings")
suspend fun getSettings(@Header("Authorization") token: String): Settings

// Caller must pass token every time — copy-paste prone:
service.getProfile("Bearer $token")
service.getSettings("Bearer $token")

// RIGHT — interceptor adds token automatically to all requests
class AuthInterceptor(private val tokenProvider: TokenProvider) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val token = tokenProvider.getToken()
            ?: throw IOException("Auth token unavailable") // fail fast — see note below
        val request = chain.request().newBuilder()
            .header("Authorization", "Bearer $token")
            .build()
        return chain.proceed(request)
    }
}

// Clean service interface — no auth boilerplate:
@GET("user/profile")
suspend fun getProfile(): Profile

@GET("user/settings")
suspend fun getSettings(): Settings
```

WRONG because adding `@Header("Authorization")` to every endpoint is repetitive and fragile — one missing parameter means an unauthenticated request that fails at runtime, not compile time. An OkHttp interceptor applies the token uniformly to all requests.

> **Throw vs proceed:** Throw when all endpoints require auth — a missing token should surface immediately rather than producing a confusing 401. If the `OkHttpClient` is shared between authenticated and public endpoints, proceed without the header instead: `?: return chain.proceed(chain.request())`.

## Checklist

- [ ] All service functions are `suspend`
- [ ] Use `Response<T>` only when specific status code handling is needed
- [ ] `OkHttpClient` logging is gated behind `BuildConfig.DEBUG`
- [ ] Sensible connect and read timeouts configured
- [ ] Network exceptions mapped to domain types in the repository
- [ ] API DTOs mapped to domain/UI models — never expose Retrofit models to the ViewModel
