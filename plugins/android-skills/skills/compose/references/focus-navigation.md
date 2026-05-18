# Focus Navigation Reference

Focus is stateful UI behaviour. On TV, ChromeOS, foldables with a keyboard, desktop, and any device using a remote or D-pad, focus is the *primary* input model — clicks are secondary. Get focus wrong and the screen is unusable with a keyboard even if it looks right with a mouse.

This reference covers `FocusRequester`, `Modifier.focusable()`, `onFocusChanged`, `focusProperties`, key event handling, lazy-list focus restoration, focus traps, TV-specific patterns, predictive back, and how to test all of it.

Source: `androidx/compose/ui/focus/` and `androidx/compose/ui/input/key/`.

---

## Core Principle

**Make focus targets explicit, request focus after composition succeeds, and test navigation with the same input model users use.** A "click-only" test passes on a TV UI that is fundamentally broken.

Three rules govern every focus decision in this file:

1. **Don't fight the framework.** `Button`, `TextField`, `Modifier.clickable`, and `Modifier.selectable` already participate in focus. Add focus hooks only when behaviour demands it.
2. **Request focus from an effect, never from the composable body.** `LaunchedEffect` ensures composition has finished and the target node is attached.
3. **Restore focus by stable id, not by index.** Lists reorder; ids don't.

---

## When To Reach For Focus APIs

| Need | Add |
|------|-----|
| Normal button / text field / clickable focus | Nothing — the component is already focusable |
| Programmatic initial or restored focus | `FocusRequester` + `Modifier.focusRequester(...)` |
| Visual or state reaction to focus changes | `Modifier.onFocusChanged { ... }` |
| Custom interactive surface that isn't already focusable | `Modifier.focusable()` plus role/semantics |
| Override directional navigation (D-pad/arrow keys) | `Modifier.focusProperties { ... }` |
| Handle hardware keys (Back, Enter, custom) | `Modifier.onPreviewKeyEvent` / `Modifier.onKeyEvent` |
| Constrain focus to a region (modal, menu) | `FocusRequester` + `focusProperties` traps |

If none of the rows apply, the screen probably doesn't need any of this. Don't sprinkle `focusRequester` and `onFocusChanged` on every button "just in case" — that's how you get unmaintainable focus graphs.

---

## FocusRequester — Programmatic Focus

`FocusRequester` lets you request focus on a specific node from code. Pair it with `Modifier.focusRequester` to bind the requester to a target.

```kotlin
val requester = remember { FocusRequester() }

Button(
    onClick = onPlay,
    modifier = Modifier.focusRequester(requester),
) {
    Text("Play")
}

// Request focus from an effect, not the composable body:
LaunchedEffect(requester) {
    requester.requestFocus()
}
```

### Why effects, not composable body

`requestFocus()` from the composable body races against composition: the node may not be attached yet, the requester may be re-allocated on the next recomposition, or the call may execute before layout completes. `LaunchedEffect` guarantees composition has finished.

### Key the effect to the condition that makes the target exist

If the target appears after data loads, key the effect to that condition — not to `Unit`:

```kotlin
// BAD: fires once on initial composition, before items exist
LaunchedEffect(Unit) {
    firstItemRequester.requestFocus()
}

// GOOD: fires when items become available
LaunchedEffect(items.isNotEmpty()) {
    if (items.isNotEmpty()) {
        firstItemRequester.requestFocus()
    }
}
```

### Special values: Default and Cancel

```kotlin
// Defer to the framework's spatial search
focusProperties { up = FocusRequester.Default }

// Block focus from leaving in this direction
focusProperties { left = FocusRequester.Cancel }
```

Use `Cancel` to trap focus inside a region (a modal, a menu); use `Default` to opt back in to spatial search.

Source: `androidx/compose/ui/focus/FocusRequester.kt`.

---

## Modifier.focusable() — Custom Interactive Surfaces

A `Box` is not focusable. `Modifier.clickable` makes it focusable as a side effect. If a surface is interactive but you don't want a click handler, add `focusable()` explicitly:

```kotlin
Box(
    modifier = Modifier
        .focusable()
        .semantics {
            role = Role.Button
            contentDescription = "Drag handle"
        }
)
```

When you do this, also add the appropriate role/semantics so screen readers and accessibility services see the same thing focus does.

### Don't add `focusable()` to layout containers

`focusable()` on a `Column` or `Row` swallows focus that should reach children. If a container is supposed to be a single focus target (a card that acts as one big button), it's almost always cleaner to use `Surface(onClick = ...)` or `Modifier.clickable`.

---

## Modifier.onFocusChanged — React to Focus State

Read focus state to adjust visuals, expand a section, or scroll into view:

```kotlin
var isFocused by remember { mutableStateOf(false) }

Card(
    modifier = Modifier
        .onFocusChanged { state -> isFocused = state.isFocused }
        .scale(if (isFocused) 1.05f else 1f)
) {
    // ...
}
```

### `FocusState` properties

- `isFocused` — this node is the focused target.
- `hasFocus` — this node *or any descendant* is focused.
- `isCaptured` — focus has been forcibly held (rare, used for modals).

`hasFocus` is the one you usually want for "this section is currently active" indicators on grouped controls.

### Don't infer focus from selection

Focus ≠ selection. A user can D-pad through a list (focus moves) without selecting (commit pressed). Treating selection state as focus state breaks every keyboard/TV interaction.

```kotlin
// WRONG — focus is conflated with selection
if (item.id == selectedId) {
    Modifier.scale(1.05f)
}

// RIGHT — observe focus separately
Modifier.onFocusChanged { state -> isFocused = state.isFocused }
```

---

## Modifier.focusProperties — Override Spatial Navigation

Compose's default spatial focus search is good. Override it only when the layout creates an unsolvable graph (off-screen item, two valid targets in the same direction, focus should jump across the screen).

```kotlin
val header = remember { FocusRequester() }
val firstRow = remember { FocusRequester() }

Column(
    modifier = Modifier.focusProperties {
        // From this column, "down" jumps directly to the first row
        down = firstRow
        // Block "left" — keep focus inside the column
        left = FocusRequester.Cancel
    }
) {
    Header(modifier = Modifier.focusRequester(header))
    LazyRow {
        item { FirstCard(modifier = Modifier.focusRequester(firstRow)) }
        // ...
    }
}
```

### Rules

- Override **only the broken edges**, not every direction. Leave spatial search responsible for everything else.
- Don't override on every node — focus graphs that "describe themselves in code" rot the moment layout changes.
- `FocusRequester.Cancel` is the right tool for traps; `FocusRequester.Default` returns to framework search.
- If you find yourself overriding more than two directions on the same node, the layout is probably wrong — fix it instead.

Source: `androidx/compose/ui/focus/FocusProperties.kt`.

---

## Key Events — onPreviewKeyEvent / onKeyEvent

Handle hardware keys for behaviour that isn't normal click/focus traversal: back, custom shortcuts, escape from a dialog, search-key-opens-search.

```kotlin
Modifier.onPreviewKeyEvent { event ->
    when {
        event.type == KeyEventType.KeyUp && event.key == Key.Back -> {
            onBack()
            true
        }
        event.type == KeyEventType.KeyDown && event.key == Key.Escape -> {
            onDismiss()
            true
        }
        else -> false
    }
}
```

### Return `true` only when consumed

Returning `true` for every event swallows text input, accessibility shortcuts, parent navigation, and platform key handling. Default to `false`; consume only the keys you actually handle.

### `onPreviewKeyEvent` vs `onKeyEvent`

- `onPreviewKeyEvent` runs **before** children — use to intercept (e.g. shortcut that should win over a focused TextField).
- `onKeyEvent` runs **after** children — use when children get first chance (the normal case).

For most dialog/back/escape handling, `onPreviewKeyEvent` is correct because focus is usually on a TextField below.

### Common keys

| Key | Use |
|-----|-----|
| `Key.Back` | Hardware back (Android), browser back (Web) |
| `Key.Escape` | Dismiss dialogs/menus |
| `Key.Enter`, `Key.NumPadEnter`, `Key.DirectionCenter` | Activate focused item (D-pad center) |
| `Key.DirectionUp/Down/Left/Right` | D-pad / arrow navigation |
| `Key.Tab` | Focus traversal (Shift+Tab reverses) |
| `Key.Spacebar` | Activate focused button |

### Throttle expensive D-pad behaviour at the boundary

Rapid D-pad input on a TV remote can fire 10+ events per second. Throttle the *expensive* response (scrolling a row, paging a feed), not the key handler itself:

```kotlin
// Channel acts as a throttle queue — only one scroll job at a time
val scrollChannel = remember { Channel<Int>(Channel.CONFLATED) }

LaunchedEffect(scrollChannel) {
    scrollChannel.receiveAsFlow().collectLatest { delta ->
        listState.animateScrollBy(delta.toFloat())
    }
}

Modifier.onPreviewKeyEvent { event ->
    if (event.type == KeyEventType.KeyDown && event.key == Key.DirectionRight) {
        scrollChannel.trySend(itemWidth)
        true
    } else false
}
```

Source: `androidx/compose/ui/input/key/KeyEvent.kt`.

---

## Initial Focus

Initial focus is the most common reason to use `FocusRequester`. Get it right or every keyboard/TV navigation starts wrong.

### Single target

```kotlin
val initial = remember { FocusRequester() }

LaunchedEffect(initial) {
    initial.requestFocus()
}

TextField(
    value = query,
    onValueChange = onQueryChange,
    modifier = Modifier.focusRequester(initial),
)
```

### Target that appears after loading

Key the effect to the condition that brings the target into composition:

```kotlin
val firstItem = remember { FocusRequester() }

when (uiState) {
    is UiState.Loading -> CircularProgressIndicator()
    is UiState.Success -> {
        LaunchedEffect(uiState.items.isNotEmpty()) {
            if (uiState.items.isNotEmpty()) {
                firstItem.requestFocus()
            }
        }

        LazyColumn {
            itemsIndexed(uiState.items, key = { _, item -> item.id }) { index, item ->
                Row(
                    modifier = Modifier
                        .let { if (index == 0) it.focusRequester(firstItem) else it }
                ) { /* ... */ }
            }
        }
    }
}
```

The `let { if (...) ... }` pattern attaches the requester only to the intended target.

### Target inside a lazy list

Lazy lists don't compose items until they're scrolled into view. If you request focus on item 47 in a lazy list, the requester's node may not be attached yet. Either:

- Scroll first, then request: `listState.scrollToItem(47); requester.requestFocus()`.
- Or keep the first visible item focused and let spatial search take over.

---

## Lazy List Focus Restoration

When list content refreshes (pull-to-refresh, filter change, paging), focus disappears unless you restore it explicitly. The trap: restoring by index breaks the moment items reorder, get inserted, or deleted.

### Restore by stable id, not by index

```kotlin
@Composable
fun ArticleList(
    articles: ImmutableList<Article>,
    listState: LazyListState = rememberLazyListState(),
) {
    // Map of article id → FocusRequester, regenerated when ids change
    val requesters = remember(articles) {
        articles.associate { it.id to FocusRequester() }
    }

    // Track which article was focused so we can restore after refresh
    var lastFocusedId by rememberSaveable { mutableStateOf<String?>(null) }

    LaunchedEffect(articles, lastFocusedId) {
        val id = lastFocusedId ?: return@LaunchedEffect
        val requester = requesters[id] ?: run {
            // The previously-focused article no longer exists — fall back deterministically
            requesters.values.firstOrNull()?.requestFocus()
            return@LaunchedEffect
        }
        // Scroll into view first, then request focus
        val index = articles.indexOfFirst { it.id == id }
        if (index >= 0) {
            listState.scrollToItem(index)
            requester.requestFocus()
        }
    }

    LazyColumn(state = listState) {
        items(articles, key = { it.id }) { article ->
            ArticleRow(
                article = article,
                modifier = Modifier
                    .focusRequester(requesters.getValue(article.id))
                    .onFocusChanged { state ->
                        if (state.isFocused) lastFocusedId = article.id
                    }
            )
        }
    }
}
```

### Fallback chain when the focused id no longer exists

A deterministic fallback prevents focus from disappearing entirely:

1. Same id, if it still exists.
2. Nearest neighbour (item at the same index, or closest remaining).
3. First item.
4. Parent container.

Pick one for the screen and stick with it. Don't let the framework "do something reasonable" — it can't, because it doesn't know which behaviour the user expects.

---

## Focus Traps — Dialogs, Modals, Menus

When a modal opens, focus must stay inside it until it closes. Otherwise D-pad/Tab walks the user out the back of the modal into the obscured content.

```kotlin
@Composable
fun ConfirmationDialog(
    onConfirm: () -> Unit,
    onDismiss: () -> Unit,
) {
    val confirmRequester = remember { FocusRequester() }

    LaunchedEffect(confirmRequester) {
        confirmRequester.requestFocus()
    }

    Dialog(onDismissRequest = onDismiss) {
        Surface(
            modifier = Modifier
                // Block focus from escaping the dialog
                .focusProperties {
                    up = FocusRequester.Cancel
                    down = FocusRequester.Cancel
                    left = FocusRequester.Cancel
                    right = FocusRequester.Cancel
                }
                .onPreviewKeyEvent { event ->
                    if (event.type == KeyEventType.KeyUp && event.key == Key.Escape) {
                        onDismiss(); true
                    } else false
                }
        ) {
            Column {
                Text("Delete this item?")
                Row {
                    Button(onClick = onDismiss) { Text("Cancel") }
                    Button(
                        onClick = onConfirm,
                        modifier = Modifier.focusRequester(confirmRequester),
                    ) { Text("Delete") }
                }
            }
        }
    }
}
```

`Dialog` and `ModalBottomSheet` already trap focus on Android — but only if focus is inside them when they open. Always request initial focus on a button inside the modal; never rely on "the framework will handle it."

---

## TV and androidx.tv.material3

For TV apps, prefer `androidx.tv.material3` over the regular Material 3 library. The TV components ship with focus-aware visuals (focus rings, scale animations, surface elevation) by default and integrate with the leanback navigation model.

```kotlin
// build.gradle.kts
implementation("androidx.tv:tv-material:<version>")
implementation("androidx.tv:tv-foundation:<version>")
```

```kotlin
import androidx.tv.material3.Button
import androidx.tv.material3.Surface
import androidx.tv.foundation.lazy.list.TvLazyColumn

@Composable
fun TvBrowseScreen(rows: List<Row>) {
    TvLazyColumn {
        items(rows, key = { it.id }) { row ->
            ImmersiveList(row = row)  // Built-in focus-driven hero image
        }
    }
}
```

### TV-specific gotchas

- **Touch targets don't apply** — D-pad jumps don't need 48dp. Visual focus indication (ring, scale, glow) matters more.
- **Hover ≠ focus** on TV — there's no hover.
- **`Modifier.clickable` registers for D-pad center** automatically; you usually don't need a custom key handler for activation.
- **Don't trap horizontal focus on rows** — a TV row that swallows `left`/`right` at the edges feels broken. Let edges escape to siblings.

---

## Predictive Back Focus Restoration (Android 14+)

`PredictiveBackHandler` lets you animate during the back gesture, but it doesn't restore focus on the destination screen — you have to do that explicitly.

```kotlin
@Composable
fun DetailScreen(
    onBack: () -> Unit,
    returnFocusId: String?,
) {
    PredictiveBackHandler(enabled = true) { progress ->
        try {
            progress.collect { /* animate */ }
            onBack()
            // Caller (parent screen) restores focus to `returnFocusId` after navigation
        } catch (e: CancellationException) {
            // Gesture cancelled — focus stays on this screen
            throw e
        }
    }
}
```

Pair it with the lazy-list restoration recipe above: when the parent screen recomposes after back, its `lastFocusedId` (saved in `rememberSaveable`) drives the restoration. The detail screen's job is just to navigate; the source screen's `LaunchedEffect(articles, lastFocusedId)` does the rest.

See `compose/references/animation.md` for the visual side of `PredictiveBackHandler`.

---

## Testing Focus

Test focus with the same input model users use. A click test on a TV/keyboard UI proves nothing.

### performKeyInput + assertIsFocused

```kotlin
@Test
fun `D-pad down from header focuses first item`() {
    composeTestRule.setContent {
        AppTheme { BrowseScreen(uiState = previewSuccessState()) }
    }

    composeTestRule
        .onNodeWithTag("header")
        .performKeyInput {
            keyDown(Key.DirectionDown)
            keyUp(Key.DirectionDown)
        }

    composeTestRule.onNodeWithTag("first-item").assertIsFocused()
}
```

### Initial focus

```kotlin
@Test
fun `search screen focuses query field on entry`() {
    composeTestRule.setContent {
        AppTheme { SearchScreen(onQueryChange = {}) }
    }

    composeTestRule.onNodeWithTag("search-field").assertIsFocused()
}
```

### Focus restoration after refresh

```kotlin
@Test
fun `refreshing list restores focus to previously-focused article by id`() {
    val initial = previewArticles()
    val refreshed = initial.shuffled()  // same ids, different order

    composeTestRule.setContent {
        var articles by remember { mutableStateOf(initial) }
        Column {
            Button(onClick = { articles = refreshed }) { Text("Refresh") }
            ArticleList(articles = articles.toImmutableList())
        }
    }

    composeTestRule.onNodeWithTag("article-${initial[2].id}").performClick()
    composeTestRule.onNodeWithText("Refresh").performClick()
    composeTestRule.waitForIdle()

    // Focus restored to the same id, even though it moved to a different index
    composeTestRule.onNodeWithTag("article-${initial[2].id}").assertIsFocused()
}
```

### Don't screenshot focus state to assert ownership

Screenshot tests are right for the *appearance* of focus (ring colour, scale, elevation). They're wrong for asserting *which node owns focus* — use `assertIsFocused()`. A passing screenshot test with the wrong node focused tells you nothing.

See `android-tdd` for the broader test-shape decision (semantics-first selectors, screenshot determinism rules).

---

## Anti-Patterns

### `requestFocus()` in the composable body

```kotlin
// WRONG — races composition, may fire before node is attached
@Composable
fun Screen() {
    val requester = remember { FocusRequester() }
    requester.requestFocus()  // unsafe
    TextField(modifier = Modifier.focusRequester(requester), ...)
}

// RIGHT
@Composable
fun Screen() {
    val requester = remember { FocusRequester() }
    LaunchedEffect(requester) { requester.requestFocus() }
    TextField(modifier = Modifier.focusRequester(requester), ...)
}
```

### Initial focus keyed to `Unit` when target appears later

```kotlin
// WRONG — fires on first composition; items don't exist yet
LaunchedEffect(Unit) {
    firstItemRequester.requestFocus()
}

// RIGHT — keyed to the condition
LaunchedEffect(items.isNotEmpty()) {
    if (items.isNotEmpty()) firstItemRequester.requestFocus()
}
```

### Storing requesters by lazy list index

```kotlin
// WRONG — index is unstable across reorder
val requesters = remember { List(items.size) { FocusRequester() } }

// RIGHT — keyed by stable id
val requesters = remember(items) { items.associate { it.id to FocusRequester() } }
```

### Every button gets `focusRequester` + `onFocusChanged`

```kotlin
// WRONG — clutters every interactive element
Button(modifier = Modifier
    .focusRequester(remember { FocusRequester() })
    .onFocusChanged { /* unused */ }
) { ... }

// RIGHT — add only what behaviour requires
Button(onClick = onClick) { Text("Save") }
```

### Key handler returns `true` for everything

```kotlin
// WRONG — swallows text input, parent shortcuts, accessibility keys
Modifier.onPreviewKeyEvent { _ -> true }

// RIGHT — consume only handled keys
Modifier.onPreviewKeyEvent { event ->
    if (event.type == KeyEventType.KeyUp && event.key == Key.Back) {
        onBack(); true
    } else false
}
```

### Inferring focus from selection state

```kotlin
// WRONG — focus and selection are different concepts
val isHighlighted = item.id == selectedId
Box(modifier = Modifier.scale(if (isHighlighted) 1.05f else 1f))

// RIGHT — observe focus
var isFocused by remember { mutableStateOf(false) }
Box(modifier = Modifier
    .onFocusChanged { isFocused = it.isFocused }
    .scale(if (isFocused) 1.05f else 1f)
)
```

---

## Red Flags During Review

- **"It focuses correctly when I tap it"** for a screen that runs on TV, ChromeOS, or with a keyboard. Tap is irrelevant; the test should drive D-pad/Tab.
- **Initial focus works only with fixed data and fails after loading/refresh** — usually `LaunchedEffect(Unit)` instead of `LaunchedEffect(loadedCondition)`.
- **Focus state is inferred from selection state.** Two different concepts; conflating them breaks every keyboard interaction.
- **The focus graph is described in comments but not encoded or tested.** If it's not in `focusProperties` and not in a test, it doesn't exist.
- **`focusable()` on a `Column`/`Row` that has interactive children.** Swallows focus.
- **`Modifier.onPreviewKeyEvent { _ -> true }`** anywhere. Always returns `true` for keys it doesn't handle — that's a swallow, not a handler.
- **A modal opens and the first focusable child isn't requested.** Focus stays on whatever was focused under the modal; D-pad walks the user out the back.
- **Lazy list refresh loses focus** with no restoration recipe — the next D-pad press jumps to wherever the framework guesses.
- **Screenshot tests assert focus** instead of `assertIsFocused()`.

---

## Resources

- **Compose focus**: https://developer.android.com/develop/ui/compose/touch-input/focus
- **TV with Compose**: https://developer.android.com/develop/ui/compose/tv
- **Key events**: https://developer.android.com/develop/ui/compose/touch-input/handling-key-events
- **androidx.tv.material3**: https://developer.android.com/jetpack/androidx/releases/tv

For broader test-shape decisions and semantics-first selectors, see `android-skills:android-tdd`. For predictive-back visual animation, see `compose/references/animation.md`. For semantics layer (separate from focus traversal), see `compose/references/accessibility.md`.
