---
name: compose
description: >
  Use when the user mentions Compose, @Composable, remember, LaunchedEffect, Scaffold, NavHost,
  MaterialTheme, LazyColumn, Modifier, recomposition, Style, styleable, MutableStyleState, or
  any Jetpack Compose API. Use when reading or editing files that contain @Composable functions
  or import androidx.compose.*. Use when the user says "Android UI", "Kotlin UI", "compose
  layout", "compose navigation", "compose animation", "material3", "compose styles", or asks
  about modern Android UI patterns. Even casual mentions like "my compose screen is slow" or
  "how do I pass data between screens" should trigger this skill.
---

# Jetpack Compose Expert Skill

Non-opinionated, practical guidance for writing correct, performant Jetpack Compose code.
Focuses on real pitfalls developers encounter daily, backed by analysis of the actual
`androidx/androidx` source code (branch: `androidx-main`).

## Workflow

When helping with Compose code, follow this checklist:

### 1. Understand the request
- What Compose layer is involved? (Runtime, UI, Foundation, Material3, Navigation)
- Is this a state problem, layout problem, performance problem, or architecture question?

### 2. Consult the right reference
Read the relevant guidance file(s) from `references/` for patterns and principles:

| Topic | Reference File |
|-------|---------------|
| `@State`, `remember`, `mutableStateOf`, state hoisting, `derivedStateOf`, `snapshotFlow` | `references/state-management.md` |
| Structuring composables, slots, extraction, preview | `references/view-composition.md` |
| Modifier ordering, custom modifiers, `Modifier.Node` | `references/modifiers.md` |
| `LaunchedEffect`, `DisposableEffect`, `SideEffect`, `rememberCoroutineScope` | `references/side-effects.md` |
| `CompositionLocal`, `LocalContext`, `LocalDensity`, custom locals | `references/composition-locals.md` |
| `LazyColumn`, `LazyRow`, `LazyGrid`, `Pager`, keys, content types | `references/lists-scrolling.md` |
| `NavHost`, type-safe routes, deep links, shared element transitions | `references/navigation.md` |
| `animate*AsState`, `AnimatedVisibility`, `Crossfade`, transitions | `references/animation.md` |
| `MaterialTheme`, `ColorScheme`, dynamic color, `Typography`, shapes | `references/theming-material3.md` |
| Recomposition skipping, stability, baseline profiles, benchmarking | `references/performance.md` |
| Semantics, content descriptions, traversal order, testing | `references/accessibility.md` |
| Removed/replaced APIs, migration paths from older Compose versions | `references/deprecated-patterns.md` |
| `AndroidView` interop, `LiveData` migration, feature modularization | `references/interop-modularization.md` |
| **Styles API** (experimental): `Style {}`, `MutableStyleState`, `Modifier.styleable()` | `references/styles-experimental.md` |

### 3. Verify against live source
For any non-trivial answer, verify against actual androidx source — **always use live source, never rely on training data alone**.

Preferred: `android-sources` MCP
```
lookup_class(className: "LazyListState")
lookup_method(className: "Composer", methodName: "startRestartGroup")
search_in_source(query: "fun rememberLazyListState")
```

Fallback: `android-source-search` skill → `raw.githubusercontent.com/androidx/androidx/androidx-main/{path}`

### 4. Apply and cite
- Write code that follows the patterns in the reference and matches live source
- Flag any anti-patterns you see in the user's existing code
- Suggest the minimal correct solution — don't over-engineer
- Cite the source file when referencing internals:
```
// See: compose/runtime/runtime/src/commonMain/kotlin/androidx/compose/runtime/Composer.kt
```

## Key Principles

1. **Compose thinks in three phases**: Composition → Layout → Drawing. State reads in each
   phase only trigger work for that phase and later ones.

2. **Recomposition is frequent and cheap** — but only if you help the compiler skip unchanged
   scopes. Use stable types, avoid allocations in composable bodies.

3. **Modifier order matters**. `Modifier.padding(16.dp).background(Color.Red)` is visually
   different from `Modifier.background(Color.Red).padding(16.dp)`.

4. **State should live as low as possible** and be hoisted only as high as needed. Don't put
   everything in a ViewModel just because you can.

5. **Side effects exist to bridge Compose's declarative world with imperative APIs**. Use the
   right one for the job — misusing them causes bugs that are hard to trace.

## Source Code Receipts

When you need to verify internals or the user asks "show me the actual implementation", fetch live from `androidx/androidx` — always up to date, zero context overhead.

### Preferred: `android-sources` MCP (if available)
```
lookup_class(className: "LazyListState")
lookup_method(className: "Composer", methodName: "startRestartGroup")
list_class_members(className: "Modifier")
get_class_hierarchy(className: "LazyListState")
search_in_source(query: "fun rememberLazyListState")
```

### Fallback: `android-source-search` skill
Fetch raw files directly via `raw.githubusercontent.com`:
```
https://raw.githubusercontent.com/androidx/androidx/androidx-main/{path}
```

### Source tree map

| Module | Path prefix |
|--------|-------------|
| Runtime | `compose/runtime/runtime/src/commonMain/kotlin/androidx/compose/runtime/` |
| UI (common) | `compose/ui/ui/src/commonMain/kotlin/androidx/compose/ui/` |
| UI (Android) | `compose/ui/ui/src/androidMain/kotlin/androidx/compose/ui/platform/` |
| Foundation | `compose/foundation/foundation/src/commonMain/kotlin/androidx/compose/foundation/` |
| Material3 | `compose/material3/material3/src/commonMain/kotlin/androidx/compose/material3/` |
| Navigation | `navigation/navigation-compose/src/commonMain/kotlin/androidx/navigation/compose/` |

### Key files per module

| Module | Key files |
|--------|-----------|
| Runtime | `Composer.kt`, `Recomposer.kt`, `State.kt`, `Effects.kt`, `CompositionLocal.kt`, `Remember.kt`, `Snapshot.kt` |
| UI | `Modifier.kt`, `Layout.kt`, `LayoutNode.kt`, `ModifierNodeElement.kt`, `DrawModifier.kt` |
| Foundation | `LazyList.kt`, `LazyGrid.kt`, `BasicTextField.kt`, `Clickable.kt`, `Scrollable.kt`, `Pager.kt` |
| Material3 | `MaterialTheme.kt`, `ColorScheme.kt`, `Button.kt`, `Scaffold.kt`, `TextField.kt`, `NavigationBar.kt` |
| Navigation | `NavHost.kt`, `ComposeNavigator.kt`, `NavGraphBuilder.kt` |
