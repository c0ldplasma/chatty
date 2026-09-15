# Chatty Modernization Plan

Bringing the codebase from Java 8 (2014) to current 2026 Java technologies.

This document tracks a fork-local modernization effort. Status markers:
`[ ]` not started, `[~]` in progress, `[x]` done.

- **Fork:** <https://github.com/c0ldplasma/chatty>
- **Upstream:** <https://github.com/chatty/chatty>
- **Baseline analysed:** `v0.29-b1` (commit `48f0c9e7`)

---

## Baseline assessment

Measured on the `v0.29-b1` baseline:

| Metric | Value |
|---|---|
| Source files | 593 `.java` |
| Total LOC | 161,362 |
| GUI package LOC | 86,469 (54%) |
| Files touching Swing/AWT | 307 (52%) |
| Test files | 45 (JUnit 4) |
| Committed dependency JARs | 9 (7.3 MB + 3.2 MB source zips) |
| CI workflows | none |

**The good news:** the baseline compiles and its tests pass unmodified on JDK 21.
There is no `sun.*` internal API use, no JAXB/Nashorn/SecurityManager dependency,
and no `--add-opens` hacks. Java 8 idioms are well adopted (840 lambdas). This is
a modernization, not a rescue.

---

## Phase 1 — Build & toolchain  ✅ done

Low risk, no application behavior change. Unblocks everything else.

- [x] Gradle wrapper 8.2.1 -> 9.7.1, and fix `build.gradle` which declared
      `wrapper { gradleVersion = '6.5.1' }` — running `gradlew wrapper` silently
      downgraded the project by two major versions.
- [x] Add `settings.gradle` (absent; project name was inferred from the directory).
- [x] Replace Gradle 9 removals that made the build self-report
      *"incompatible with Gradle 9.0"*:
  - `JavaPluginConvention` / `BasePluginConvention` / `Convention`
  - `AbstractArchiveTask.archivePath` -> `archiveFile`
  - `sourceCompatibility = 1.8` -> `java { toolchain { ... } }`
  - `archivesBaseName` -> `base { archivesName }`
- [x] Shadow plugin `com.github.johnrengelman` 8.1.1 (abandoned) ->
      `com.gradleup.shadow` 9.6.1.
- [x] Target **JDK 25 LTS** via Gradle toolchains, with Foojay auto-provisioning
      so contributors do not need a matching local JDK.
- [x] Remove the legacy Java 8 `javapackager` build path (cannot work with a
      modern toolchain); keep `jpackage` only.
- [x] Remove the `1.8.0_161`/`1.8.0_162` focus workaround in `MainGui.java` and
      the now-dead `GuiUtil.installTextComponentFocusWorkaround()`.
- [x] Fix 2x `new Long(...)` (deprecated *for removal* — will eventually stop
      compiling): `SimpleCache.java`, `SliderLongSetting.java`.
- [x] Add GitHub Actions CI (build + test) — nothing currently verifies a PR.
- [x] Update README build instructions.

### Note on the JDK choice

**JDK 25 LTS** is the target. JDK 27 (GA 2026-09-15) was evaluated first and the
build was briefly moved to it, but it is **not** an LTS release: non-LTS releases
receive roughly six months of updates.

That matters here specifically because Chatty ships a *bundled runtime* to end
users via `jpackage`. Tracking a non-LTS release would mean refreshing that
bundled JRE every six months just to stay on patched builds — an ongoing
maintenance cost with no corresponding benefit for a desktop chat client. LTS 25
is supported for years instead.

Choosing 25 also removed two pieces of incidental complexity that JDK 27
required:

- Gradle 9.7.1 (built 2026-08-19) lists Java 26 as its highest supported
  *runtime*, so JDK 27 forced a split setup: Gradle running on one JDK and
  forking the compiler for another. With 25, Gradle runs on the same JDK it
  compiles with.
- Temurin had no JDK 27 GA build yet, so toolchain auto-provisioning resolved to
  SapMachine. Temurin has JDK 25, so CI and local builds now use the same
  mainstream distribution.

Everything above is a one-line change in `build.gradle` if a future release
warrants moving again.

### Issues found during implementation

Two failures surfaced only after moving to Gradle 9; both are fixed:

1. **`shadowJar` crashed with `StackOverflowError`.** Shadow 9 already inherits
   the `jar` task's manifest, so the explicit `manifest { inheritFrom
   project.tasks.jar.manifest }` (required under Shadow 8) made the merge
   self-referential and recursed infinitely. Removing the block fixes it; the
   resulting manifest still carries `Main-Class: chatty.Chatty`.
2. **Gradle 9 changed test-class detection** and began scanning nested classes,
   so JUnit 4 failed on non-public helpers like
   `CacheBulkManagerTest$MyRequester` ("is not public", "No runnable methods").
   All 45 real test classes are named `*Test`, so `test` now sets
   `scanForTestClasses = false` with `include '**/*Test.class'`. This becomes
   unnecessary once Phase 2 migrates to JUnit 5.

### Verification

- `gradlew clean build allPlatformsZip` succeeds; **144 tests, 0 failures**.
- Bytecode major version **69** (= Java 25) confirmed via `javap`.
- The shadow JAR launches and runs on JDK 25.
- CI green on ubuntu-latest, windows-latest and macos-latest.
- `gradlew help --warning-mode all` reports **0 deprecation warnings** (the
  baseline self-reported "incompatible with Gradle 9.0").

---

## Phase 2 — Dependencies

The central problem: only three dependencies come from Maven Central. Nine JARs
are committed into `assets/lib/` and pulled in with `fileTree`, which means no
transitive resolution, no version metadata, no CVE scanning, and no way for
Dependabot or Renovate to see them.

- [ ] Introduce a version catalog (`gradle/libs.versions.toml`).
- [ ] Move committed JARs to Maven Central coordinates.
- [ ] Drop `httpclient5`/`httpcore5` entirely by migrating the four
      `HttpURLConnection` users to `java.net.http.HttpClient` (in the JDK since 11).
- [ ] Replace `txtmark` 0.13 (abandoned ~2015, single-file usage in `Changes.java`)
      with `commonmark-java`.
- [ ] JUnit 4.12 (2014) -> JUnit 6 via `junit-bom`, with the vintage engine so the
      45 existing test files keep running while new tests use Jupiter.
- [ ] Enable Dependabot.

### Patched artifacts

Two JARs are **locally patched** builds (note the `-2`/`-3` suffixes) and cannot
simply be bumped:

- **JTattoo 1.6.12-3** — resolved by Phase 3's FlatLaf consolidation, which
  removes the dependency altogether.
- **jkeymaster 1.3-2** — upstream is stale (latest tag `jkeymaster-1.3`). Rebase
  the patch and either publish it or vendor the patched sources into `src/`.

### Deferred: JSON

`json-simple` 1.1.1 has been unmaintained since **2012** and is the app's JSON
backbone — used in **55 source files**, covering the whole Twitch API layer and
all EventSub payload parsing, with raw `Map`/`JSONObject` and unchecked casts.

Replacing it with Jackson or Gson would give type-safe binding and let the ~50
EventSub payload classes become annotated records. This is weeks of work, not
days, and is tracked separately as **Phase 5**. `json-simple` is dead, but it is
not broken — this is the one item worth deferring.

---

## Phase 3 — Swing modernization

### Decision: keep Swing

The GUI is not a swappable layer. 86,469 of 161,362 LOC (54%) live in
`src/chatty/gui`, and 307 of 593 files touch `javax.swing` or `java.awt`:

- **`ChannelTextPane.java` is 4,718 lines** built on `javax.swing.text`, with
  custom `MyEditorKit`, `MyIconView`, `WrapLabelView` and `MyParagraphView`
  subclasses. This is the actual product: scrollback with inline animated emotes,
  per-message styling, selection, link hit-testing, timeout strikethrough.
  44 files touch `javax.swing.text` in total.
- **`util/dnd` is a 5,063-line custom docking framework** (tear-off channel tabs).
- **The settings dialogs are 20,660 LOC** across 62 `JDialog` subclasses;
  143 custom component subclasses overall.

Alternatives were considered and rejected:

- **JavaFX** was removed from the JDK at Java 11 and is now a separate OpenJFX
  dependency, so migrating would *add* a dependency and platform-specific native
  bundles rather than remove any. `TextFlow` has no equivalent to
  `StyledDocument`'s element model; the 4,718-line pane would be rebuilt from
  scratch on a weaker foundation.
- **Compose Multiplatform** would mean adopting Kotlin as a second language plus a
  full rewrite, with a larger Skia-based runtime.
- **SWT** trades a pure-Java JAR for platform natives, directly worsening the
  `jpackage` story.

Swing is not deprecated and remains maintained in the JDK. For a desktop chat
client whose differentiator *is* its custom text rendering, no alternative clears
the bar.

### Work

- [ ] **Consolidate on FlatLaf 3.7.2, drop JTattoo.** `LaF.java` currently offers
      nine JTattoo themes (`hifi`, `hifi2`, `hifiCustom`, `mint`, `noire`,
      `graphite`, `fast`, `aero`, `luna`) alongside two FlatLaf ones. Dropping
      JTattoo removes a patched JAR and 1.2 MB of binaries from git.
- [ ] **Theme settings migration** (user-visible). Map removed theme codes to
      FlatLaf equivalents: `hifi`/`hifi2`/`noire`/`graphite` -> dark variants,
      `mint`/`aero`/`luna`/`fast` -> light. `hifiCustom` feeds user-chosen colors
      into JTattoo theme properties via `customColors()`; FlatLaf takes overrides
      through `UIManager` keys, so it is portable but not mechanical.
- [ ] **HiDPI.** `TwitchClient.java` sets `sun.java2d.uiScale` from a manual
      percentage setting whose own tooltip admits *"Settings higher than 1.0 can
      cause blurry images. May not work at all in some circumstances."* That is a
      Java 8 limitation. Java 9+ has per-monitor HiDPI and FlatLaf's `UIScale`
      handles fractional scaling and scales icons/borders properly, so this should
      become automatic detection with an optional override.
- [ ] **Re-test the Java 8-era Windows rendering workarounds.** `sun.java2d.d3d`
      and `sun.java2d.noddraw` are forced off in `TwitchClient.java` *and*
      hardcoded into the `jpackage` launcher args. Measure whether they are still
      needed on a modern JDK rather than carrying them forward by inertia.
- [ ] Keep `util/dnd` and `ChannelTextPane` as-is — no Swing equivalent exists.

---

## Phase 4 — Language level

Once the toolchain is on JDK 25, sweep the code for idioms that are now 10+ years
behind:

- [ ] `record` for data holders — 726 `public final` fields, and the ~50 EventSub
      payload classes especially.
- [ ] Pattern matching for `instanceof` — 241 `instanceof X)` + cast sites.
- [ ] Switch expressions — 146 `switch` statements.
- [ ] Text blocks — 92 `"<html>"` string concatenations.
- [ ] `var` for obvious locals.
- [ ] `java.time` consistently — currently `Date`/`SimpleDateFormat`/`Calendar` in
      18 files vs. `java.time` in 10.

### Explicitly out of scope: JPMS

`jlink`/module descriptors are **not** worth it here. Swing plus JNA plus
reflection-heavy Look-and-Feel code makes JPMS painful for little gain, and
`jpackage` already produces a trimmed runtime without them.

---

## Phase 5 — JSON migration

Deferred from Phase 2. Replace `json-simple` across 55 files with Jackson (or
Gson), converting EventSub payloads to annotated records. Tracked separately
because of its size and risk profile.

---

## Version summary

| Area | Baseline | Target |
|---|---|---|
| Java language level | 8 (2014) | **25 LTS** |
| Build JDK | Java 8 JDK | JDK 25 LTS toolchain |
| Gradle | 8.2.1 wrapper / **6.5.1 declared** | **9.7.1**, consistent |
| Build script style | `Convention` APIs, `archivePath` | `java{}`/`base{}`, `archiveFile` |
| `settings.gradle` | absent | present, with Foojay resolver |
| Shadow plugin | `com.github.johnrengelman` 8.1.1 (abandoned) | `com.gradleup.shadow` 9.6.1 |
| `org.gradle.crypto.checksum` | 1.4.0 | 1.4.0 (already current) |
| Dependency management | 9 JARs committed (7.3 MB) | Version catalog -> Maven Central |
| CI | none | GitHub Actions, build + test |
| JNA / jna-platform | 5.12.1 | 5.19.1 |
| FlatLaf | 3.2.5 | 3.7.2 |
| commons-codec | 1.15 | 1.22.1 |
| httpclient5 / httpcore5 | 5.1 / 5.1.1 | **removed** (`java.net.http`) |
| Java-WebSocket | 1.4.0 | 1.6.0 |
| slf4j-api / slf4j-jdk14 | 1.7.29 / **1.7.9** (mismatched) | 2.0.19 both |
| txtmark | 0.13 (abandoned 2015) | commonmark-java |
| json-simple | 1.1.1 (abandoned 2012) | Jackson / Gson (Phase 5) |
| JTattoo | 1.6.12-3 (patched) | **removed** (FlatLaf only) |
| jkeymaster | 1.3-2 (patched) | Rebase patch, publish or vendor |
| Testing | JUnit 4.12 (2014) | JUnit 6 + vintage engine |
| HTTP client code | `HttpURLConnection` x4 | `java.net.http.HttpClient` |
| UI toolkit | Swing/AWT | **Swing** — kept deliberately |
| Look & Feel | JTattoo (9) + FlatLaf (2) + Nimbus/Metal | FlatLaf + system |
| HiDPI | Manual `uiScale`, documented unreliable | Automatic, with override |
| `sun.java2d.d3d` / `noddraw` | Forced off in code and launcher | Re-test, drop if obsolete |
| Windows packaging | `javapackager` + `jpackage` | `jpackage` only |
| Modularization (JPMS) | none | none (deliberate) |
