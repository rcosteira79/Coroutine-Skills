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

Catch network exceptions at the repository layer. Never let `HttpException`, `IOException`, or `UnknownHostException` leak into the ViewModel.

```kotlin
class GitHubRepository @Inject constructor(
    private val service: GitHubService
) {
    suspend fun listRepos(user: String): Result<List<Repo>> = runCatching {
        service.listRepos(user)
    }.recoverCatching { throwable ->
        when (throwable) {
            is HttpException -> throw NetworkException.HttpError(throwable.code(), throwable.message())
            is IOException -> throw NetworkException.ConnectionError(throwable)
            else -> throw throwable
        }
    }
}

sealed class NetworkException(message: String) : Exception(message) {
    class HttpError(val code: Int, message: String) : NetworkException(message)
    class ConnectionError(cause: Throwable) : NetworkException(cause.message ?: "Connection failed")
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

## Checklist

- [ ] All service functions are `suspend`
- [ ] Use `Response<T>` only when specific status code handling is needed
- [ ] `OkHttpClient` logging is gated behind `BuildConfig.DEBUG`
- [ ] Sensible connect and read timeouts configured
- [ ] Network exceptions mapped to domain types in the repository
- [ ] API DTOs mapped to domain/UI models — never expose Retrofit models to the ViewModel
