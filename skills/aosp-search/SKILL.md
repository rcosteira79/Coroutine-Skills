---
name: aosp-search
description: Use when needing to verify Android framework internals, find @hide APIs or internal constants, confirm underdocumented behavior, or trace how a framework class actually implements something — not just what the public API promises.
---

# AOSP Source Search

## Overview

Use `android.googlesource.com` (Gitiles) to fetch AOSP source files directly. **`cs.android.com` blocks automated fetching** (robots.txt `Disallow: /*`) — it is for human browsing only. For web searches, use it; for fetching file content, use Gitiles.

## When to Use

**Do search AOSP when:**
- Public docs say X but runtime behavior is Y
- You need `@hide` API details, internal constants, or default values
- Tracing how a framework class delegates to native or system service
- Confirming behavior across API levels

**Don't search AOSP when:**
- Public JavaDoc/KDoc answers the question
- The question is about AndroidX (use AndroidX source at `cs.android.com` or GitHub)
- You'd be guessing file paths — search first

## Key Repository Structure

| Repo | Path on Gitiles | Contains |
|------|-----------------|----------|
| `platform/frameworks/base` | `core/java/android/` | View, Activity, Context, etc. |
| `platform/frameworks/base` | `services/core/java/` | System services (AMS, WMS, etc.) |
| `platform/frameworks/base` | `core/res/` | Default attrs, styles, drawables |
| `platform/libcore` | `ojluni/src/main/java/` | Java core libraries |
| `platform/art` | — | Android Runtime |

## Fetching Source Files

### HTML view (readable by Claude)
```
https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/{path}
```

### Raw text (base64-encoded, smaller payload)
```
https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/{path}?format=TEXT
```
Decode with: the content is standard base64, readable after decoding.

### Example
```
# Fetch ViewGroup.java
https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/java/android/view/ViewGroup.java
```

## Search Strategy

cs.android.com is still the best **search UI** for humans. When you know the class name but not the file path, guide the user to:

```
https://cs.android.com/search?q=ClassName+methodName&ss=android
```

Then fetch the resulting file via Gitiles once you have the path.

For known classes, paths follow `core/java/{package/path}/{ClassName}.java`. Example:
- `android.view.ViewGroup` → `core/java/android/view/ViewGroup.java`
- `android.app.ActivityThread` → `core/java/android/app/ActivityThread.java`

## Common Patterns

| Goal | What to look for |
|------|-----------------|
| Default attribute value | `mGroupFlags \|=` or field initializer |
| Flag definition | `static final int FLAG_` or `static final int` near class top |
| Delegation to system service | `getService()` or `IActivityManager` calls |
| `@hide` method signature | Full method with `/** @hide */` Javadoc |
| API level gating | `Build.VERSION.SDK_INT >= Build.VERSION_CODES.X` |

## Common Mistakes

- **Trusting training data line numbers** — AOSP changes frequently. Always fetch current source.
- **Searching cs.android.com with WebFetch** — it's a JS SPA, results won't be in raw HTML.
- **Using `master` branch** — prefer `refs/heads/main` for recent AOSP; use a tag like `android-14.0.0_r74` for specific API levels.
- **Assuming AndroidX = AOSP** — AndroidX lives in a separate repo (`platform/frameworks/support`), better browsed at `cs.android.com/androidx`.
