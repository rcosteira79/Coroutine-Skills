# Android Skills for Claude Code

Claude Code skills for Android and KMP development — covering architecture, data layer, networking, testing, debugging, Jetpack Compose, coroutines, flows, Gradle, and RxJava migration.

## Skills

### `android-dev`
Senior Android engineering knowledge and best practices for Android and KMP projects. Covers architecture, code quality, and platform-specific patterns.

> Inspired by [awesome-android-agent-skills](https://github.com/new-silvermoon/awesome-android-agent-skills)

### `android-tdd`
Test-driven development for Android/KMP — extends TDD with Android's three-tier test model, fake-first strategy, coroutine testing, and Compose UI testing.

> Inspired by [awesome-android-agent-skills](https://github.com/new-silvermoon/awesome-android-agent-skills)

### `android-ux`
Material Design 3 UX principles for Android — touch targets (48×48dp), 8dp spacing grid, navigation patterns (Bottom Bar, Rail, Drawer), safe area handling, accessibility, animation timing, and keyboard input types.

### `android-debugging`
Debugging Android and KMP issues — Logcat, ADB, ANR traces, R8 stack trace decoding, memory leaks, Gradle build failures, and Compose recomposition bugs.

### `compose`
Jetpack Compose expert guidance — state management (`@Composable`, `remember`, `mutableStateOf`, `derivedStateOf`, state hoisting), Modifier chains, lazy lists, navigation, animation, side effects, theming, accessibility, and performance optimization.

> Inspired by [compose-skill](https://github.com/aldefy/compose-skill) and [awesome-android-agent-skills](https://github.com/new-silvermoon/awesome-android-agent-skills). Both skills take the same approach of bundling AndroidX source code alongside guidance docs — the difference is in scope and how the guidance is structured.

### `android-source-search`
Fetch and verify Android source code — AOSP platform internals (`@hide` APIs, framework classes, system services via Gitiles) and AndroidX/Jetpack library source and samples (via GitHub). Also useful when public docs are insufficient to complete a task.

> **Looking for more power?** [android-source-explorer-mcp](https://github.com/mrmike/android-source-explorer-mcp) is a purpose-built MCP server that goes much further: local source sync, sub-10ms Tree-sitter parsing, method-level extraction, class hierarchy, and cross-file navigation via LSP. If you're doing serious framework investigation, set that up instead — this skill is the zero-setup fallback.

### `kotlin-coroutines`
Dispatcher selection, scope management, structured concurrency, cancellation, exception handling, and Android/KMP async patterns. Includes the DispatcherProvider pattern for testable dispatcher injection.

> Inspired by [awesome-android-agent-skills](https://github.com/new-silvermoon/awesome-android-agent-skills)

### `kotlin-flows`
Flow type selection (`Flow`/`StateFlow`/`SharedFlow`), operator chains, callback bridging, lifecycle-safe collection, Channel migration, and UI state management.

### `rxjava-migration`
Triggered only when you explicitly ask to migrate. Assesses complexity, maps RxJava types and operators to coroutines equivalents, and provides interop patterns for incremental migration.

> Inspired by [awesome-android-agent-skills](https://github.com/new-silvermoon/awesome-android-agent-skills)

### `xml-to-compose-migration`
Migrate Android XML layouts to Jetpack Compose — layout mapping tables (RecyclerView → LazyColumn, LinearLayout → Column/Row, etc.), attribute mapping, state migration from LiveData/ViewBinding, and incremental adoption via `ComposeView`.

> Inspired by [awesome-android-agent-skills](https://github.com/new-silvermoon/awesome-android-agent-skills)

### `android-retrofit`
Retrofit setup for Android — service interface patterns (`@GET`, `@POST`, `@Path`, `@Query`, `@Body`), coroutines integration, OkHttp configuration, Hilt module, and error handling in the repository layer.

> Inspired by [awesome-android-agent-skills](https://github.com/new-silvermoon/awesome-android-agent-skills)

### `android-data-layer`
Data layer implementation — Repository pattern as single source of truth, Room DAOs with `Flow`, offline-first strategies (stale-while-revalidate, outbox pattern), and model mapping between DTO/entity/domain types.

> Inspired by [awesome-android-agent-skills](https://github.com/new-silvermoon/awesome-android-agent-skills)

### `coil-compose`
Image loading in Compose with Coil — `AsyncImage` vs `SubcomposeAsyncImage` vs `rememberAsyncImagePainter`, `ImageRequest` configuration, performance in lazy lists, and Hilt setup for a shared `ImageLoader`.

> Inspired by [awesome-android-agent-skills](https://github.com/new-silvermoon/awesome-android-agent-skills)

### `android-gradle-logic`
Scalable Gradle build logic — Convention Plugins, composite builds, shared `compileSdk`/`minSdk`/Compose configuration across modules, and clean per-module `build.gradle.kts` files.

> Inspired by [awesome-android-agent-skills](https://github.com/new-silvermoon/awesome-android-agent-skills)

### `gradle-build-performance`
Gradle build optimisation — Build Scans, configuration cache, build cache, kapt→KSP migration, parallel execution, lazy task configuration, and a recommended `gradle.properties` baseline.

> Inspired by [awesome-android-agent-skills](https://github.com/new-silvermoon/awesome-android-agent-skills)

## Installation

### Claude Code

Copy the skill directories into your Claude Code skills folder:

```bash
git clone https://github.com/rcosteira79/android-skills.git
cp -r android-skills/skills/* ~/.claude/skills/
```

Skills are invoked automatically based on context:

- Working on Android or KMP code → `android-dev` skill activates
- Writing or fixing tests → `android-tdd` skill activates
- Debugging Android issues → `android-debugging` skill activates
- Designing or reviewing Android UI → `android-ux` skill activates
- Working with Compose → `compose` skill activates
- Fetching Android/AndroidX source or when public docs aren't enough → `android-source-search` skill activates
- Working with coroutines → `kotlin-coroutines` skill activates
- Working with Flow/StateFlow/SharedFlow → `kotlin-flows` skill activates
- Migrating from RxJava → ask Claude to migrate, `rxjava-migration` skill activates
- Migrating XML layouts to Compose → `xml-to-compose-migration` skill activates
- Setting up networking with Retrofit → `android-retrofit` skill activates
- Implementing the data/repository layer → `android-data-layer` skill activates
- Loading images with Coil → `coil-compose` skill activates
- Setting up Gradle build logic → `android-gradle-logic` skill activates
- Optimising build performance → `gradle-build-performance` skill activates

### Cursor, Windsurf, and other agentic editors

The skills are plain markdown files. Copy the content of whichever `SKILL.md` files are relevant into your editor's context mechanism:

- **Cursor** — add to `.cursor/rules/` as `.mdc` files
- **Windsurf** — add to `.windsurf/rules/` as `.md` files
- **Copilot** — add to `.github/copilot-instructions.md`
- **Other editors** — paste into your project's custom instructions or context file

Each skill is self-contained and can be used independently.

## License

MIT
