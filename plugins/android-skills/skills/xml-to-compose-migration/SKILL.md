---
name: xml-to-compose-migration
description: Use when migrating Android XML layouts to Jetpack Compose — converting Views to Composables, mapping layout attributes, migrating state from LiveData/ViewBinding, and planning incremental adoption via ComposeView.
---

# XML to Compose Migration

Systematically convert Android XML layouts to idiomatic Jetpack Compose, preserving functionality while embracing Compose patterns.

## Workflow

### 1. Analyse the XML Layout

- Identify the root layout type (`ConstraintLayout`, `LinearLayout`, `FrameLayout`, etc.)
- List all View widgets and their key attributes
- Map data binding expressions (`@{}`) or ViewBinding references
- Identify custom views that need special handling
- Note any `include`, `merge`, or `ViewStub` usage

### 2. Plan the Migration

- Decide: **full rewrite** or **incremental** (embedding Compose into existing XML via `ComposeView`)
- Identify state sources (`ViewModel`, `LiveData`, `savedInstanceState`)
- List reusable components to extract as separate composables
- Plan navigation integration if the screen uses the Navigation component

### 3. Convert Layouts

Apply the mapping tables below to convert each View to its Compose equivalent.

### 4. Migrate State

- Convert `LiveData` → `StateFlow` + `collectAsStateWithLifecycle()`
- Replace `findViewById` / ViewBinding references with Compose state
- Convert click listeners to lambda parameters passed down from the screen composable

### 5. Test and Verify

- Compare visual output between XML and Compose versions
- Test accessibility (content descriptions, touch targets, TalkBack traversal)
- Verify state survives configuration changes (`rememberSaveable` where needed)

---

## Layout Mapping

### Container Layouts

| XML | Compose | Notes |
|-----|---------|-------|
| `LinearLayout (vertical)` | `Column` | Use `Arrangement` and `Alignment` |
| `LinearLayout (horizontal)` | `Row` | Use `Arrangement` and `Alignment` |
| `FrameLayout` | `Box` | Children stack; last child is on top |
| `ConstraintLayout` | `ConstraintLayout` (Compose) | Use `createRefs()` and `constrainAs` |
| `RelativeLayout` | `Box` or `ConstraintLayout` | Prefer `Box` for simple overlap |
| `ScrollView` | `Column` + `Modifier.verticalScroll()` | Use `LazyColumn` for lists |
| `HorizontalScrollView` | `Row` + `Modifier.horizontalScroll()` | Use `LazyRow` for lists |
| `RecyclerView` | `LazyColumn` / `LazyRow` / `LazyVerticalGrid` | Most common migration |
| `ViewPager2` | `HorizontalPager` | From `androidx.compose.foundation` |
| `CoordinatorLayout` + `AppBarLayout` | `Scaffold` + `TopAppBar` with `scrollBehavior` | |
| `NestedScrollView` | `Column` + `Modifier.verticalScroll()` | Prefer `LazyColumn` for large content |

### Common Widgets

| XML | Compose | Notes |
|-----|---------|-------|
| `TextView` | `Text` | Map `style` → `TextStyle` |
| `EditText` | `TextField` / `OutlinedTextField` | Requires state hoisting |
| `Button` | `Button` | Use `onClick` lambda |
| `ImageView` | `Image` (local) / `AsyncImage` (remote) | Use Coil for URLs |
| `ImageButton` | `IconButton` | Wrap `Icon` inside |
| `CheckBox` | `Checkbox` | Requires `checked` + `onCheckedChange` |
| `RadioButton` | `RadioButton` in a `Column` | Manage selected state externally |
| `Switch` | `Switch` | Requires state hoisting |
| `ProgressBar` (circular) | `CircularProgressIndicator` | |
| `ProgressBar` (horizontal) | `LinearProgressIndicator` | |
| `SeekBar` | `Slider` | Requires state hoisting |
| `Spinner` | `ExposedDropdownMenuBox` + `DropdownMenuItem` | More explicit pattern |
| `CardView` | `Card` | Material 3 |
| `Toolbar` | `TopAppBar` | Inside `Scaffold` |
| `BottomNavigationView` | `NavigationBar` | Material 3 |
| `NavigationView` (drawer) | `ModalNavigationDrawer` | Material 3 |
| `FloatingActionButton` | `FloatingActionButton` | Inside `Scaffold` |
| `Divider` | `HorizontalDivider` / `VerticalDivider` | |
| `Space` | `Spacer` + `Modifier.size()` | |

### Attribute Mapping

| XML Attribute | Compose Equivalent |
|---------------|--------------------|
| `layout_width="match_parent"` | `Modifier.fillMaxWidth()` |
| `layout_height="match_parent"` | `Modifier.fillMaxHeight()` |
| `layout_width="wrap_content"` | Default (implicit) |
| `layout_weight` | `Modifier.weight(1f)` |
| `android:padding` | `Modifier.padding()` |
| `android:layout_margin` | `Modifier.padding()` on parent, or `Arrangement.spacedBy()` |
| `android:background` | `Modifier.background()` |
| `visibility="gone"` | Conditional composition — don't emit the composable |
| `visibility="invisible"` | `Modifier.alpha(0f)` — keeps the space |
| `android:clickable` | `Modifier.clickable { }` |
| `android:contentDescription` | `contentDescription` parameter or `Modifier.semantics` |
| `android:elevation` | `Modifier.shadow()` or component `elevation` param |
| `android:alpha` | `Modifier.alpha()` |
| `android:rotation` | `Modifier.rotate()` |
| `android:scaleX/Y` | `Modifier.scale()` |
| `android:gravity` | `Alignment` / `Arrangement` parameters |
| `android:textSize` | `fontSize` in `TextStyle` |
| `android:textColor` | `color` in `Text()` |
| `android:textStyle="bold"` | `fontWeight = FontWeight.Bold` |
| `android:maxLines` | `maxLines` parameter in `Text` |
| `android:ellipsize` | `overflow = TextOverflow.Ellipsis` |

---

## Common Migration Patterns

### RecyclerView → LazyColumn

```kotlin
// Before: XML RecyclerView + Adapter
<androidx.recyclerview.widget.RecyclerView
    android:id="@+id/recycler"
    android:layout_width="match_parent"
    android:layout_height="match_parent" />

// After: Compose
LazyColumn(modifier = Modifier.fillMaxSize()) {
    items(items, key = { it.id }) { item ->
        ItemRow(item = item, onItemClick = onItemClick)
    }
}
```

### LiveData → StateFlow

```kotlin
// Before: LiveData in ViewModel
val userName: LiveData<String> = MutableLiveData("Alice")

// After: StateFlow in ViewModel
private val _userName = MutableStateFlow("Alice")
val userName: StateFlow<String> = _userName.asStateFlow()

// In composable:
val name by viewModel.userName.collectAsStateWithLifecycle()
```

### ViewBinding click listener → lambda

```kotlin
// Before: ViewBinding
binding.loginButton.setOnClickListener {
    viewModel.onLoginClicked(binding.emailInput.text.toString())
}

// After: Compose — state hoisted to screen, events flow up
LoginScreen(
    uiState = uiState,
    onLoginClick = viewModel::onLoginClicked
)

@Composable
fun LoginScreen(
    uiState: LoginUiState,
    onLoginClick: (String) -> Unit
) {
    var email by remember { mutableStateOf("") }
    TextField(value = email, onValueChange = { email = it })
    Button(onClick = { onLoginClick(email) }) { Text("Log in") }
}
```

### Incremental migration with ComposeView

For large apps, migrate one screen at a time using `ComposeView` inside the existing XML Activity/Fragment:

```xml
<!-- In existing XML layout -->
<androidx.compose.ui.platform.ComposeView
    android:id="@+id/compose_view"
    android:layout_width="match_parent"
    android:layout_height="match_parent" />
```

```kotlin
// In Activity or Fragment
val composeView = findViewById<ComposeView>(R.id.compose_view)
composeView.setViewCompositionStrategy(
    ViewCompositionStrategy.DisposeOnViewTreeLifecycleDestroyed
)
composeView.setContent {
    AppTheme {
        MyMigratedScreen(viewModel = viewModel())
    }
}
```

Always set `ViewCompositionStrategy.DisposeOnViewTreeLifecycleDestroyed` in Fragments to avoid state leaks.

---

## Gotchas

| Pitfall | Fix |
|---------|-----|
| `visibility="gone"` equivalent | Don't emit the composable at all — `if (isVisible) { MyComposable() }` |
| `drawableLeft` / `drawableRight` on `TextView` | Compose `Row` with `Icon` + `Text` |
| `include` / `merge` tags | Extract as a composable function |
| `ViewStub` (lazy inflation) | Conditional composable wrapped in `if` |
| `savedInstanceState` in Fragment | `rememberSaveable` in composable |
| Shared element transitions between Activities | Compose shared element transitions (`sharedElement` modifier, Navigation 2.8+) |
| Custom `View` subclass | Wrap in `AndroidView` initially; rewrite to composable over time |
