---
name: coil-compose
description: Use when loading images in Compose or Compose Multiplatform with Coil 3 — AsyncImage vs SubcomposeAsyncImage vs rememberAsyncImagePainter, ImageRequest configuration, placeholder/error states, performance in lists, and KMP setup with LocalPlatformContext.
---

# Coil 3 for Compose & Compose Multiplatform

## Primary API: `AsyncImage`

Use `AsyncImage` for the vast majority of cases. It resolves the image size from layout constraints automatically, which avoids loading oversized bitmaps.

```kotlin
AsyncImage(
    model = ImageRequest.Builder(LocalPlatformContext.current)
        .data("https://example.com/avatar.jpg")
        .crossfade(true)
        .build(),
    contentDescription = stringResource(R.string.user_avatar),
    contentScale = ContentScale.Crop,
    placeholder = painterResource(R.drawable.ic_placeholder),
    error = painterResource(R.drawable.ic_error),
    modifier = Modifier
        .size(64.dp)
        .clip(CircleShape)
)
```

---

## Custom State Slots: `SubcomposeAsyncImage`

Use `SubcomposeAsyncImage` only when you need fully custom composables for loading, success, and error states.

```kotlin
SubcomposeAsyncImage(
    model = "https://example.com/hero.jpg",
    contentDescription = null
) {
    when (painter.state) {
        is AsyncImagePainter.State.Loading -> CircularProgressIndicator()
        is AsyncImagePainter.State.Error -> Icon(Icons.Default.BrokenImage, null)
        else -> SubcomposeAsyncImageContent()
    }
}
```

> **Avoid in lists.** `SubcomposeAsyncImage` uses subcomposition, which is significantly slower than regular composition. Never use it inside `LazyColumn` or `LazyRow`.

---

## Low-Level: `rememberAsyncImagePainter`

Use only when a `Painter` is strictly required (e.g., `Canvas`, `Icon`, or a custom draw operation).

```kotlin
val painter = rememberAsyncImagePainter(
    model = ImageRequest.Builder(LocalPlatformContext.current)
        .data("https://example.com/image.jpg")
        .size(Size.ORIGINAL)  // Must specify size — not inferred automatically
        .build()
)
Image(painter = painter, contentDescription = null)
```

> Unlike `AsyncImage`, `rememberAsyncImagePainter` does **not** infer the display size. Without an explicit `.size()`, it loads the image at its original dimensions, which wastes memory. Always provide `.size()` or use `AsyncImage` instead.

---

## ImageRequest Configuration

```kotlin
ImageRequest.Builder(platformContext)
    .data(imageUrl)
    .crossfade(300)                            // Smooth fade-in (ms)
    .size(200, 200)                            // Explicit size to avoid over-fetching
    .scale(Scale.CROP)                         // Match ContentScale.Crop
    .transformations(CircleCropTransformation()) // Apply transformations
    .memoryCachePolicy(CachePolicy.ENABLED)
    .diskCachePolicy(CachePolicy.ENABLED)
    .build()
```

---

## Singleton ImageLoader Setup

Provide a single `ImageLoader` instance for the whole app to share disk and memory caches.

### Android with Hilt

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object ImageModule {

    @Provides
    @Singleton
    fun provideImageLoader(@ApplicationContext context: Context): ImageLoader =
        ImageLoader.Builder(context)
            .crossfade(true)
            .respectCacheHeaders(false)  // Ignore server cache-control headers if needed
            .build()
}
```

### KMP (commonMain)

Use `SingletonImageLoader.setSafe` early in the app lifecycle, or pass `ImageLoader` explicitly:

```kotlin
// Option 1: Set the singleton factory (call once at app startup)
SingletonImageLoader.setSafe {
    ImageLoader.Builder(it)
        .crossfade(true)
        .build()
}

// Option 2: Pass ImageLoader explicitly (no singleton needed)
val imageLoader = ImageLoader.Builder(LocalPlatformContext.current)
    .crossfade(true)
    .build()

AsyncImage(
    model = url,
    contentDescription = null,
    imageLoader = imageLoader
)
```

When using the `coil-compose` module (not `coil-compose-core`), `AsyncImage` without an `imageLoader` parameter automatically uses the singleton. With `coil-compose-core`, the `imageLoader` parameter is required.

---

## Decision Guide

| Scenario | Use |
|----------|-----|
| Standard image loading | `AsyncImage` |
| Need `Painter` for `Canvas`/`Icon` | `rememberAsyncImagePainter` + explicit `.size()` |
| Custom loading/error composables | `SubcomposeAsyncImage` (not in lists) |
| Decorative image (no accessibility meaning) | `contentDescription = null` |

---

## KMP / Compose Multiplatform

Coil 3.x is fully multiplatform. The `AsyncImage`, `SubcomposeAsyncImage`, and `rememberAsyncImagePainter` APIs are identical in `commonMain` — no `expect`/`actual` needed.

**Key difference:** Use `LocalPlatformContext.current` instead of `LocalContext.current`. On Android this resolves to `android.content.Context`; on other platforms it resolves to a singleton `PlatformContext.INSTANCE`.

```kotlin
// commonMain — works on all platforms
@Composable
fun Avatar(url: String) {
    AsyncImage(
        model = ImageRequest.Builder(LocalPlatformContext.current)
            .data(url)
            .crossfade(true)
            .build(),
        contentDescription = "User avatar",
        modifier = Modifier.size(48.dp).clip(CircleShape)
    )
}
```

**Dependencies:** Use `coil-compose` (singleton convenience) or `coil-compose-core` (explicit `imageLoader` parameter) in your `commonMain` source set. Add platform-specific network engines as needed (e.g., `coil-network-ktor3` for KMP networking).

---

## Checklist

- [ ] `AsyncImage` preferred over other variants
- [ ] `contentDescription` provided for meaningful images; `null` for decorative
- [ ] `crossfade(true)` enabled for smoother UX
- [ ] `SubcomposeAsyncImage` not used inside `LazyColumn` / `LazyRow`
- [ ] Single `ImageLoader` instance provided app-wide (shared cache)
- [ ] Explicit `.size()` set when using `rememberAsyncImagePainter`
- [ ] `LocalPlatformContext` used instead of `LocalContext` (KMP compatibility)
