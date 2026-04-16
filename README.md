# Android Skills

Skills for Android and KMP development — covering architecture, data layer, networking, testing, debugging, Jetpack Compose, coroutines, flows, Gradle, and RxJava migration. Works as a plugin for Claude Code and Copilot CLI.

Several skills in this collection were inspired by or built on top of work from the community — specifically [awesome-android-agent-skills](https://github.com/new-silvermoon/awesome-android-agent-skills) and [compose-skill](https://github.com/aldefy/compose-skill). Skills with no attribution tag are original work.

## Installation

### Claude Code (plugin)

Install as a plugin to get all skills:

```
/plugin marketplace add rcosteira79/android-skills
/plugin install android-skills@android-skills
```

Updates are picked up automatically when the plugin version is bumped.

#### MCP server (optional)

For enhanced source code navigation (local source sync, Tree-sitter parsing, class hierarchy, LSP), install [android-source-explorer-mcp](https://github.com/mrmike/android-source-explorer-mcp) separately:

```bash
uv tool install git+https://github.com/mrmike/android-source-explorer-mcp
```

The `android-source-search` skill will automatically use the MCP tools if available, and falls back to Gitiles/GitHub otherwise.

#### Android CLI (recommended)

Several skills — notably [`compose`](plugins/android-skills/skills/compose/SKILL.md) and [`android-debugging`](plugins/android-skills/skills/android-debugging/SKILL.md) — have CLI-native shortcuts when Google's `android` CLI is installed: documentation search over the Android Knowledge Base (`android docs`), runtime UI layout inspection (`android layout`), device/emulator orchestration, and SDK management. The skills still work without it, but recommend installing it for the best experience.

Follow Google's installation instructions: <https://developer.android.com/tools/agents/android-cli>

### Claude Code (manual)

Alternatively, copy the skill directories into your Claude Code skills folder:

```bash
git clone https://github.com/rcosteira79/android-skills.git
cp -r android-skills/plugins/android-skills/skills/* ~/.claude/skills/
```

Note: the manual approach does not include the MCP server. See the optional MCP section above.

### Copilot CLI (plugin)

Copilot CLI detects the same plugin format automatically:

```bash
copilot plugin install rcosteira79/android-skills
```

### Other agentic editors

Each skill is self-contained — the `SKILL.md` files are plain markdown, so they can be adapted to other editors that support custom instructions.

## Skills

Skills are invoked automatically based on context (e.g. working on Compose code activates the `compose` skill).

### `android-dev`
Senior Android engineering knowledge and best practices for Android and KMP projects. Covers architecture, code quality, and platform-specific patterns.

### `android-tdd`
Test-driven development for Android/KMP — extends TDD with Android's three-tier test model, fake-first strategy, coroutine testing, Compose UI testing, and Roborazzi screenshot testing.

### `android-ux`
Material Design 3 UX principles for Android — touch targets (48×48dp), 8dp spacing grid, navigation patterns (Bottom Bar, Rail, Drawer), canonical layouts (Feed, List-Detail, Supporting Pane), foldable postures (tabletop, book mode), M3 contrast levels, safe area handling, accessibility, animation timing, keyboard input types, and an M3 compliance audit that scores screens across 10 categories.

> M3 contrast levels, canonical layouts, and foldable posture patterns inspired by [material-3-skill](https://github.com/hamen/material-3-skill)

### `android-debugging`
Debugging Android and KMP issues — Logcat, ADB, ANR traces, R8 stack trace decoding, memory leaks, Gradle build failures, and Compose recomposition bugs.

### `android-source-search`
Fetch and verify Android source code — AOSP platform internals (`@hide` APIs, framework classes, system services via Gitiles) and AndroidX/Jetpack library source and samples (via GitHub). Also useful when public docs are insufficient to complete a task.

> This skill is a zero-setup fallback. For enhanced capabilities (local source sync, sub-10ms Tree-sitter parsing, method-level extraction, class hierarchy, LSP), install [android-source-explorer-mcp](https://github.com/mrmike/android-source-explorer-mcp) separately — the skill will use it automatically when available.

### `kotlin-coroutines`
Dispatcher selection, scope management, structured concurrency, cancellation, exception handling, and Android/KMP async patterns. Includes the DispatcherProvider pattern for testable dispatcher injection.

> Incorporates material from [awesome-android-agent-skills](https://github.com/new-silvermoon/awesome-android-agent-skills)

### `kotlin-flows`
Flow type selection (`Flow`/`StateFlow`/`SharedFlow`), operator chains, callback bridging, lifecycle-safe collection, Channel migration, and UI state management.

### `compose`
Jetpack Compose expert guidance — state management (`@Composable`, `remember`, `mutableStateOf`, `derivedStateOf`, state hoisting), Modifier chains, lazy lists, navigation, animation, side effects, theming, accessibility, and performance optimization.

> Forked from [compose-skill](https://github.com/aldefy/compose-skill). The reference docs share the same foundation, but this version replaces the bundled static AndroidX source snapshots with live source verification via [android-source-explorer-mcp](https://github.com/mrmike/android-source-explorer-mcp) (preferred) or the `android-source-search` skill — always up to date, zero context overhead.

### `rxjava-migration`
Triggered only when you explicitly ask to migrate. Assesses complexity, maps RxJava types and operators to coroutines equivalents, and provides interop patterns for incremental migration.

> Incorporates material from [awesome-android-agent-skills](https://github.com/new-silvermoon/awesome-android-agent-skills)

### `xml-to-compose-migration`
Migrate Android XML layouts to Jetpack Compose — layout mapping tables (RecyclerView → LazyColumn, LinearLayout → Column/Row, etc.), attribute mapping, state migration from LiveData/ViewBinding, and incremental adoption via `ComposeView`.

> Incorporates material from [awesome-android-agent-skills](https://github.com/new-silvermoon/awesome-android-agent-skills)

### `android-retrofit`
Retrofit setup for Android — service interface patterns (`@GET`, `@POST`, `@Path`, `@Query`, `@Body`), coroutines integration, OkHttp configuration, Hilt module, and error handling in the repository layer.

> Inspired by [awesome-android-agent-skills](https://github.com/new-silvermoon/awesome-android-agent-skills)

### `kmp-ktor`
Ktor client setup for KMP and Android — per-platform engine selection (OkHttp/Darwin/CIO), `kotlinx.serialization` configuration, bearer token auth with refresh via the `Auth` plugin, `MockEngine` testing, and error mapping at the repository boundary.

> Inspired by [compose-skill](https://github.com/Meet-Miyani/compose-skill) (Meet-Miyani)

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

## Related Projects

- [android/skills](https://github.com/android/skills) — Google's official Android agent skills covering AGP 9 migration, XML-to-Compose migration, Navigation 3, R8 analysis, Play Billing upgrades, and edge-to-edge support
- [awesome-android-agent-skills](https://github.com/new-silvermoon/awesome-android-agent-skills) — curated list of Android agent skills that inspired many of the skills in this repo
- [compose-skill](https://github.com/aldefy/compose-skill) — alternative Compose skill that bundles a static AndroidX snapshot
- [android-source-explorer-mcp](https://github.com/mrmike/android-source-explorer-mcp) — MCP server for navigating Android source code (optional, used by `android-source-search` skill)
- [material-3-skill](https://github.com/hamen/material-3-skill) — comprehensive Material Design 3 reference covering 30+ components, design tokens, theming, responsive layout, and dynamic color across platforms
- [compose_skill](https://github.com/hamen/compose_skill) — strict, evidence-based Compose audit skill that scores repos on performance, state management, side effects, and API quality using automated Compose Compiler reports
- [compose-skill (Meet-Miyani)](https://github.com/Meet-Miyani/compose-skill) — broad Compose + KMP skill covering MVI/MVVM, Navigation 2 & 3, Ktor, Koin/Hilt, Room, DataStore, Paging, and iOS interop; inspired the `kmp-ktor` skill in this repo
- [android-reverse-engineering-skill](https://github.com/SimoneAvogadro/android-reverse-engineering-skill) — Claude Code plugin for decompiling APKs, extracting API endpoints, and tracing call flows

## License

MIT
