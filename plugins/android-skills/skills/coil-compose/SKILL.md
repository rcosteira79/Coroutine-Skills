---
name: coil-compose
description: Use when loading images in Jetpack Compose with Coil — AsyncImage vs SubcomposeAsyncImage vs rememberAsyncImagePainter, ImageRequest configuration, placeholder/error states, and performance in lists.
---

# Coil for Jetpack Compose

## Primary API: `AsyncImage`

Use `AsyncImage` for the vast majority of cases. It resolves the image size from layout constraints automatically, which avoids loading oversized bitmaps.

```kotlin
AsyncImage(
    model = ImageRequest.Builder(LocalContext.current)
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
    model = ImageRequest.Builder(LocalContext.current)
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
ImageRequest.Builder(context)
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

## Hilt Setup

Provide a single `ImageLoader` instance for the whole app to share disk and memory caches:

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

Pass it to `AsyncImage` via `LocalContext` or inject it directly:

```kotlin
AsyncImage(
    model = url,
    contentDescription = null,
    imageLoader = imageLoader  // Injected via hiltViewModel or passed as param
)
```

---

## Decision Guide

| Scenario | Use |
|----------|-----|
| Standard image loading | `AsyncImage` |
| Need `Painter` for `Canvas`/`Icon` | `rememberAsyncImagePainter` + explicit `.size()` |
| Custom loading/error composables | `SubcomposeAsyncImage` (not in lists) |
| Decorative image (no accessibility meaning) | `contentDescription = null` |

---

## Checklist

- [ ] `AsyncImage` preferred over other variants
- [ ] `contentDescription` provided for meaningful images; `null` for decorative
- [ ] `crossfade(true)` enabled for smoother UX
- [ ] `SubcomposeAsyncImage` not used inside `LazyColumn` / `LazyRow`
- [ ] Single `ImageLoader` instance provided app-wide (shared cache)
- [ ] Explicit `.size()` set when using `rememberAsyncImagePainter`
