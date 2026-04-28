# Core Terminal — Hexagonal Foundation (Plan #1) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Refactor `core/terminal/` from a flat layout (where `TermLabTerminalVirtualFile` and `TermLabTerminalEditor` mix five concerns each) into a strict hexagonal layout (`domain/`, `application/`, `ports/`, `adapters/{intellij,jediterm}/`) with ArchUnit-enforced dependency rules. Plugins stay disabled across this whole plan and Plan #2; manual smoke testing is end-of-plan only.

**Architecture:** Approach B (big-bang per file). Each task dismantles one file or builds one cohesive unit; intermediate states between tasks may be temporarily broken. Pattern documentation per step + memory write-back per phase prevents drift. Test-driven development for `domain/` and `application/` (plain JUnit, no IntelliJ test framework). Adapter classes get manual verification — IntelliJ light-platform tests are deferred.

**Tech Stack:** Java 21, Bazel, IntelliJ Platform (monolith layout pinned via `/Users/dustin/projects/intellij-community`), JediTerm, JUnit 5, ArchUnit (new dependency, added in Task 4).

**Spec:** `docs/superpowers/specs/2026-04-28-core-terminal-hexagonal-foundation-design.md`

**Architecture review source:** `docs/termlab-clean-architecture-plan.md`

---

## Orientation for the Implementing Engineer

Before touching any code, read these files to understand the conventions you'll follow:

- `core/src/com/termlab/core/terminal/TermLabTerminalVirtualFile.java` — the 299-line god class this plan dismantles. Read end-to-end. Note: identity (`sessionId`), provider reference, CWD, title, manual-override flag, session context, and the entire `SharedTerminalSession` inner class are all current responsibilities and *all* move out by the end of Task 15.
- `core/src/com/termlab/core/terminal/TermLabTerminalEditor.java` — the 326-line editor. Step 4 (Tasks 16–22) extracts five controllers from this.
- `sdk/src/com/termlab/sdk/TerminalSessionProvider.java` — the existing SPI. Task 10 adds a non-breaking default-method `capabilities()`.
- `core/BUILD.bazel` — current core build target. Task 4 modifies this to add ArchUnit.
- `BUILD.bazel` (repo root) — `termlab_run` target. Task 1 modifies this to disable plugins.
- `core/test/com/termlab/core/terminal/LocalPtySessionProviderTest.java` — example of an existing core terminal test (plain JUnit; no IntelliJ test framework needed). Mirror this convention.
- `plugins/vault/BUILD.bazel` — reference for the `*_test_lib` pattern (compile-only test targets in this codebase). The pattern note is in the comment: "Executable test target can be added later once the TermLab tree has a pattern for running pure-JUnit5 tests via Bazel."

**Build & run commands** (from repo root):

- Build everything: `bash bazel.cmd build //termlab/...`
- Build core only: `bash bazel.cmd build //termlab/core:core`
- Compile-check tests: `bash bazel.cmd build //termlab/core:core_test_lib`
- Run TermLab from source: `bash bazel.cmd run //termlab:termlab_run`

**Test-running convention:** the user runs tests via IntelliJ. Each task that adds a test has a step "Run via IntelliJ → expected output." A Bazel `core_test_runner` target is *not* added by this plan (CI test infrastructure stays unchanged per user constraint).

**Commit convention:** lowercase prefix (`refactor(core):`, `feat(core):`, `test(core):`, `docs:`); short summary; co-authorship trailer. Each task ends with a commit step. Frequent commits — every TDD cycle is a commit.

**Memory directory:** `/Users/dustin/.claude/projects/-Users-dustin-projects-TermLab/memory/`. Pattern documentation memory writes happen at the end of Step 3 (Task 15) and Step 4 (Task 22).

**Branch name:** This plan assumes the branch is named `core-rework`. Task 1 creates it. If you prefer a different name, change it in Task 1 only.

---

## Pragmatic deviations from the spec

1. **No Bazel `core_test_runner` target.** The spec notes "tests run via IntelliJ as today." We add `core_test_lib` dependencies (ArchUnit) but no `java_binary` runner. CI behavior is unchanged. Tests are runnable from IntelliJ via JUnit run configurations.

2. **Runtime state lives on `TerminalSessionHandle`, not on `TerminalSessionDescriptor`.** The spec frames the descriptor as the immutable "opening intent." Current CWD and current title (mutated by OSC 7 / OSC 0/2 sequences) are runtime state — they live on `TerminalSessionHandle`'s implementation, queried by `TermLabEditorTabTitleProvider`. The descriptor never mutates after creation. This is implicit in the spec but worth flagging because it determines where state lives.

3. **`TermLabTerminalVirtualFile` survives Plan #1 as a slim adapter.** It is *not* deleted; it shrinks from 299 lines to ~25. It still implements `LightVirtualFile` so `FileEditorManager` can route it.

4. **`TermLabMultiExecManager` is left untouched in Plan #1.** Plan #2 deletes it. Until then it remains in `core/src/com/termlab/core/terminal/` and is non-functional because plugins (notably SSH) are disabled — `SSH_PROVIDER_ID` is never registered, so MultiExec's hardcoded check finds no targets. Don't fix MultiExec to use new APIs; it's getting deleted.

5. **`JediTermSessionFactory` and `TerminalWidgetFactory` live in `adapters/intellij/`, not `adapters/jediterm/`.** The spec's component table places `JediTermSessionFactory` in `adapters/jediterm/`, but the existing `TermLabTerminalWidget` requires an IntelliJ `Project`, which would force `adapters/jediterm/` to import `com.intellij.*` and break the rule. The compromise: factories that need `Project` live on the IntelliJ side; the JediTerm side owns the handle (`JediTermSessionHandle`) and the OSC connector. ArchUnit accepts this — `adapters/intellij/` is allowed to import IntelliJ.

6. **`TerminalSessionHandle` exposes `awaitExit()` instead of leaking `TtyConnector`.** The spec describes the handle in terms of write/resize/onExit/dispose. Implementing the exit-watcher cleanly required adding `int awaitExit() throws InterruptedException` so `TerminalSessionMonitor` (in `adapters/intellij/`) doesn't need to import `com.jediterm.terminal.TtyConnector`. The handle's internal `connector()` accessor is kept package-private inside `adapters/jediterm/`.

---

## Task 1: Create branch, disable plugins, verify boot

**Files:**
- Modify: `BUILD.bazel:54-71` (comment out plugin runtime_deps)

- [ ] **Step 1: Confirm current branch is `main` and tree is clean**

```bash
git status
git branch --show-current
```

Expected: `main`, "nothing to commit, working tree clean" (the spec commits already landed).

- [ ] **Step 2: Create the rework branch**

```bash
git checkout -b core-rework
```

Expected: "Switched to a new branch 'core-rework'".

- [ ] **Step 3: Disable all non-core plugins in root `BUILD.bazel`**

Open `BUILD.bazel` (repo root). Locate the `termlab_run` `runtime_deps` block at lines 41–84. Comment out every `//termlab/plugins/*` entry, every IntelliJ-platform plugin entry except `classic-ui`, and the `credentialStore-impl` entry (vault dependency). The block becomes:

```bazel
java_binary(
    name = "termlab_run",
    runtime_deps = [
        # Platform core runtime (no product plugins)
        "//platform/main/intellij.platform.monolith.main:monolith-main",
        "//platform/boot",
        "//platform/bootstrap",
        "//platform/starter",
        "//platform/tips-of-the-day:tips",

        # TermLab modules
        "//termlab/customization",
        "//termlab/core",
        "//termlab/sdk",

        # CORE-REWORK: TermLab bundled plugins disabled until core rework completes.
        # Re-enable in Plan #2 Step 3 (the verification phase). MultiExec stays
        # disabled past Plan #2 — it's reintroduced in plugins/ssh in a follow-up.
        # "//termlab/plugins/vault",
        # "//termlab/plugins/ssh",
        # "//termlab/plugins/tunnels",
        # "//termlab/plugins/share",
        # "//termlab/plugins/sftp",
        # "//termlab/plugins/search",
        # "//termlab/plugins/sysinfo",
        # "//termlab/plugins/proxmox",
        # "//termlab/plugins/editor",
        # "//termlab/plugins/runner",
        # "//platform/credential-store-impl:credentialStore-impl",

        # IntelliJ-platform bundled plugins — ONLY classic-ui during core rework
        "//plugins/classic-ui",
        # CORE-REWORK: TextMate disabled — only consumed by the editor plugin.
        # "//plugins/textmate/plugin",
        # "//plugins/textmate:textmate",
        # "//plugins/textmate/common:common",
        # "//plugins/textmate/backend:backend",
    ],
    main_class = "com.intellij.idea.Main",
    jvm_flags = [
        # ... unchanged ...
    ],
)
```

Leave the `jvm_flags` block exactly as-is. Leave the `termlab-main` target (lines 8–23) alone — that's the JPS roll-up, not the run binary.

- [ ] **Step 4: Build to confirm the run target still resolves**

```bash
bash bazel.cmd build //termlab:termlab_run
```

Expected: BUILD SUCCESS. If anything in `core/` references a disabled plugin (it shouldn't — core can't depend on plugins), the build fails here. Investigate; do not work around by re-enabling a plugin.

- [ ] **Step 5: Run TermLab and verify boot**

```bash
bash bazel.cmd run //termlab:termlab_run
```

Manually verify:
1. App launches.
2. Welcome screen / "open new terminal" entry point is reachable.
3. Opening a new terminal succeeds and shows a working local PTY (your shell prompt appears).
4. No "Hosts" / "Vault" / "SFTP" tool windows are present.
5. No errors in the IDE log relating to missing extension points (open Help → Show Log).

Quit the app.

- [ ] **Step 6: Commit**

```bash
git add BUILD.bazel
git commit -m "$(cat <<'EOF'
refactor(core): disable non-core plugins for core-rework branch

CORE-REWORK Plan #1 Task 1. Plugins stay disabled across Plan #1 and
Plan #2; re-enabled in Plan #2 Step 3 verification phase.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 2: Scaffold the hexagonal package skeleton

**Pattern:** Package-level Information Hiding. Each package gets a `package-info.java` declaring its role and import constraints. The packages are empty in this task; subsequent tasks fill them.

**Why:** ArchUnit rules in Task 4 reference package paths. Creating empty packages first lets Task 4's rules compile and pass trivially before Task 5 starts moving code in.

**Files:**
- Create: `core/src/com/termlab/core/terminal/domain/package-info.java`
- Create: `core/src/com/termlab/core/terminal/application/package-info.java`
- Create: `core/src/com/termlab/core/terminal/ports/package-info.java`
- Create: `core/src/com/termlab/core/terminal/adapters/intellij/package-info.java`
- Create: `core/src/com/termlab/core/terminal/adapters/jediterm/package-info.java`

- [ ] **Step 1: Create `core/src/com/termlab/core/terminal/domain/package-info.java`**

```java
/**
 * Pure domain types for the TermLab terminal bounded context.
 *
 * <p>Allowed imports: JDK only, plus value types from {@code com.termlab.sdk}.
 * Forbidden imports: {@code com.intellij.*}, {@code javax.swing.*},
 * {@code com.jediterm.*}, {@code org.apache.sshd.*}.
 *
 * <p>Enforced by {@code HexagonalBoundariesTest}.
 */
package com.termlab.core.terminal.domain;
```

- [ ] **Step 2: Create `core/src/com/termlab/core/terminal/application/package-info.java`**

```java
/**
 * Application services and use cases for the TermLab terminal bounded context.
 *
 * <p>Orchestrates domain types via ports. Holds no infrastructure references.
 *
 * <p>Allowed imports: {@code com.termlab.core.terminal.domain},
 * {@code com.termlab.core.terminal.ports}, JDK.
 * Forbidden: {@code com.intellij.*}, {@code javax.swing.*}, {@code com.jediterm.*}.
 *
 * <p>Enforced by {@code HexagonalBoundariesTest}.
 */
package com.termlab.core.terminal.application;
```

- [ ] **Step 3: Create `core/src/com/termlab/core/terminal/ports/package-info.java`**

```java
/**
 * Port interfaces for the TermLab terminal bounded context.
 *
 * <p>Pure interfaces — no fields, no instance state, no default methods carrying
 * logic. Implemented by {@code adapters/}.
 *
 * <p>Allowed imports: {@code com.termlab.core.terminal.domain}, JDK.
 * Forbidden: everything else.
 *
 * <p>Enforced by {@code HexagonalBoundariesTest}.
 */
package com.termlab.core.terminal.ports;
```

- [ ] **Step 4: Create `core/src/com/termlab/core/terminal/adapters/intellij/package-info.java`**

```java
/**
 * IntelliJ Platform adapters (anti-corruption layer) for the TermLab terminal
 * bounded context.
 *
 * <p>Allowed imports: {@code com.termlab.core.terminal.{domain,application,ports}},
 * {@code com.intellij.*}, {@code javax.swing.*}, plus the *single* allowed
 * cross-adapter touchpoint: {@code com.termlab.core.terminal.adapters.jediterm.JediTermSessionHandle}
 * (used to access the embedded JediTerm widget for IntelliJ FileEditor mounting).
 * Forbidden direct imports: {@code com.jediterm.*} — route through the JediTerm adapter.
 *
 * <p>Enforced by {@code HexagonalBoundariesTest}.
 */
package com.termlab.core.terminal.adapters.intellij;
```

- [ ] **Step 5: Create `core/src/com/termlab/core/terminal/adapters/jediterm/package-info.java`**

```java
/**
 * JediTerm adapters for the TermLab terminal bounded context.
 *
 * <p>Allowed imports: {@code com.termlab.core.terminal.{domain,application,ports}},
 * {@code com.jediterm.*}, {@code javax.swing.*} (only for {@code JComponent}
 * exposure to {@code adapters.intellij}). Forbidden: {@code com.intellij.*}.
 *
 * <p>Enforced by {@code HexagonalBoundariesTest}.
 */
package com.termlab.core.terminal.adapters.jediterm;
```

- [ ] **Step 6: Build to confirm the new packages compile**

```bash
bash bazel.cmd build //termlab/core:core
```

Expected: BUILD SUCCESS. New `package-info.java` files contribute no code; the build is unchanged.

- [ ] **Step 7: Commit**

```bash
git add core/src/com/termlab/core/terminal/domain/ \
        core/src/com/termlab/core/terminal/application/ \
        core/src/com/termlab/core/terminal/ports/ \
        core/src/com/termlab/core/terminal/adapters/
git commit -m "$(cat <<'EOF'
refactor(core): scaffold hexagonal package skeleton for terminal

Plan #1 Task 2. Empty domain/application/ports/adapters packages with
package-info.java declaring import constraints. Packages will be
populated by subsequent tasks; ArchUnit rules in Task 4 lock them in.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 3: Write `docs/architecture.md` guardrails document

**Pattern:** Documentation-as-Contract. A single canonical reference for the dependency rules ArchUnit will enforce. New contributors and future plugin work copy-paste this layout.

**Files:**
- Create: `docs/architecture.md`

- [ ] **Step 1: Create `docs/architecture.md`**

```markdown
# TermLab Architecture Rules

This document is the canonical reference for TermLab's hexagonal architecture.
ArchUnit tests in `core/test/com/termlab/core/terminal/architecture/` enforce
these rules mechanically.

## Bounded contexts

TermLab's first-party code is split into bounded contexts. Each owns its own
hexagonal layout (`domain/`, `application/`, `ports/`, `adapters/`) and may
depend on other contexts only through their `ports/` and `domain/` packages.

Current contexts:

- `core/terminal/` — terminal sessions, providers, lifecycle (this plan)
- `core/workspace/` — workspace persistence (Plan #2)
- `plugins/ssh/` — SSH provider, host catalog, MultiExec (future plan)
- `plugins/vault/` — encrypted credential storage (future plan)
- `plugins/sftp/` — SFTP file transfer (future plan)

## Dependency rules

Inside any bounded context:

| Package | May import | May NOT import |
|---|---|---|
| `domain/` | JDK, `sdk/` value types | `com.intellij.*`, `javax.swing.*`, `com.jediterm.*`, `org.apache.sshd.*` |
| `application/` | `domain/`, `ports/` | same as above |
| `ports/` | `domain/` (interfaces only — no fields, no instance state, no default methods carrying logic) | everything else |
| `adapters/intellij/` | `application/`, `domain/`, `ports/`, `com.intellij.*`, `javax.swing.*`, plus the *single* cross-adapter touchpoint listed below | `com.jediterm.*` directly |
| `adapters/jediterm/` | `application/`, `domain/`, `ports/`, `com.jediterm.*`, `javax.swing.*` | `com.intellij.*` |

The dependency arrow points strictly inward. `domain/` has no imports out.

## Adapter cross-dependency (one allowed exception)

`adapters/intellij/` must embed a JediTerm widget into IntelliJ's
`FileEditor.getComponent()`. The widget is a `com.jediterm.*` type, which
`adapters/intellij/` is forbidden from importing directly. Resolution:
`adapters/jediterm/JediTermSessionHandle` exposes a `swingComponent(): JComponent`
accessor. `adapters/intellij/` imports `JediTermSessionHandle` (our class) and
calls that accessor. The IntelliJ adapter sees only Swing.

ArchUnit explicitly permits `adapters/intellij/` to reference
`adapters/jediterm.JediTermSessionHandle` and forbids any other
`adapters/jediterm/*` import from `adapters/intellij/*`.

## Pattern catalog

Patterns deliberately used in TermLab core:

| Pattern | Where it fits |
|---|---|
| Value Object | `TerminalSessionId`, `TerminalProviderId`, `TerminalSessionDescriptor`, `TerminalCapabilities` |
| Application Service / Use Case | `TerminalTabService`, `OpenTerminalUseCase`, `CloseTerminalUseCase` |
| Registry | `TerminalSessionProviderRegistry` |
| Factory | `TerminalSessionFactory`, `TerminalWidgetFactory` |
| Port | `TerminalSessionHandle`, `TerminalEventPublisher`, `TerminalNotificationPort` |
| Adapter / Anti-Corruption Layer | `TermLabTerminalVirtualFile`, `TermLabTerminalEditor`, `JediTermSessionHandle` |
| Observer | `TerminalSessionMonitor` (watches `TtyConnector` for exit) |
| Strategy | `TerminalSessionProvider` implementations selected by id |
| Result Object | `Result<T, E>` for expected failures |

Patterns deliberately *not* used in TermLab core:

- CQRS
- Event sourcing
- Abstract factories for every small object
- Generic "manager" classes
- Global service locators in domain/application code

## Where IntelliJ APIs are allowed

- `core/terminal/adapters/intellij/` — yes
- `core/customization/` — yes
- Existing IntelliJ-customization classes at the root of `core/` (theme strippers, etc.) — yes (legacy)
- Anywhere else — no

## Where Swing APIs are allowed

- `core/terminal/adapters/intellij/` — yes
- `core/terminal/adapters/jediterm/` — yes (only for `JComponent` exposure)
- `core/terminal/{domain,application,ports}/` — no

## Where JediTerm APIs are allowed

- `core/terminal/adapters/jediterm/` — yes
- `core/terminal/adapters/intellij/` — no, except the `JediTermSessionHandle` cross-touchpoint

## Where Apache MINA / SSHD APIs are allowed

- `plugins/ssh/adapters/mina/` (future) — yes
- Anywhere else — no

## Testing rules

- `domain/` and `application/` tests use plain JUnit 5 — no IntelliJ test framework.
- `adapters/intellij/` tests, when present, use IntelliJ's `LightPlatformTestCase` /
  `BasePlatformTestCase`. Plan #1 defers these.
- ArchUnit boundary tests live in `core/test/com/termlab/core/terminal/architecture/`.
- New tests follow the existing `core/test/com/termlab/core/...` mirroring convention.
- Tests run via IntelliJ's JUnit runner. Bazel `*_test_lib` targets compile-check
  only; no `*_test_runner` is added by the core rework.

## Why hexagonal here

TermLab is permanently built on the IntelliJ Platform. We are *not* using
hexagonal for portability. The four real payoffs:

1. **Testability** — `domain/` and `application/` are unit-testable with plain
   JUnit + JDK. No IntelliJ test framework spin-up.
2. **Plugin SPI clarity** — `ports/` is the literal SPI surface. Plugin
   authors depend on `application/` and `ports/`, not on IntelliJ surface area.
3. **Drift prevention** — the dependency rule mechanically prevents IntelliJ
   leakage into pure logic. Visible imports become visible violations.
4. **Anti-corruption layer for IntelliJ Platform churn** — IntelliJ breaks
   APIs across major versions; isolating those imports to `adapters/intellij/`
   contains the blast radius.
```

- [ ] **Step 2: Commit**

```bash
git add docs/architecture.md
git commit -m "$(cat <<'EOF'
docs: add architecture guardrails for TermLab core rework

Plan #1 Task 3. Canonical reference for hexagonal dependency rules.
ArchUnit tests in Task 4 enforce these rules mechanically.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 4: Add ArchUnit dependency and write boundary tests

**Pattern:** Architecture Conformance Test. Mechanically verifies dependency rules on every test run.

**Why:** the user's primary concern is drift. Code review alone is insufficient — ArchUnit catches violations at build/test time, not at PR time. Tests trivially pass at the end of this task because no code lives in the new packages yet (`package-info.java` files don't count); they begin to enforce the moment Task 5 places types in `domain/`.

**How it works:** ArchUnit's `DescribedPredicate` / `ArchRule` API. Each rule is a one-liner like `noClasses().that().resideInAPackage("..domain..").should().dependOnClassesThat().resideInAnyPackage("com.intellij..")`. Failing rules produce readable messages naming offending classes and imports.

**Files:**
- Modify: `core/BUILD.bazel:45-61` (add ArchUnit deps to `core_test_lib`)
- Create: `core/test/com/termlab/core/terminal/architecture/HexagonalBoundariesTest.java`

- [ ] **Step 1: Add ArchUnit to `core/BUILD.bazel` test_lib deps**

ArchUnit is published to Maven Central as `com.tngtech.archunit:archunit-junit5`. The `intellij-community` Bazel tree exposes Maven coordinates via the `@lib//` repository. Find the existing `@lib//:` entries in this BUILD or in plugin BUILDs (e.g., `@lib//:gson`) for the syntax. ArchUnit may not yet be in `@lib//`; if not, follow the IntelliJ-community `MAVEN_DEPS` pattern.

Inspect `intellij-community/lib/BUILD.bazel` (via `$INTELLIJ_ROOT`) for the registered libraries. If `archunit` is not among them, add it: in this case, escalate by running:

```bash
ls "$INTELLIJ_ROOT/lib" | grep -i arch
grep -r "archunit" "$INTELLIJ_ROOT" --include="BUILD.bazel" 2>/dev/null | head -5
```

If ArchUnit is unavailable in the platform tree, fall back to plain JUnit reflection-based boundary tests (see Step 1 fallback below). Otherwise, modify `core/BUILD.bazel`:

```bazel
jvm_library(
    name = "core_test_lib",
    module_name = "intellij.termlab.core.tests",
    visibility = ["//visibility:public"],
    srcs = glob(["test/**/*.java"], allow_empty = True),
    deps = [
        ":core",
        "//termlab/sdk",
        "//libraries/jediterm-core:jediterm-core",
        "//libraries/jediterm-ui:jediterm-ui",
        "//libraries/junit5",
        "//libraries/junit5-jupiter",
        "//libraries/junit5-launcher",
        "@lib//:archunit",
        "@lib//:archunit-junit5",
        "@lib//:jetbrains-annotations",
        "@lib//:kotlin-stdlib",
    ],
)
```

**Step 1 fallback (if ArchUnit unavailable in `@lib//`):** skip the `@lib//:archunit*` deps and write reflection-based boundary tests using `java.lang.Class.getPackage()` and walking the classpath via `ClassGraph` or hand-rolled `Files.walk`. Acceptable but uglier. Prefer ArchUnit.

- [ ] **Step 2: Write `HexagonalBoundariesTest.java`**

Create `core/test/com/termlab/core/terminal/architecture/HexagonalBoundariesTest.java`:

```java
package com.termlab.core.terminal.architecture;

import com.tngtech.archunit.core.domain.JavaClasses;
import com.tngtech.archunit.core.importer.ClassFileImporter;
import com.tngtech.archunit.core.importer.ImportOption;
import com.tngtech.archunit.lang.ArchRule;
import org.junit.jupiter.api.Test;

import static com.tngtech.archunit.lang.syntax.ArchRuleDefinition.classes;
import static com.tngtech.archunit.lang.syntax.ArchRuleDefinition.noClasses;

/**
 * Mechanically enforces the hexagonal dependency rules for
 * {@code core/terminal/}. Run via IntelliJ's JUnit runner.
 *
 * <p>If a test fails here, do not relax the rule — fix the offending import.
 * The rules are the architecture; loosening them is architectural drift.
 */
final class HexagonalBoundariesTest {

    private static final JavaClasses TERMINAL_CLASSES =
        new ClassFileImporter()
            .withImportOption(ImportOption.Predefined.DO_NOT_INCLUDE_TESTS)
            .importPackages("com.termlab.core.terminal");

    @Test
    void domainPackageHasNoForbiddenImports() {
        ArchRule rule = noClasses()
            .that().resideInAPackage("..terminal.domain..")
            .should().dependOnClassesThat().resideInAnyPackage(
                "com.intellij..",
                "javax.swing..",
                "com.jediterm..",
                "org.apache.sshd..",
                "com.termlab.core.terminal.application..",
                "com.termlab.core.terminal.ports..",
                "com.termlab.core.terminal.adapters.."
            );
        rule.check(TERMINAL_CLASSES);
    }

    @Test
    void applicationPackageOnlyDependsOnDomainAndPorts() {
        ArchRule rule = noClasses()
            .that().resideInAPackage("..terminal.application..")
            .should().dependOnClassesThat().resideInAnyPackage(
                "com.intellij..",
                "javax.swing..",
                "com.jediterm..",
                "org.apache.sshd..",
                "com.termlab.core.terminal.adapters.."
            );
        rule.check(TERMINAL_CLASSES);
    }

    @Test
    void portsPackageHasNoForbiddenImports() {
        ArchRule rule = noClasses()
            .that().resideInAPackage("..terminal.ports..")
            .should().dependOnClassesThat().resideInAnyPackage(
                "com.intellij..",
                "javax.swing..",
                "com.jediterm..",
                "org.apache.sshd..",
                "com.termlab.core.terminal.application..",
                "com.termlab.core.terminal.adapters.."
            );
        rule.check(TERMINAL_CLASSES);
    }

    @Test
    void portsPackageContainsOnlyInterfaces() {
        ArchRule rule = classes()
            .that().resideInAPackage("..terminal.ports..")
            .and().areNotAnnotations()
            .and().areTopLevelClasses()
            .should().beInterfaces();
        rule.check(TERMINAL_CLASSES);
    }

    @Test
    void intellijAdapterDoesNotImportJediTermDirectly() {
        ArchRule rule = noClasses()
            .that().resideInAPackage("..terminal.adapters.intellij..")
            .should().dependOnClassesThat().resideInAPackage("com.jediterm..");
        rule.check(TERMINAL_CLASSES);
    }

    @Test
    void jediTermAdapterDoesNotImportIntellij() {
        ArchRule rule = noClasses()
            .that().resideInAPackage("..terminal.adapters.jediterm..")
            .should().dependOnClassesThat().resideInAPackage("com.intellij..");
        rule.check(TERMINAL_CLASSES);
    }

    /**
     * The single allowed cross-adapter touchpoint: intellij adapter may
     * reference {@code adapters.jediterm.JediTermSessionHandle} (our class)
     * to access the embedded widget. No other jediterm-package class may be
     * imported from intellij.
     */
    @Test
    void intellijAdapterMayOnlyReferenceJediTermSessionHandleFromJediTermPackage() {
        ArchRule rule = noClasses()
            .that().resideInAPackage("..terminal.adapters.intellij..")
            .should().dependOnClassesThat()
                .resideInAPackage("..terminal.adapters.jediterm..")
                .andShould().notHaveSimpleName("JediTermSessionHandle");
        // Phrasing this rule with ArchUnit's syntax is awkward; simpler form:
        // assert that any reference to adapters.jediterm.* resolves only to JediTermSessionHandle.
        // If the above syntax doesn't compile, replace with the imperative form below.
        rule.check(TERMINAL_CLASSES);
    }
}
```

If the last rule's syntax doesn't compile cleanly with the ArchUnit version available, replace it with the imperative form:

```java
@Test
void intellijAdapterMayOnlyReferenceJediTermSessionHandleFromJediTermPackage() {
    var imported = TERMINAL_CLASSES.stream()
        .filter(c -> c.getPackageName().startsWith("com.termlab.core.terminal.adapters.intellij"))
        .toList();
    for (var clazz : imported) {
        for (var dep : clazz.getDirectDependenciesFromSelf()) {
            String tgt = dep.getTargetClass().getName();
            if (tgt.startsWith("com.termlab.core.terminal.adapters.jediterm.")
                && !tgt.equals("com.termlab.core.terminal.adapters.jediterm.JediTermSessionHandle")) {
                throw new AssertionError(
                    "intellij adapter " + clazz.getName()
                    + " imports " + tgt
                    + " — only JediTermSessionHandle is allowed across adapter packages.");
            }
        }
    }
}
```

- [ ] **Step 3: Compile-check the new test class**

```bash
bash bazel.cmd build //termlab/core:core_test_lib
```

Expected: BUILD SUCCESS.

- [ ] **Step 4: Run via IntelliJ to verify all rules pass on empty packages**

In IntelliJ: right-click `HexagonalBoundariesTest` → "Run 'HexagonalBoundariesTest'".
Expected: 7 tests, all pass. Empty packages have no imports, so all rules trivially pass.

- [ ] **Step 5: Commit**

```bash
git add core/BUILD.bazel core/test/com/termlab/core/terminal/architecture/
git commit -m "$(cat <<'EOF'
test(core): add ArchUnit hexagonal boundary tests

Plan #1 Task 4. Mechanically enforces domain/application/ports/adapters
dependency rules. Trivially passes on empty packages; begins to enforce
the moment Task 5 places types in domain/.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 5: Domain — `Result<T, E>` sealed type

**Pattern:** Result Object. A two-arm sealed interface (`Success<T>` | `Failure<E>`) for use cases that have expected failures. Forces callers to handle both arms via switch-pattern-matching.

**Why:** the spec rejects throwing exceptions for expected failures (cancelled, unknown provider, missing capability). Java 21's sealed interfaces + records let us model this in ~30 lines without a third-party library.

**How it works:** `Result<T, E>` is a sealed interface with two records: `Success(T value)` and `Failure(E error)`. Static factories `Result.success(value)` and `Result.failure(error)` produce them. Helpers `isSuccess()`, `value()`, `error()`, `map(...)`, `flatMap(...)` round it out.

**Precursor to:** every use case in `application/` returns `Result<T, OpenError>` or similar. SSH plan's `ConnectSshSessionUseCase` will use the same pattern.

**Files:**
- Create: `core/src/com/termlab/core/terminal/domain/Result.java`
- Create: `core/test/com/termlab/core/terminal/domain/ResultTest.java`

- [ ] **Step 1: Write the failing test**

Create `core/test/com/termlab/core/terminal/domain/ResultTest.java`:

```java
package com.termlab.core.terminal.domain;

import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.*;

final class ResultTest {

    @Test
    void successCarriesValue() {
        Result<Integer, String> r = Result.success(42);
        assertTrue(r.isSuccess());
        assertFalse(r.isFailure());
        assertEquals(42, r.value());
    }

    @Test
    void failureCarriesError() {
        Result<Integer, String> r = Result.failure("boom");
        assertFalse(r.isSuccess());
        assertTrue(r.isFailure());
        assertEquals("boom", r.error());
    }

    @Test
    void successValueOnFailureThrows() {
        Result<Integer, String> r = Result.failure("boom");
        assertThrows(IllegalStateException.class, r::value);
    }

    @Test
    void failureErrorOnSuccessThrows() {
        Result<Integer, String> r = Result.success(42);
        assertThrows(IllegalStateException.class, r::error);
    }

    @Test
    void mapTransformsSuccess() {
        Result<Integer, String> r = Result.<Integer, String>success(21).map(v -> v * 2);
        assertTrue(r.isSuccess());
        assertEquals(42, r.value());
    }

    @Test
    void mapPassesFailureThrough() {
        Result<Integer, String> r = Result.<Integer, String>failure("boom").map(v -> v * 2);
        assertTrue(r.isFailure());
        assertEquals("boom", r.error());
    }

    @Test
    void flatMapChainsSuccess() {
        Result<Integer, String> r = Result.<Integer, String>success(21)
            .flatMap(v -> Result.success(v * 2));
        assertTrue(r.isSuccess());
        assertEquals(42, r.value());
    }

    @Test
    void flatMapShortCircuitsOnFailure() {
        Result<Integer, String> r = Result.<Integer, String>failure("boom")
            .flatMap(v -> Result.success(v * 2));
        assertTrue(r.isFailure());
        assertEquals("boom", r.error());
    }

    @Test
    void successAndFailureAreSwitchExhaustive() {
        Result<Integer, String> r = Result.success(42);
        String s = switch (r) {
            case Result.Success<Integer, String> ok -> "ok=" + ok.value();
            case Result.Failure<Integer, String> err -> "err=" + err.error();
        };
        assertEquals("ok=42", s);
    }
}
```

- [ ] **Step 2: Verify it fails to compile (no `Result` yet)**

```bash
bash bazel.cmd build //termlab/core:core_test_lib
```

Expected: BUILD FAILURE. The error message names `Result` as not found.

- [ ] **Step 3: Implement `Result<T, E>`**

Create `core/src/com/termlab/core/terminal/domain/Result.java`:

```java
package com.termlab.core.terminal.domain;

import java.util.function.Function;

/**
 * Two-armed sealed result type for use-case return values.
 *
 * <p>{@link Success} carries a value; {@link Failure} carries an error.
 * Both arms are exhaustive in switch-pattern-matching.
 *
 * @param <T> success value type
 * @param <E> failure error type
 */
public sealed interface Result<T, E> permits Result.Success, Result.Failure {

    static <T, E> Result<T, E> success(T value) {
        return new Success<>(value);
    }

    static <T, E> Result<T, E> failure(E error) {
        return new Failure<>(error);
    }

    default boolean isSuccess() {
        return this instanceof Success<T, E>;
    }

    default boolean isFailure() {
        return this instanceof Failure<T, E>;
    }

    default T value() {
        if (this instanceof Success<T, E> s) {
            return s.value();
        }
        throw new IllegalStateException("Result is a Failure");
    }

    default E error() {
        if (this instanceof Failure<T, E> f) {
            return f.error();
        }
        throw new IllegalStateException("Result is a Success");
    }

    default <U> Result<U, E> map(Function<? super T, ? extends U> f) {
        return switch (this) {
            case Success<T, E> s -> Result.success(f.apply(s.value()));
            case Failure<T, E> err -> Result.failure(err.error());
        };
    }

    default <U> Result<U, E> flatMap(Function<? super T, Result<U, E>> f) {
        return switch (this) {
            case Success<T, E> s -> f.apply(s.value());
            case Failure<T, E> err -> Result.failure(err.error());
        };
    }

    record Success<T, E>(T value) implements Result<T, E> {}

    record Failure<T, E>(E error) implements Result<T, E> {}
}
```

- [ ] **Step 4: Run tests via IntelliJ**

In IntelliJ: right-click `ResultTest` → "Run 'ResultTest'".
Expected: 9 tests, all pass.

- [ ] **Step 5: Compile-check via Bazel**

```bash
bash bazel.cmd build //termlab/core:core_test_lib
```

Expected: BUILD SUCCESS.

- [ ] **Step 6: Run ArchUnit boundary test to confirm no rule violation**

In IntelliJ: re-run `HexagonalBoundariesTest`.
Expected: 7 tests pass. `Result` lives in `domain/` and imports only `java.util.function.Function`.

- [ ] **Step 7: Commit**

```bash
git add core/src/com/termlab/core/terminal/domain/Result.java \
        core/test/com/termlab/core/terminal/domain/ResultTest.java
git commit -m "$(cat <<'EOF'
feat(core): add Result<T,E> sealed type for use-case returns

Plan #1 Task 5. Two-armed sealed Result with map/flatMap. Domain-only
import surface (java.util.function.Function). Used by upcoming
OpenTerminalUseCase and CloseTerminalUseCase.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 6: Domain — `TerminalSessionId` and `TerminalProviderId`

**Pattern:** Value Object (typed identity wrapper). Opaque around `String`/`UUID` so call sites can't accidentally pass a raw string where an id is expected, and can't mix up the two id types.

**Why:** today the codebase uses raw `String` for both session id (random UUID) and provider id (e.g., `"com.termlab.local-pty"`). Typed wrappers prevent accidental mismatches and make refactors mechanical.

**Files:**
- Create: `core/src/com/termlab/core/terminal/domain/TerminalSessionId.java`
- Create: `core/src/com/termlab/core/terminal/domain/TerminalProviderId.java`
- Create: `core/test/com/termlab/core/terminal/domain/TerminalSessionIdTest.java`
- Create: `core/test/com/termlab/core/terminal/domain/TerminalProviderIdTest.java`

- [ ] **Step 1: Write failing tests for `TerminalSessionId`**

`core/test/com/termlab/core/terminal/domain/TerminalSessionIdTest.java`:

```java
package com.termlab.core.terminal.domain;

import org.junit.jupiter.api.Test;
import java.util.UUID;
import static org.junit.jupiter.api.Assertions.*;

final class TerminalSessionIdTest {

    @Test
    void newIdGeneratesUniqueValues() {
        TerminalSessionId a = TerminalSessionId.newId();
        TerminalSessionId b = TerminalSessionId.newId();
        assertNotEquals(a, b);
    }

    @Test
    void equalityIsByValue() {
        UUID raw = UUID.randomUUID();
        TerminalSessionId a = TerminalSessionId.of(raw);
        TerminalSessionId b = TerminalSessionId.of(raw);
        assertEquals(a, b);
        assertEquals(a.hashCode(), b.hashCode());
    }

    @Test
    void asStringReturnsUuidString() {
        UUID raw = UUID.fromString("00000000-0000-0000-0000-000000000001");
        assertEquals("00000000-0000-0000-0000-000000000001", TerminalSessionId.of(raw).asString());
    }

    @Test
    void parseRoundTripsAsString() {
        TerminalSessionId a = TerminalSessionId.newId();
        TerminalSessionId b = TerminalSessionId.parse(a.asString());
        assertEquals(a, b);
    }

    @Test
    void parseRejectsInvalidString() {
        assertThrows(IllegalArgumentException.class, () -> TerminalSessionId.parse("not-a-uuid"));
    }

    @Test
    void ofRejectsNullUuid() {
        assertThrows(NullPointerException.class, () -> TerminalSessionId.of(null));
    }
}
```

- [ ] **Step 2: Write failing tests for `TerminalProviderId`**

`core/test/com/termlab/core/terminal/domain/TerminalProviderIdTest.java`:

```java
package com.termlab.core.terminal.domain;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

final class TerminalProviderIdTest {

    @Test
    void equalityIsByValue() {
        TerminalProviderId a = TerminalProviderId.of("com.termlab.local-pty");
        TerminalProviderId b = TerminalProviderId.of("com.termlab.local-pty");
        assertEquals(a, b);
        assertEquals(a.hashCode(), b.hashCode());
    }

    @Test
    void differentValuesAreUnequal() {
        assertNotEquals(
            TerminalProviderId.of("com.termlab.local-pty"),
            TerminalProviderId.of("com.termlab.ssh")
        );
    }

    @Test
    void asStringReturnsRawValue() {
        assertEquals("com.termlab.local-pty",
            TerminalProviderId.of("com.termlab.local-pty").asString());
    }

    @Test
    void ofRejectsBlank() {
        assertThrows(IllegalArgumentException.class, () -> TerminalProviderId.of(""));
        assertThrows(IllegalArgumentException.class, () -> TerminalProviderId.of("  "));
    }

    @Test
    void ofRejectsNull() {
        assertThrows(NullPointerException.class, () -> TerminalProviderId.of(null));
    }
}
```

- [ ] **Step 3: Verify tests fail to compile**

```bash
bash bazel.cmd build //termlab/core:core_test_lib
```

Expected: BUILD FAILURE — `TerminalSessionId` and `TerminalProviderId` not found.

- [ ] **Step 4: Implement `TerminalSessionId`**

`core/src/com/termlab/core/terminal/domain/TerminalSessionId.java`:

```java
package com.termlab.core.terminal.domain;

import java.util.Objects;
import java.util.UUID;

/**
 * Stable identity for a terminal session. Opaque wrapper around {@link UUID}
 * so call sites can't accidentally pass a raw String.
 */
public final class TerminalSessionId {

    private final UUID value;

    private TerminalSessionId(UUID value) {
        this.value = Objects.requireNonNull(value, "value");
    }

    public static TerminalSessionId newId() {
        return new TerminalSessionId(UUID.randomUUID());
    }

    public static TerminalSessionId of(UUID value) {
        return new TerminalSessionId(value);
    }

    public static TerminalSessionId parse(String text) {
        return new TerminalSessionId(UUID.fromString(text));
    }

    public String asString() {
        return value.toString();
    }

    @Override
    public boolean equals(Object o) {
        return o instanceof TerminalSessionId other && value.equals(other.value);
    }

    @Override
    public int hashCode() {
        return value.hashCode();
    }

    @Override
    public String toString() {
        return "TerminalSessionId[" + value + "]";
    }
}
```

- [ ] **Step 5: Implement `TerminalProviderId`**

`core/src/com/termlab/core/terminal/domain/TerminalProviderId.java`:

```java
package com.termlab.core.terminal.domain;

import java.util.Objects;

/**
 * Stable identity for a terminal session provider (e.g., {@code "com.termlab.local-pty"}).
 * Typed so registry lookups don't mix provider ids with session ids.
 */
public final class TerminalProviderId {

    private final String value;

    private TerminalProviderId(String value) {
        Objects.requireNonNull(value, "value");
        if (value.isBlank()) {
            throw new IllegalArgumentException("provider id must not be blank");
        }
        this.value = value;
    }

    public static TerminalProviderId of(String value) {
        return new TerminalProviderId(value);
    }

    public String asString() {
        return value;
    }

    @Override
    public boolean equals(Object o) {
        return o instanceof TerminalProviderId other && value.equals(other.value);
    }

    @Override
    public int hashCode() {
        return value.hashCode();
    }

    @Override
    public String toString() {
        return "TerminalProviderId[" + value + "]";
    }
}
```

- [ ] **Step 6: Run tests via IntelliJ**

Run both `TerminalSessionIdTest` and `TerminalProviderIdTest`.
Expected: 11 tests, all pass. Re-run `HexagonalBoundariesTest` — should still pass (these classes only import JDK).

- [ ] **Step 7: Commit**

```bash
git add core/src/com/termlab/core/terminal/domain/TerminalSessionId.java \
        core/src/com/termlab/core/terminal/domain/TerminalProviderId.java \
        core/test/com/termlab/core/terminal/domain/TerminalSessionIdTest.java \
        core/test/com/termlab/core/terminal/domain/TerminalProviderIdTest.java
git commit -m "$(cat <<'EOF'
feat(core): add TerminalSessionId and TerminalProviderId value objects

Plan #1 Task 6. Typed identity wrappers replace raw String. Domain-only
imports.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 7: Domain — `TerminalCapability` enum and `TerminalCapabilities` value object

**Pattern:** Value Object holding a `Set` of enum flags. Replaces the hardcoded `SSH_PROVIDER_ID = "com.termlab.ssh"` check by letting providers declare what they support.

**Files:**
- Create: `core/src/com/termlab/core/terminal/domain/TerminalCapability.java`
- Create: `core/src/com/termlab/core/terminal/domain/TerminalCapabilities.java`
- Create: `core/test/com/termlab/core/terminal/domain/TerminalCapabilitiesTest.java`

- [ ] **Step 1: Write failing test**

`core/test/com/termlab/core/terminal/domain/TerminalCapabilitiesTest.java`:

```java
package com.termlab.core.terminal.domain;

import org.junit.jupiter.api.Test;
import static com.termlab.core.terminal.domain.TerminalCapability.*;
import static org.junit.jupiter.api.Assertions.*;

final class TerminalCapabilitiesTest {

    @Test
    void emptyHasNoCapabilities() {
        TerminalCapabilities caps = TerminalCapabilities.none();
        assertFalse(caps.has(SUPPORTS_MULTI_EXEC));
        assertFalse(caps.has(CLOSE_TAB_ON_SESSION_END));
    }

    @Test
    void ofCapturesProvidedFlags() {
        TerminalCapabilities caps = TerminalCapabilities.of(SUPPORTS_MULTI_EXEC, CLOSE_TAB_ON_SESSION_END);
        assertTrue(caps.has(SUPPORTS_MULTI_EXEC));
        assertTrue(caps.has(CLOSE_TAB_ON_SESSION_END));
        assertFalse(caps.has(SUPPORTS_BROADCAST));
    }

    @Test
    void unionMergesFlags() {
        TerminalCapabilities a = TerminalCapabilities.of(SUPPORTS_MULTI_EXEC);
        TerminalCapabilities b = TerminalCapabilities.of(CLOSE_TAB_ON_SESSION_END);
        TerminalCapabilities merged = a.union(b);
        assertTrue(merged.has(SUPPORTS_MULTI_EXEC));
        assertTrue(merged.has(CLOSE_TAB_ON_SESSION_END));
    }

    @Test
    void equalityIsByContents() {
        TerminalCapabilities a = TerminalCapabilities.of(SUPPORTS_MULTI_EXEC, CLOSE_TAB_ON_SESSION_END);
        TerminalCapabilities b = TerminalCapabilities.of(CLOSE_TAB_ON_SESSION_END, SUPPORTS_MULTI_EXEC);
        assertEquals(a, b);
        assertEquals(a.hashCode(), b.hashCode());
    }

    @Test
    void capabilitiesAreImmutable() {
        TerminalCapabilities caps = TerminalCapabilities.of(SUPPORTS_MULTI_EXEC);
        assertThrows(UnsupportedOperationException.class, () -> caps.asSet().add(SUPPORTS_BROADCAST));
    }
}
```

- [ ] **Step 2: Verify failure**

```bash
bash bazel.cmd build //termlab/core:core_test_lib
```

Expected: BUILD FAILURE.

- [ ] **Step 3: Implement `TerminalCapability` enum**

`core/src/com/termlab/core/terminal/domain/TerminalCapability.java`:

```java
package com.termlab.core.terminal.domain;

/**
 * Individual capabilities a {@code TerminalSessionProvider} may declare.
 *
 * <p>New capabilities are added here as plugins need them. Replace hardcoded
 * provider-id checks with capability lookups.
 */
public enum TerminalCapability {

    /**
     * Provider opts into MultiExec broadcast targeting. Set by the SSH
     * provider once MultiExec is reintroduced in plugins/ssh; not set by
     * local PTY (which doesn't broadcast).
     */
    SUPPORTS_MULTI_EXEC,

    /**
     * Provider opts into broadcast input — its sessions can receive
     * keystrokes pushed by another session.
     */
    SUPPORTS_BROADCAST,

    /**
     * When the underlying session ends (process exit, disconnect), the
     * editor tab should close automatically. Local PTY sets this; long-lived
     * remote providers (SSH) do not — they prefer to show a disconnected
     * state in place of closing the tab.
     */
    CLOSE_TAB_ON_SESSION_END
}
```

- [ ] **Step 4: Implement `TerminalCapabilities`**

`core/src/com/termlab/core/terminal/domain/TerminalCapabilities.java`:

```java
package com.termlab.core.terminal.domain;

import java.util.Collections;
import java.util.EnumSet;
import java.util.Objects;
import java.util.Set;

/**
 * Immutable {@link Set} of {@link TerminalCapability} flags declared by a
 * provider. Replaces hardcoded provider-id checks.
 */
public final class TerminalCapabilities {

    private static final TerminalCapabilities NONE =
        new TerminalCapabilities(EnumSet.noneOf(TerminalCapability.class));

    private final Set<TerminalCapability> flags;

    private TerminalCapabilities(Set<TerminalCapability> flags) {
        this.flags = Collections.unmodifiableSet(EnumSet.copyOf(flags));
    }

    public static TerminalCapabilities none() {
        return NONE;
    }

    public static TerminalCapabilities of(TerminalCapability... caps) {
        Objects.requireNonNull(caps, "caps");
        if (caps.length == 0) {
            return NONE;
        }
        EnumSet<TerminalCapability> set = EnumSet.noneOf(TerminalCapability.class);
        for (TerminalCapability c : caps) {
            set.add(Objects.requireNonNull(c, "capability"));
        }
        return new TerminalCapabilities(set);
    }

    public boolean has(TerminalCapability cap) {
        return flags.contains(cap);
    }

    public TerminalCapabilities union(TerminalCapabilities other) {
        EnumSet<TerminalCapability> merged = EnumSet.copyOf(flags);
        merged.addAll(other.flags);
        return new TerminalCapabilities(merged);
    }

    public Set<TerminalCapability> asSet() {
        return flags;
    }

    @Override
    public boolean equals(Object o) {
        return o instanceof TerminalCapabilities other && flags.equals(other.flags);
    }

    @Override
    public int hashCode() {
        return flags.hashCode();
    }

    @Override
    public String toString() {
        return "TerminalCapabilities" + flags;
    }
}
```

- [ ] **Step 5: Run tests via IntelliJ**

Run `TerminalCapabilitiesTest`. Expected: 5 tests pass. Re-run `HexagonalBoundariesTest` — pass.

- [ ] **Step 6: Commit**

```bash
git add core/src/com/termlab/core/terminal/domain/TerminalCapability.java \
        core/src/com/termlab/core/terminal/domain/TerminalCapabilities.java \
        core/test/com/termlab/core/terminal/domain/TerminalCapabilitiesTest.java
git commit -m "$(cat <<'EOF'
feat(core): add TerminalCapability + TerminalCapabilities value object

Plan #1 Task 7. Replaces hardcoded provider-id checks. Local PTY will
declare CLOSE_TAB_ON_SESSION_END; SSH (future) declares
SUPPORTS_MULTI_EXEC and SUPPORTS_BROADCAST.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 8: Domain — `TerminalTitle` and `TerminalSessionDescriptor`

**Pattern:** Value Object (record) for the descriptor. The descriptor is the *immutable opening intent* — what's needed to open or persist a session. Runtime title mutation lives on `TerminalSessionHandle` (Task 12).

**Files:**
- Create: `core/src/com/termlab/core/terminal/domain/TerminalTitle.java`
- Create: `core/src/com/termlab/core/terminal/domain/TerminalSessionDescriptor.java`
- Create: `core/test/com/termlab/core/terminal/domain/TerminalTitleTest.java`
- Create: `core/test/com/termlab/core/terminal/domain/TerminalSessionDescriptorTest.java`

- [ ] **Step 1: Write failing tests**

`core/test/com/termlab/core/terminal/domain/TerminalTitleTest.java`:

```java
package com.termlab.core.terminal.domain;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

final class TerminalTitleTest {

    @Test
    void displayValueReturnsRawString() {
        assertEquals("zsh", TerminalTitle.of("zsh").displayValue());
    }

    @Test
    void blankIsRejected() {
        assertThrows(IllegalArgumentException.class, () -> TerminalTitle.of(""));
        assertThrows(IllegalArgumentException.class, () -> TerminalTitle.of("   "));
    }

    @Test
    void nullIsRejected() {
        assertThrows(NullPointerException.class, () -> TerminalTitle.of(null));
    }

    @Test
    void equalityIsByValue() {
        assertEquals(TerminalTitle.of("zsh"), TerminalTitle.of("zsh"));
    }
}
```

`core/test/com/termlab/core/terminal/domain/TerminalSessionDescriptorTest.java`:

```java
package com.termlab.core.terminal.domain;

import org.junit.jupiter.api.Test;
import java.util.Map;
import static org.junit.jupiter.api.Assertions.*;

final class TerminalSessionDescriptorTest {

    private static final TerminalProviderId LOCAL_PTY = TerminalProviderId.of("com.termlab.local-pty");

    @Test
    void localPtyFactoryBuildsDescriptor() {
        TerminalSessionDescriptor d = TerminalSessionDescriptor.localPty(
            TerminalTitle.of("zsh"), "/home/dustin", Map.of());
        assertEquals(LOCAL_PTY, d.providerId());
        assertEquals("zsh", d.displayTitle().displayValue());
        assertEquals("/home/dustin", d.workingDirectory());
        assertTrue(d.providerState().isEmpty());
    }

    @Test
    void providerStateIsImmutable() {
        Map<String, String> state = new java.util.HashMap<>();
        state.put("key", "value");
        TerminalSessionDescriptor d = new TerminalSessionDescriptor(
            LOCAL_PTY, TerminalTitle.of("zsh"), null, state);
        assertThrows(UnsupportedOperationException.class,
            () -> d.providerState().put("other", "x"));
    }

    @Test
    void equalityIsByContents() {
        TerminalSessionDescriptor a = TerminalSessionDescriptor.localPty(
            TerminalTitle.of("zsh"), "/home/dustin", Map.of());
        TerminalSessionDescriptor b = TerminalSessionDescriptor.localPty(
            TerminalTitle.of("zsh"), "/home/dustin", Map.of());
        assertEquals(a, b);
    }

    @Test
    void nullProviderIdRejected() {
        assertThrows(NullPointerException.class, () ->
            new TerminalSessionDescriptor(null, TerminalTitle.of("zsh"), null, Map.of()));
    }

    @Test
    void nullTitleRejected() {
        assertThrows(NullPointerException.class, () ->
            new TerminalSessionDescriptor(LOCAL_PTY, null, null, Map.of()));
    }

    @Test
    void nullWorkingDirectoryAllowed() {
        TerminalSessionDescriptor d = new TerminalSessionDescriptor(
            LOCAL_PTY, TerminalTitle.of("zsh"), null, Map.of());
        assertNull(d.workingDirectory());
    }

    @Test
    void nullProviderStateBecomesEmptyMap() {
        TerminalSessionDescriptor d = new TerminalSessionDescriptor(
            LOCAL_PTY, TerminalTitle.of("zsh"), "/", null);
        assertTrue(d.providerState().isEmpty());
    }
}
```

- [ ] **Step 2: Verify failure**

```bash
bash bazel.cmd build //termlab/core:core_test_lib
```

Expected: BUILD FAILURE.

- [ ] **Step 3: Implement `TerminalTitle`**

`core/src/com/termlab/core/terminal/domain/TerminalTitle.java`:

```java
package com.termlab.core.terminal.domain;

import java.util.Objects;

/** Display title for a terminal tab. Immutable. */
public final class TerminalTitle {

    private final String value;

    private TerminalTitle(String value) {
        Objects.requireNonNull(value, "value");
        if (value.isBlank()) {
            throw new IllegalArgumentException("title must not be blank");
        }
        this.value = value;
    }

    public static TerminalTitle of(String value) {
        return new TerminalTitle(value);
    }

    public String displayValue() {
        return value;
    }

    @Override
    public boolean equals(Object o) {
        return o instanceof TerminalTitle other && value.equals(other.value);
    }

    @Override
    public int hashCode() {
        return value.hashCode();
    }

    @Override
    public String toString() {
        return "TerminalTitle[" + value + "]";
    }
}
```

- [ ] **Step 4: Implement `TerminalSessionDescriptor`**

`core/src/com/termlab/core/terminal/domain/TerminalSessionDescriptor.java`:

```java
package com.termlab.core.terminal.domain;

import java.util.Collections;
import java.util.HashMap;
import java.util.Map;
import java.util.Objects;

/**
 * Immutable opening intent for a terminal session.
 *
 * <p>Captures everything needed to open or persist a session:
 * <ul>
 *   <li>{@code providerId} — which {@code TerminalSessionProvider} handles it</li>
 *   <li>{@code displayTitle} — initial tab title (may be updated at runtime
 *       via OSC 0/2; runtime updates live on the session handle, not here)</li>
 *   <li>{@code workingDirectory} — initial CWD ({@code null} = provider default)</li>
 *   <li>{@code providerState} — provider-specific opaque state for restore
 *       (e.g., SSH host id). String-to-String for trivial JSON serialization.</li>
 * </ul>
 *
 * <p>Runtime state (current CWD after {@code cd}, current title after OSC 0/2,
 * manual title override) lives on {@code TerminalSessionHandle}, not here.
 */
public record TerminalSessionDescriptor(
    TerminalProviderId providerId,
    TerminalTitle displayTitle,
    String workingDirectory,
    Map<String, String> providerState
) {

    public TerminalSessionDescriptor {
        Objects.requireNonNull(providerId, "providerId");
        Objects.requireNonNull(displayTitle, "displayTitle");
        // workingDirectory may be null
        providerState = providerState == null
            ? Collections.emptyMap()
            : Collections.unmodifiableMap(new HashMap<>(providerState));
    }

    public static TerminalSessionDescriptor localPty(
        TerminalTitle displayTitle, String workingDirectory, Map<String, String> providerState) {
        return new TerminalSessionDescriptor(
            TerminalProviderId.of("com.termlab.local-pty"),
            displayTitle,
            workingDirectory,
            providerState
        );
    }
}
```

- [ ] **Step 5: Run tests via IntelliJ**

Run both new test classes. Expected: 11 tests pass. Re-run `HexagonalBoundariesTest`.

- [ ] **Step 6: Commit**

```bash
git add core/src/com/termlab/core/terminal/domain/TerminalTitle.java \
        core/src/com/termlab/core/terminal/domain/TerminalSessionDescriptor.java \
        core/test/com/termlab/core/terminal/domain/TerminalTitleTest.java \
        core/test/com/termlab/core/terminal/domain/TerminalSessionDescriptorTest.java
git commit -m "$(cat <<'EOF'
feat(core): add TerminalTitle + TerminalSessionDescriptor

Plan #1 Task 8. Descriptor is the immutable opening intent; runtime
state (current CWD, current title) lives on TerminalSessionHandle.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 9: Domain — `OpenError` sealed type

**Pattern:** Sealed Type for exhaustive error modeling. Each subtype is one of `OpenTerminalUseCase`'s expected failures.

**Files:**
- Create: `core/src/com/termlab/core/terminal/domain/OpenError.java`
- Create: `core/test/com/termlab/core/terminal/domain/OpenErrorTest.java`

- [ ] **Step 1: Write failing test**

```java
package com.termlab.core.terminal.domain;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

final class OpenErrorTest {

    @Test
    void unknownProviderCarriesId() {
        TerminalProviderId id = TerminalProviderId.of("com.termlab.local-pty");
        OpenError.UnknownProvider err = new OpenError.UnknownProvider(id);
        assertEquals(id, err.providerId());
    }

    @Test
    void capabilityMissingCarriesCapability() {
        OpenError.CapabilityMissing err = new OpenError.CapabilityMissing(TerminalCapability.SUPPORTS_BROADCAST);
        assertEquals(TerminalCapability.SUPPORTS_BROADCAST, err.capability());
    }

    @Test
    void userCancelledIsSingleton() {
        assertSame(OpenError.userCancelled(), OpenError.userCancelled());
    }

    @Test
    void switchIsExhaustive() {
        OpenError err = OpenError.userCancelled();
        String description = switch (err) {
            case OpenError.UserCancelled c -> "cancelled";
            case OpenError.UnknownProvider u -> "unknown:" + u.providerId().asString();
            case OpenError.CapabilityMissing m -> "missing:" + m.capability();
        };
        assertEquals("cancelled", description);
    }
}
```

- [ ] **Step 2: Verify failure**

```bash
bash bazel.cmd build //termlab/core:core_test_lib
```

- [ ] **Step 3: Implement `OpenError`**

```java
package com.termlab.core.terminal.domain;

import java.util.Objects;

/**
 * Expected failures from opening a terminal session. Returned in
 * {@code Result.failure(...)}.
 */
public sealed interface OpenError
    permits OpenError.UserCancelled,
            OpenError.UnknownProvider,
            OpenError.CapabilityMissing {

    static UserCancelled userCancelled() {
        return UserCancelled.INSTANCE;
    }

    final class UserCancelled implements OpenError {
        static final UserCancelled INSTANCE = new UserCancelled();
        private UserCancelled() {}
    }

    record UnknownProvider(TerminalProviderId providerId) implements OpenError {
        public UnknownProvider {
            Objects.requireNonNull(providerId, "providerId");
        }
    }

    record CapabilityMissing(TerminalCapability capability) implements OpenError {
        public CapabilityMissing {
            Objects.requireNonNull(capability, "capability");
        }
    }
}
```

- [ ] **Step 4: Run tests via IntelliJ**

Expected: 4 tests pass.

- [ ] **Step 5: Commit**

```bash
git add core/src/com/termlab/core/terminal/domain/OpenError.java \
        core/test/com/termlab/core/terminal/domain/OpenErrorTest.java
git commit -m "$(cat <<'EOF'
feat(core): add OpenError sealed type for terminal-open failures

Plan #1 Task 9. UserCancelled | UnknownProvider | CapabilityMissing.
Used by OpenTerminalUseCase return type.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 10: Ports — five interfaces

**Pattern:** Ports (interfaces only — no fields, no logic, no default methods carrying behavior). The five ports together form the seam between `application/` and infrastructure adapters.

**Why:** ports are exactly what plugins implement. Making them a literal package keeps the SPI surface small and explicit. ArchUnit Test #4 (`portsPackageContainsOnlyInterfaces`) enforces the constraint.

**Files:**
- Create: `core/src/com/termlab/core/terminal/ports/TerminalSessionHandle.java`
- Create: `core/src/com/termlab/core/terminal/ports/TerminalSessionFactory.java`
- Create: `core/src/com/termlab/core/terminal/ports/TerminalSessionProviderRegistry.java`
- Create: `core/src/com/termlab/core/terminal/ports/TerminalEventPublisher.java`
- Create: `core/src/com/termlab/core/terminal/ports/TerminalNotificationPort.java`
- Create: `core/src/com/termlab/core/terminal/ports/TerminalLifecycleEvent.java` (sealed event marker, lives in `ports/` because it's part of the publisher contract; alternative: put in `domain/` — but pure events are protocol, fitting `ports/` better)

- [ ] **Step 1: Create `TerminalLifecycleEvent.java`**

```java
package com.termlab.core.terminal.ports;

import com.termlab.core.terminal.domain.TerminalSessionDescriptor;
import com.termlab.core.terminal.domain.TerminalSessionId;
import com.termlab.core.terminal.domain.TerminalTitle;

/**
 * Sealed event protocol for terminal lifecycle. Published via
 * {@link TerminalEventPublisher}. Subscribers (Plan #2) listen on a
 * matching subscriber port.
 */
public sealed interface TerminalLifecycleEvent
    permits TerminalLifecycleEvent.TerminalOpened,
            TerminalLifecycleEvent.TerminalClosed,
            TerminalLifecycleEvent.TerminalCwdChanged,
            TerminalLifecycleEvent.TerminalTitleChanged {

    TerminalSessionId sessionId();

    record TerminalOpened(TerminalSessionId sessionId, TerminalSessionDescriptor descriptor)
        implements TerminalLifecycleEvent {}

    record TerminalClosed(TerminalSessionId sessionId, ExitReason reason)
        implements TerminalLifecycleEvent {}

    record TerminalCwdChanged(TerminalSessionId sessionId, String newCwd)
        implements TerminalLifecycleEvent {}

    record TerminalTitleChanged(TerminalSessionId sessionId, TerminalTitle newTitle)
        implements TerminalLifecycleEvent {}

    enum ExitReason {
        PROCESS_EXIT,
        USER_CLOSED,
        MONITOR_FAILED,
        DISPOSED
    }
}
```

- [ ] **Step 2: Create `TerminalSessionHandle.java`**

```java
package com.termlab.core.terminal.ports;

import com.termlab.core.terminal.domain.TerminalSessionId;
import com.termlab.core.terminal.domain.TerminalTitle;

import java.util.function.Consumer;

/**
 * Live-session control surface. The {@code application/} layer holds this;
 * it never sees {@code TtyConnector} or {@code JediTermWidget}.
 *
 * <p>Implementations: {@code adapters/jediterm/JediTermSessionHandle}.
 */
public interface TerminalSessionHandle {

    TerminalSessionId sessionId();

    /** Write bytes to the session's input stream. */
    void write(byte[] data);

    /** Resize the underlying terminal (rows × columns). */
    void resize(int rows, int columns);

    /** Current working directory as last reported (e.g., via OSC 7). May be null. */
    String currentWorkingDirectory();

    /** Current title as last reported (e.g., via OSC 0/2). */
    TerminalTitle currentTitle();

    /** Override the title and pin it (manual rename). */
    void overrideTitle(TerminalTitle title);

    /** Whether the title has been manually pinned (suppresses OSC 0/2 updates). */
    boolean hasManualTitleOverride();

    /** Subscribe to session-exit events. */
    void onExit(Consumer<TerminalLifecycleEvent.ExitReason> handler);

    /**
     * Block until the underlying session exits, returning its exit code.
     * Adapters call this from a watcher thread instead of touching the
     * vendor connector directly — keeps {@code adapters.intellij} from
     * importing JediTerm types.
     */
    int awaitExit() throws InterruptedException;

    /** Release resources. Idempotent. */
    void dispose();

    /** Whether {@link #dispose()} has been called. */
    boolean isDisposed();
}
```

- [ ] **Step 3: Create `TerminalSessionFactory.java`**

```java
package com.termlab.core.terminal.ports;

import com.termlab.core.terminal.domain.TerminalSessionDescriptor;
import com.termlab.core.terminal.domain.TerminalSessionId;
import com.termlab.sdk.TerminalSessionProvider;

/**
 * Builds a {@link TerminalSessionHandle} from a descriptor + provider.
 *
 * <p>Implementations: {@code adapters/jediterm/JediTermSessionFactory}.
 */
public interface TerminalSessionFactory {

    TerminalSessionHandle create(
        TerminalSessionId sessionId,
        TerminalSessionDescriptor descriptor,
        TerminalSessionProvider provider,
        TerminalSessionProvider.SessionContext context);
}
```

- [ ] **Step 4: Create `TerminalSessionProviderRegistry.java`**

```java
package com.termlab.core.terminal.ports;

import com.termlab.core.terminal.domain.TerminalProviderId;
import com.termlab.sdk.TerminalSessionProvider;

import java.util.List;
import java.util.Optional;

/**
 * Registry of {@link TerminalSessionProvider} keyed by {@link TerminalProviderId}.
 *
 * <p>Implementations: {@code application/InMemoryTerminalSessionProviderRegistry}.
 * Discovery (e.g., via IntelliJ extension points) is the adapter's responsibility;
 * the application layer only sees this port.
 */
public interface TerminalSessionProviderRegistry {

    /** Register a provider. Throws on duplicate id. */
    void register(TerminalSessionProvider provider);

    /** Look up a provider by id. */
    Optional<TerminalSessionProvider> find(TerminalProviderId id);

    /** All registered providers. */
    List<TerminalSessionProvider> all();
}
```

- [ ] **Step 5: Create `TerminalEventPublisher.java`**

```java
package com.termlab.core.terminal.ports;

/**
 * Publishes {@link TerminalLifecycleEvent}s. Plan #1 emits only — no
 * subscriber port yet (added in Plan #2).
 */
public interface TerminalEventPublisher {

    void publish(TerminalLifecycleEvent event);
}
```

- [ ] **Step 6: Create `TerminalNotificationPort.java`**

```java
package com.termlab.core.terminal.ports;

/**
 * Surfaces user-facing notifications. The {@code application/} layer never
 * knows about IntelliJ's {@code Notifications.Bus}.
 */
public interface TerminalNotificationPort {

    void info(String message);

    void warn(String message);

    void error(String message);
}
```

- [ ] **Step 7: Compile-check**

```bash
bash bazel.cmd build //termlab/core:core_test_lib
```

Expected: BUILD SUCCESS.

- [ ] **Step 8: Run `HexagonalBoundariesTest`**

In IntelliJ. Expected: 7 tests pass. Specifically `portsPackageContainsOnlyInterfaces` should pass — every top-level type in `ports/` is an interface (the records *inside* `TerminalLifecycleEvent` are inner types and don't count as top-level).

- [ ] **Step 9: Commit**

```bash
git add core/src/com/termlab/core/terminal/ports/
git commit -m "$(cat <<'EOF'
feat(core): add terminal ports (interfaces only)

Plan #1 Task 10. Five port interfaces + TerminalLifecycleEvent sealed
protocol. ArchUnit verifies ports/ contains only interfaces.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 11: SDK — add default-method `capabilities()` to `TerminalSessionProvider`

**Pattern:** Non-breaking SPI extension via Java default method. Existing implementations compile unchanged; new implementations override.

**Why:** `TerminalCapabilities` is the replacement for hardcoded provider-id checks. The SPI must expose capabilities so the application layer can query them. Adding a default method (returning empty caps) keeps existing implementations working.

**Files:**
- Modify: `sdk/src/com/termlab/sdk/TerminalSessionProvider.java`

- [ ] **Step 1: Modify the SDK SPI**

Open `sdk/src/com/termlab/sdk/TerminalSessionProvider.java`. Add the import and method. The existing `closeTabOnSessionEnd()` default method stays for source compatibility but is now expressed as "does the provider declare `CLOSE_TAB_ON_SESSION_END`?" — its default implementation delegates to `capabilities()`. Replace the existing `closeTabOnSessionEnd()` body:

```java
package com.termlab.sdk;

import com.jediterm.terminal.TtyConnector;
import com.termlab.core.terminal.domain.TerminalCapabilities;
import com.termlab.core.terminal.domain.TerminalCapability;
import org.jetbrains.annotations.NotNull;
import org.jetbrains.annotations.Nullable;

import javax.swing.*;

/**
 * Extension point for plugins that provide terminal session backends.
 * Implementations supply a TtyConnector that JediTerm renders.
 * The core ships a local PTY provider; plugins add SSH, Docker, serial, etc.
 */
public interface TerminalSessionProvider {

    /** Unique identifier for this provider (e.g., "com.termlab.local-pty"). */
    @NotNull String getId();

    /** Human-readable name shown in UI (e.g., "Local Terminal", "SSH"). */
    @NotNull String getDisplayName();

    /** Icon for tab bar and menus. */
    @Nullable Icon getIcon();

    /**
     * Whether this provider can open a session immediately without user input.
     * Local PTY returns true. SSH (needs host selection) returns false.
     */
    boolean canQuickOpen();

    /**
     * Capabilities this provider declares. Replaces hardcoded provider-id
     * dispatch. Default: no capabilities.
     */
    default @NotNull TerminalCapabilities capabilities() {
        return TerminalCapabilities.none();
    }

    /**
     * Whether the owning terminal editor tab should close automatically when
     * the backing session exits or disconnects. Default: derived from
     * {@link #capabilities()} — true iff CLOSE_TAB_ON_SESSION_END is declared.
     *
     * <p>Existing implementations may continue overriding this directly; new
     * implementations should declare the capability instead.
     */
    default boolean closeTabOnSessionEnd() {
        return capabilities().has(TerminalCapability.CLOSE_TAB_ON_SESSION_END);
    }

    /**
     * Create a new terminal session. May show UI to collect parameters
     * (e.g., host picker for SSH). Returns null if the user cancels.
     *
     * @param context provides access to project and application services
     * @return a connected TtyConnector, or null if cancelled
     */
    @Nullable TtyConnector createSession(@NotNull SessionContext context);

    /**
     * Context passed to createSession. Wraps project and app services
     * without requiring plugins to depend on IntelliJ Platform directly.
     */
    interface SessionContext {
        /** The working directory to start the session in (for local PTY). */
        @Nullable String getWorkingDirectory();
    }
}
```

- [ ] **Step 2: Update SDK BUILD.bazel to depend on core (for `TerminalCapabilities`)**

Wait — that's a circular dependency. Core depends on SDK; SDK can't depend on core. Resolution: move `TerminalCapability` and `TerminalCapabilities` to `sdk/`, OR keep them in `core.terminal.domain` and have the SDK SPI return a JDK type instead.

Choose: **move the two value-object classes to `sdk/`**. They're pure JDK, used by the SPI surface. Move:

- `core/src/com/termlab/core/terminal/domain/TerminalCapability.java` → `sdk/src/com/termlab/sdk/TerminalCapability.java`
- `core/src/com/termlab/core/terminal/domain/TerminalCapabilities.java` → `sdk/src/com/termlab/sdk/TerminalCapabilities.java`

Update package declarations from `com.termlab.core.terminal.domain` to `com.termlab.sdk`. Update `core/test/com/termlab/core/terminal/domain/TerminalCapabilitiesTest.java`'s package and imports to match (test stays in `core/test/...` but imports the new SDK location). Or move the test into `sdk/test/...` if SDK has a test setup. **Inspect `sdk/BUILD.bazel` first**; if no `sdk_test_lib` exists, leave the test in `core/test/` with updated imports.

```bash
ls /Users/dustin/projects/TermLab/sdk/test/ 2>/dev/null || echo "no sdk test directory"
cat /Users/dustin/projects/TermLab/sdk/BUILD.bazel
```

- [ ] **Step 3: Move the two classes and update imports across the codebase**

```bash
mkdir -p sdk/src/com/termlab/sdk
mv core/src/com/termlab/core/terminal/domain/TerminalCapability.java \
   sdk/src/com/termlab/sdk/TerminalCapability.java
mv core/src/com/termlab/core/terminal/domain/TerminalCapabilities.java \
   sdk/src/com/termlab/sdk/TerminalCapabilities.java
```

Edit both moved files: change the first line from `package com.termlab.core.terminal.domain;` to `package com.termlab.sdk;`.

Edit `core/test/com/termlab/core/terminal/domain/TerminalCapabilitiesTest.java`: change package to `package com.termlab.core.terminal.domain;` (still in core/test, even though the SUT moved); import the SDK types via `import com.termlab.sdk.TerminalCapability;` and `import com.termlab.sdk.TerminalCapabilities;`. The static `import static com.termlab.core.terminal.domain.TerminalCapability.*;` becomes `import static com.termlab.sdk.TerminalCapability.*;`.

Edit `core/src/com/termlab/core/terminal/domain/OpenError.java`: change `import com.termlab.core.terminal.domain.TerminalCapability` (it's same-package — actually no import needed). The reference is `TerminalCapability` — change to `import com.termlab.sdk.TerminalCapability;` at the top of `OpenError.java`. Also update `OpenErrorTest.java`.

Search and replace all references:

```bash
grep -rn "com.termlab.core.terminal.domain.TerminalCapability" \
    core/ sdk/ plugins/ 2>/dev/null
```

Update each match: replace `com.termlab.core.terminal.domain.TerminalCapability` with `com.termlab.sdk.TerminalCapability` (and the same for `TerminalCapabilities`).

- [ ] **Step 4: Compile-check**

```bash
bash bazel.cmd build //termlab/sdk //termlab/core:core_test_lib
```

Expected: BUILD SUCCESS.

- [ ] **Step 5: Run all domain tests via IntelliJ**

`ResultTest`, `TerminalSessionIdTest`, `TerminalProviderIdTest`, `TerminalCapabilitiesTest`, `TerminalTitleTest`, `TerminalSessionDescriptorTest`, `OpenErrorTest`, `HexagonalBoundariesTest`. All should pass.

- [ ] **Step 6: Commit**

```bash
git add sdk/src/com/termlab/sdk/TerminalCapability.java \
        sdk/src/com/termlab/sdk/TerminalCapabilities.java \
        sdk/src/com/termlab/sdk/TerminalSessionProvider.java \
        core/test/com/termlab/core/terminal/domain/TerminalCapabilitiesTest.java \
        core/src/com/termlab/core/terminal/domain/OpenError.java \
        core/test/com/termlab/core/terminal/domain/OpenErrorTest.java
git rm core/src/com/termlab/core/terminal/domain/TerminalCapability.java \
       core/src/com/termlab/core/terminal/domain/TerminalCapabilities.java 2>/dev/null || true
git commit -m "$(cat <<'EOF'
feat(sdk): add capabilities() to TerminalSessionProvider

Plan #1 Task 11. Non-breaking default method returning empty caps.
closeTabOnSessionEnd() now derives from CLOSE_TAB_ON_SESSION_END
capability for forward compatibility while remaining overridable.
TerminalCapability/TerminalCapabilities moved from core.domain to sdk
to keep SDK self-contained; SPI surface stays small.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 12: Application — `InMemoryTerminalSessionProviderRegistry` and `TerminalSessionLifecycleService`

**Pattern:** Application Service / Session Registry. The lifecycle service owns the live-session table; the provider registry owns the registered-providers table. Both are pure-JVM classes — no IntelliJ.

**Why:** the lifecycle service replaces the `SharedTerminalSession` ownership currently inside `TermLabTerminalVirtualFile`. Moving it out of the virtual file is the single largest unbundling in this plan.

**Files:**
- Create: `core/src/com/termlab/core/terminal/application/InMemoryTerminalSessionProviderRegistry.java`
- Create: `core/src/com/termlab/core/terminal/application/TerminalSessionLifecycleService.java`
- Create: `core/test/com/termlab/core/terminal/application/InMemoryTerminalSessionProviderRegistryTest.java`
- Create: `core/test/com/termlab/core/terminal/application/TerminalSessionLifecycleServiceTest.java`
- Create: `core/test/com/termlab/core/terminal/testfixtures/FakeTerminalSessionProvider.java`
- Create: `core/test/com/termlab/core/terminal/testfixtures/FakeTerminalSessionHandle.java`
- Create: `core/test/com/termlab/core/terminal/testfixtures/FakeTerminalEventPublisher.java`

- [ ] **Step 1: Create the test fixtures (reusable fakes)**

`core/test/com/termlab/core/terminal/testfixtures/FakeTerminalSessionProvider.java`:

```java
package com.termlab.core.terminal.testfixtures;

import com.jediterm.terminal.TtyConnector;
import com.termlab.sdk.TerminalCapabilities;
import com.termlab.sdk.TerminalSessionProvider;
import org.jetbrains.annotations.NotNull;
import org.jetbrains.annotations.Nullable;

import javax.swing.Icon;

/**
 * In-memory fake provider for tests. Returns a no-op connector or null
 * (configurable to simulate cancellation).
 */
public final class FakeTerminalSessionProvider implements TerminalSessionProvider {

    private final String id;
    private final TerminalCapabilities capabilities;
    private final boolean returnsNullOnCreate;

    public FakeTerminalSessionProvider(String id) {
        this(id, TerminalCapabilities.none(), false);
    }

    public FakeTerminalSessionProvider(String id, TerminalCapabilities capabilities, boolean returnsNullOnCreate) {
        this.id = id;
        this.capabilities = capabilities;
        this.returnsNullOnCreate = returnsNullOnCreate;
    }

    @Override public @NotNull String getId() { return id; }
    @Override public @NotNull String getDisplayName() { return id; }
    @Override public @Nullable Icon getIcon() { return null; }
    @Override public boolean canQuickOpen() { return true; }
    @Override public @NotNull TerminalCapabilities capabilities() { return capabilities; }

    @Override
    public @Nullable TtyConnector createSession(@NotNull SessionContext context) {
        if (returnsNullOnCreate) return null;
        // Tests using this fixture either don't call createSession or supply
        // a stubbed TerminalSessionFactory that ignores the connector.
        throw new UnsupportedOperationException(
            "FakeTerminalSessionProvider.createSession should not be invoked in unit tests; "
            + "use a stubbed TerminalSessionFactory instead.");
    }
}
```

`core/test/com/termlab/core/terminal/testfixtures/FakeTerminalSessionHandle.java`:

```java
package com.termlab.core.terminal.testfixtures;

import com.termlab.core.terminal.domain.TerminalSessionId;
import com.termlab.core.terminal.domain.TerminalTitle;
import com.termlab.core.terminal.ports.TerminalLifecycleEvent;
import com.termlab.core.terminal.ports.TerminalSessionHandle;

import java.util.ArrayList;
import java.util.List;
import java.util.function.Consumer;

public final class FakeTerminalSessionHandle implements TerminalSessionHandle {

    private final TerminalSessionId sessionId;
    private final List<byte[]> writes = new ArrayList<>();
    private final List<int[]> resizes = new ArrayList<>();
    private final List<Consumer<TerminalLifecycleEvent.ExitReason>> exitHandlers = new ArrayList<>();
    private TerminalTitle title;
    private String cwd;
    private boolean manualOverride;
    private boolean disposed;

    public FakeTerminalSessionHandle(TerminalSessionId sessionId, TerminalTitle title) {
        this.sessionId = sessionId;
        this.title = title;
    }

    @Override public TerminalSessionId sessionId() { return sessionId; }
    @Override public void write(byte[] data) { writes.add(data); }
    @Override public void resize(int rows, int columns) { resizes.add(new int[]{rows, columns}); }
    @Override public String currentWorkingDirectory() { return cwd; }
    @Override public TerminalTitle currentTitle() { return title; }
    @Override public void overrideTitle(TerminalTitle t) { this.title = t; this.manualOverride = true; }
    @Override public boolean hasManualTitleOverride() { return manualOverride; }
    @Override public void onExit(Consumer<TerminalLifecycleEvent.ExitReason> handler) { exitHandlers.add(handler); }
    @Override public int awaitExit() { return 0; /* Tests drive exits via simulateExit(). */ }
    @Override public void dispose() { disposed = true; }
    @Override public boolean isDisposed() { return disposed; }

    public List<byte[]> writes() { return writes; }
    public List<int[]> resizes() { return resizes; }
    public void simulateExit(TerminalLifecycleEvent.ExitReason reason) {
        for (var handler : exitHandlers) handler.accept(reason);
    }
}
```

`core/test/com/termlab/core/terminal/testfixtures/FakeTerminalEventPublisher.java`:

```java
package com.termlab.core.terminal.testfixtures;

import com.termlab.core.terminal.ports.TerminalEventPublisher;
import com.termlab.core.terminal.ports.TerminalLifecycleEvent;

import java.util.ArrayList;
import java.util.List;

public final class FakeTerminalEventPublisher implements TerminalEventPublisher {

    private final List<TerminalLifecycleEvent> events = new ArrayList<>();

    @Override
    public void publish(TerminalLifecycleEvent event) {
        events.add(event);
    }

    public List<TerminalLifecycleEvent> events() {
        return events;
    }

    public TerminalLifecycleEvent last() {
        if (events.isEmpty()) throw new AssertionError("no events published");
        return events.get(events.size() - 1);
    }
}
```

- [ ] **Step 2: Write `InMemoryTerminalSessionProviderRegistryTest.java`**

```java
package com.termlab.core.terminal.application;

import com.termlab.core.terminal.domain.TerminalProviderId;
import com.termlab.core.terminal.testfixtures.FakeTerminalSessionProvider;
import com.termlab.sdk.TerminalSessionProvider;
import org.junit.jupiter.api.Test;

import java.util.Optional;

import static org.junit.jupiter.api.Assertions.*;

final class InMemoryTerminalSessionProviderRegistryTest {

    private final InMemoryTerminalSessionProviderRegistry registry =
        new InMemoryTerminalSessionProviderRegistry();

    @Test
    void registerThenFindRoundTrips() {
        TerminalSessionProvider p = new FakeTerminalSessionProvider("com.termlab.local-pty");
        registry.register(p);
        Optional<TerminalSessionProvider> found = registry.find(TerminalProviderId.of("com.termlab.local-pty"));
        assertTrue(found.isPresent());
        assertSame(p, found.get());
    }

    @Test
    void findUnknownReturnsEmpty() {
        assertTrue(registry.find(TerminalProviderId.of("com.termlab.nobody")).isEmpty());
    }

    @Test
    void registerTwiceFailsFast() {
        registry.register(new FakeTerminalSessionProvider("com.termlab.local-pty"));
        assertThrows(IllegalStateException.class,
            () -> registry.register(new FakeTerminalSessionProvider("com.termlab.local-pty")));
    }

    @Test
    void allReturnsRegisteredProviders() {
        registry.register(new FakeTerminalSessionProvider("a"));
        registry.register(new FakeTerminalSessionProvider("b"));
        assertEquals(2, registry.all().size());
    }
}
```

- [ ] **Step 3: Write `TerminalSessionLifecycleServiceTest.java`**

```java
package com.termlab.core.terminal.application;

import com.termlab.core.terminal.domain.TerminalSessionId;
import com.termlab.core.terminal.domain.TerminalTitle;
import com.termlab.core.terminal.ports.TerminalLifecycleEvent;
import com.termlab.core.terminal.ports.TerminalSessionHandle;
import com.termlab.core.terminal.testfixtures.FakeTerminalEventPublisher;
import com.termlab.core.terminal.testfixtures.FakeTerminalSessionHandle;
import org.junit.jupiter.api.Test;

import java.util.Optional;

import static org.junit.jupiter.api.Assertions.*;

final class TerminalSessionLifecycleServiceTest {

    private final FakeTerminalEventPublisher events = new FakeTerminalEventPublisher();
    private final TerminalSessionLifecycleService service =
        new TerminalSessionLifecycleService(events);

    @Test
    void registerThenGetRoundTrips() {
        TerminalSessionId id = TerminalSessionId.newId();
        TerminalSessionHandle h = new FakeTerminalSessionHandle(id, TerminalTitle.of("zsh"));
        service.register(id, h);
        Optional<TerminalSessionHandle> got = service.get(id);
        assertTrue(got.isPresent());
        assertSame(h, got.get());
    }

    @Test
    void registerTwiceFailsFast() {
        TerminalSessionId id = TerminalSessionId.newId();
        TerminalSessionHandle h = new FakeTerminalSessionHandle(id, TerminalTitle.of("zsh"));
        service.register(id, h);
        assertThrows(IllegalStateException.class, () -> service.register(id, h));
    }

    @Test
    void releaseRemovesAndDisposes() {
        TerminalSessionId id = TerminalSessionId.newId();
        FakeTerminalSessionHandle h = new FakeTerminalSessionHandle(id, TerminalTitle.of("zsh"));
        service.register(id, h);
        service.release(id, TerminalLifecycleEvent.ExitReason.USER_CLOSED);
        assertTrue(service.get(id).isEmpty());
        assertTrue(h.isDisposed());
    }

    @Test
    void releasePublishesClosedEvent() {
        TerminalSessionId id = TerminalSessionId.newId();
        service.register(id, new FakeTerminalSessionHandle(id, TerminalTitle.of("zsh")));
        service.release(id, TerminalLifecycleEvent.ExitReason.USER_CLOSED);
        assertTrue(events.last() instanceof TerminalLifecycleEvent.TerminalClosed closed
            && closed.sessionId().equals(id)
            && closed.reason() == TerminalLifecycleEvent.ExitReason.USER_CLOSED);
    }

    @Test
    void releaseUnknownIsIdempotentNoOp() {
        TerminalSessionId id = TerminalSessionId.newId();
        service.release(id, TerminalLifecycleEvent.ExitReason.USER_CLOSED);
        // No throw; no events published.
        assertTrue(events.events().isEmpty());
    }

    @Test
    void allReturnsActiveSessions() {
        TerminalSessionId a = TerminalSessionId.newId();
        TerminalSessionId b = TerminalSessionId.newId();
        service.register(a, new FakeTerminalSessionHandle(a, TerminalTitle.of("a")));
        service.register(b, new FakeTerminalSessionHandle(b, TerminalTitle.of("b")));
        assertEquals(2, service.activeSessionIds().size());
    }
}
```

- [ ] **Step 4: Verify failure**

```bash
bash bazel.cmd build //termlab/core:core_test_lib
```

Expected: BUILD FAILURE.

- [ ] **Step 5: Implement `InMemoryTerminalSessionProviderRegistry`**

```java
package com.termlab.core.terminal.application;

import com.termlab.core.terminal.domain.TerminalProviderId;
import com.termlab.core.terminal.ports.TerminalSessionProviderRegistry;
import com.termlab.sdk.TerminalSessionProvider;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Objects;
import java.util.Optional;

public final class InMemoryTerminalSessionProviderRegistry
    implements TerminalSessionProviderRegistry {

    private final Map<TerminalProviderId, TerminalSessionProvider> providers = new HashMap<>();

    @Override
    public synchronized void register(TerminalSessionProvider provider) {
        Objects.requireNonNull(provider, "provider");
        TerminalProviderId id = TerminalProviderId.of(provider.getId());
        if (providers.containsKey(id)) {
            throw new IllegalStateException("Provider already registered: " + id);
        }
        providers.put(id, provider);
    }

    @Override
    public synchronized Optional<TerminalSessionProvider> find(TerminalProviderId id) {
        return Optional.ofNullable(providers.get(id));
    }

    @Override
    public synchronized List<TerminalSessionProvider> all() {
        return new ArrayList<>(providers.values());
    }
}
```

- [ ] **Step 6: Implement `TerminalSessionLifecycleService`**

```java
package com.termlab.core.terminal.application;

import com.termlab.core.terminal.domain.TerminalSessionId;
import com.termlab.core.terminal.ports.TerminalEventPublisher;
import com.termlab.core.terminal.ports.TerminalLifecycleEvent;
import com.termlab.core.terminal.ports.TerminalSessionHandle;

import java.util.HashMap;
import java.util.HashSet;
import java.util.Map;
import java.util.Objects;
import java.util.Optional;
import java.util.Set;

/**
 * Owns the active-session table and publishes lifecycle events on
 * register/release. Replaces the {@code SharedTerminalSession} ownership
 * currently inside {@code TermLabTerminalVirtualFile}.
 */
public final class TerminalSessionLifecycleService {

    private final TerminalEventPublisher publisher;
    private final Map<TerminalSessionId, TerminalSessionHandle> sessions = new HashMap<>();

    public TerminalSessionLifecycleService(TerminalEventPublisher publisher) {
        this.publisher = Objects.requireNonNull(publisher, "publisher");
    }

    public synchronized void register(TerminalSessionId id, TerminalSessionHandle handle) {
        Objects.requireNonNull(id, "id");
        Objects.requireNonNull(handle, "handle");
        if (sessions.containsKey(id)) {
            throw new IllegalStateException("Session already registered: " + id);
        }
        sessions.put(id, handle);
    }

    public synchronized Optional<TerminalSessionHandle> get(TerminalSessionId id) {
        return Optional.ofNullable(sessions.get(id));
    }

    public synchronized Set<TerminalSessionId> activeSessionIds() {
        return new HashSet<>(sessions.keySet());
    }

    public void release(TerminalSessionId id, TerminalLifecycleEvent.ExitReason reason) {
        TerminalSessionHandle removed;
        synchronized (this) {
            removed = sessions.remove(id);
        }
        if (removed == null) {
            return;
        }
        try {
            removed.dispose();
        } finally {
            publisher.publish(new TerminalLifecycleEvent.TerminalClosed(id, reason));
        }
    }
}
```

- [ ] **Step 7: Run tests via IntelliJ**

Run both new test classes + `HexagonalBoundariesTest`. Expected: 10 + 7 tests pass.

- [ ] **Step 8: Commit**

```bash
git add core/src/com/termlab/core/terminal/application/ \
        core/test/com/termlab/core/terminal/application/ \
        core/test/com/termlab/core/terminal/testfixtures/
git commit -m "$(cat <<'EOF'
feat(core): add provider registry and session lifecycle service

Plan #1 Task 12. InMemoryTerminalSessionProviderRegistry implements
the registry port; TerminalSessionLifecycleService owns the active-
session table and publishes Closed events on release. Test fixtures
(FakeProvider, FakeHandle, FakePublisher) are reusable across later
tasks.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 13: Application — `OpenTerminalUseCase`, `CloseTerminalUseCase`, and `TerminalTabService` façade

**Pattern:** Use Case (single-method classes) plus Application Service Façade. The use cases are the testable seams; the façade is what actions call.

**Why:** the use case shape (input → `Result<output, error>`) is what makes domain/application code unit-testable and side-effect-explicit.

**Files:**
- Create: `core/src/com/termlab/core/terminal/application/OpenTerminalUseCase.java`
- Create: `core/src/com/termlab/core/terminal/application/OpenResult.java`
- Create: `core/src/com/termlab/core/terminal/application/CloseTerminalUseCase.java`
- Create: `core/src/com/termlab/core/terminal/application/TerminalTabService.java`
- Create: `core/test/com/termlab/core/terminal/application/OpenTerminalUseCaseTest.java`
- Create: `core/test/com/termlab/core/terminal/application/CloseTerminalUseCaseTest.java`
- Create: `core/test/com/termlab/core/terminal/testfixtures/StubTerminalSessionFactory.java`

- [ ] **Step 1: Create the stub session factory test fixture**

`core/test/com/termlab/core/terminal/testfixtures/StubTerminalSessionFactory.java`:

```java
package com.termlab.core.terminal.testfixtures;

import com.termlab.core.terminal.domain.TerminalSessionDescriptor;
import com.termlab.core.terminal.domain.TerminalSessionId;
import com.termlab.core.terminal.ports.TerminalSessionFactory;
import com.termlab.core.terminal.ports.TerminalSessionHandle;
import com.termlab.sdk.TerminalSessionProvider;

import java.util.function.BiFunction;

/**
 * Test factory that returns whatever {@link FakeTerminalSessionHandle} the
 * builder function produces. Lets tests inject specific handles to assert on.
 */
public final class StubTerminalSessionFactory implements TerminalSessionFactory {

    private final BiFunction<TerminalSessionId, TerminalSessionDescriptor, TerminalSessionHandle> builder;

    public StubTerminalSessionFactory(
        BiFunction<TerminalSessionId, TerminalSessionDescriptor, TerminalSessionHandle> builder
    ) {
        this.builder = builder;
    }

    @Override
    public TerminalSessionHandle create(
        TerminalSessionId sessionId,
        TerminalSessionDescriptor descriptor,
        TerminalSessionProvider provider,
        TerminalSessionProvider.SessionContext context
    ) {
        return builder.apply(sessionId, descriptor);
    }

    public static StubTerminalSessionFactory returningFakeHandle() {
        return new StubTerminalSessionFactory(
            (id, d) -> new FakeTerminalSessionHandle(id, d.displayTitle()));
    }
}
```

- [ ] **Step 2: Write `OpenTerminalUseCaseTest`**

```java
package com.termlab.core.terminal.application;

import com.termlab.core.terminal.domain.OpenError;
import com.termlab.core.terminal.domain.Result;
import com.termlab.core.terminal.domain.TerminalProviderId;
import com.termlab.core.terminal.domain.TerminalSessionDescriptor;
import com.termlab.core.terminal.domain.TerminalTitle;
import com.termlab.core.terminal.ports.TerminalLifecycleEvent;
import com.termlab.core.terminal.testfixtures.FakeTerminalEventPublisher;
import com.termlab.core.terminal.testfixtures.FakeTerminalSessionProvider;
import com.termlab.core.terminal.testfixtures.StubTerminalSessionFactory;
import org.junit.jupiter.api.Test;

import java.util.Map;

import static org.junit.jupiter.api.Assertions.*;

final class OpenTerminalUseCaseTest {

    private final FakeTerminalEventPublisher events = new FakeTerminalEventPublisher();
    private final InMemoryTerminalSessionProviderRegistry registry = new InMemoryTerminalSessionProviderRegistry();
    private final TerminalSessionLifecycleService lifecycle = new TerminalSessionLifecycleService(events);

    private OpenTerminalUseCase newUseCase() {
        return new OpenTerminalUseCase(registry, StubTerminalSessionFactory.returningFakeHandle(), lifecycle, events);
    }

    @Test
    void happyPathReturnsSessionId() {
        registry.register(new FakeTerminalSessionProvider("com.termlab.local-pty"));
        TerminalSessionDescriptor d = TerminalSessionDescriptor.localPty(
            TerminalTitle.of("zsh"), "/home/dustin", Map.of());
        Result<OpenResult, OpenError> result = newUseCase().execute(d, () -> "/home/dustin");
        assertTrue(result.isSuccess());
        assertNotNull(result.value().sessionId());
    }

    @Test
    void happyPathRegistersHandleInLifecycle() {
        registry.register(new FakeTerminalSessionProvider("com.termlab.local-pty"));
        TerminalSessionDescriptor d = TerminalSessionDescriptor.localPty(
            TerminalTitle.of("zsh"), "/home/dustin", Map.of());
        Result<OpenResult, OpenError> result = newUseCase().execute(d, () -> "/home/dustin");
        assertTrue(lifecycle.get(result.value().sessionId()).isPresent());
    }

    @Test
    void happyPathPublishesOpenedEvent() {
        registry.register(new FakeTerminalSessionProvider("com.termlab.local-pty"));
        TerminalSessionDescriptor d = TerminalSessionDescriptor.localPty(
            TerminalTitle.of("zsh"), "/home/dustin", Map.of());
        newUseCase().execute(d, () -> "/home/dustin");
        assertTrue(events.last() instanceof TerminalLifecycleEvent.TerminalOpened);
    }

    @Test
    void unknownProviderReturnsFailure() {
        TerminalSessionDescriptor d = new TerminalSessionDescriptor(
            TerminalProviderId.of("com.termlab.nobody"),
            TerminalTitle.of("zsh"), null, Map.of());
        Result<OpenResult, OpenError> result = newUseCase().execute(d, () -> null);
        assertTrue(result.isFailure());
        assertTrue(result.error() instanceof OpenError.UnknownProvider up
            && up.providerId().asString().equals("com.termlab.nobody"));
        assertTrue(events.events().isEmpty());
        assertTrue(lifecycle.activeSessionIds().isEmpty());
    }

    @Test
    void factoryRuntimeExceptionPropagates() {
        registry.register(new FakeTerminalSessionProvider("com.termlab.local-pty"));
        TerminalSessionDescriptor d = TerminalSessionDescriptor.localPty(
            TerminalTitle.of("zsh"), null, Map.of());
        OpenTerminalUseCase useCase = new OpenTerminalUseCase(
            registry,
            new StubTerminalSessionFactory((id, dd) -> { throw new RuntimeException("boom"); }),
            lifecycle, events);
        RuntimeException ex = assertThrows(RuntimeException.class,
            () -> useCase.execute(d, () -> null));
        assertEquals("boom", ex.getMessage());
    }
}
```

- [ ] **Step 3: Write `CloseTerminalUseCaseTest`**

```java
package com.termlab.core.terminal.application;

import com.termlab.core.terminal.domain.TerminalSessionId;
import com.termlab.core.terminal.domain.TerminalTitle;
import com.termlab.core.terminal.ports.TerminalLifecycleEvent;
import com.termlab.core.terminal.testfixtures.FakeTerminalEventPublisher;
import com.termlab.core.terminal.testfixtures.FakeTerminalSessionHandle;
import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.*;

final class CloseTerminalUseCaseTest {

    private final FakeTerminalEventPublisher events = new FakeTerminalEventPublisher();
    private final TerminalSessionLifecycleService lifecycle = new TerminalSessionLifecycleService(events);
    private final CloseTerminalUseCase useCase = new CloseTerminalUseCase(lifecycle);

    @Test
    void releasesAndPublishesEvent() {
        TerminalSessionId id = TerminalSessionId.newId();
        FakeTerminalSessionHandle h = new FakeTerminalSessionHandle(id, TerminalTitle.of("zsh"));
        lifecycle.register(id, h);
        useCase.execute(id, TerminalLifecycleEvent.ExitReason.USER_CLOSED);
        assertTrue(h.isDisposed());
        assertTrue(events.last() instanceof TerminalLifecycleEvent.TerminalClosed);
    }

    @Test
    void unknownIdIsIdempotent() {
        TerminalSessionId id = TerminalSessionId.newId();
        useCase.execute(id, TerminalLifecycleEvent.ExitReason.USER_CLOSED);
        assertTrue(events.events().isEmpty());
    }
}
```

- [ ] **Step 4: Verify failure**

```bash
bash bazel.cmd build //termlab/core:core_test_lib
```

Expected: BUILD FAILURE.

- [ ] **Step 5: Implement `OpenResult`**

```java
package com.termlab.core.terminal.application;

import com.termlab.core.terminal.domain.TerminalSessionId;
import com.termlab.core.terminal.ports.TerminalSessionHandle;

/** Successful outcome of {@link OpenTerminalUseCase}. */
public record OpenResult(TerminalSessionId sessionId, TerminalSessionHandle handle) {}
```

- [ ] **Step 6: Implement `OpenTerminalUseCase`**

```java
package com.termlab.core.terminal.application;

import com.termlab.core.terminal.domain.OpenError;
import com.termlab.core.terminal.domain.Result;
import com.termlab.core.terminal.domain.TerminalSessionDescriptor;
import com.termlab.core.terminal.domain.TerminalSessionId;
import com.termlab.core.terminal.ports.TerminalEventPublisher;
import com.termlab.core.terminal.ports.TerminalLifecycleEvent;
import com.termlab.core.terminal.ports.TerminalSessionFactory;
import com.termlab.core.terminal.ports.TerminalSessionHandle;
import com.termlab.core.terminal.ports.TerminalSessionProviderRegistry;
import com.termlab.sdk.TerminalSessionProvider;

import java.util.Objects;
import java.util.Optional;

/**
 * Open a terminal session given a descriptor.
 *
 * <p>Pure orchestration: registry lookup → factory create → lifecycle register
 * → publish opened event → return id. No IntelliJ, no JediTerm.
 */
public final class OpenTerminalUseCase {

    private final TerminalSessionProviderRegistry registry;
    private final TerminalSessionFactory factory;
    private final TerminalSessionLifecycleService lifecycle;
    private final TerminalEventPublisher events;

    public OpenTerminalUseCase(
        TerminalSessionProviderRegistry registry,
        TerminalSessionFactory factory,
        TerminalSessionLifecycleService lifecycle,
        TerminalEventPublisher events
    ) {
        this.registry = Objects.requireNonNull(registry, "registry");
        this.factory = Objects.requireNonNull(factory, "factory");
        this.lifecycle = Objects.requireNonNull(lifecycle, "lifecycle");
        this.events = Objects.requireNonNull(events, "events");
    }

    public Result<OpenResult, OpenError> execute(
        TerminalSessionDescriptor descriptor,
        TerminalSessionProvider.SessionContext context
    ) {
        Optional<TerminalSessionProvider> maybeProvider = registry.find(descriptor.providerId());
        if (maybeProvider.isEmpty()) {
            return Result.failure(new OpenError.UnknownProvider(descriptor.providerId()));
        }
        TerminalSessionProvider provider = maybeProvider.get();

        TerminalSessionId sessionId = TerminalSessionId.newId();
        TerminalSessionHandle handle = factory.create(sessionId, descriptor, provider, context);
        if (handle == null) {
            return Result.failure(OpenError.userCancelled());
        }

        lifecycle.register(sessionId, handle);
        events.publish(new TerminalLifecycleEvent.TerminalOpened(sessionId, descriptor));
        return Result.success(new OpenResult(sessionId, handle));
    }
}
```

- [ ] **Step 7: Implement `CloseTerminalUseCase`**

```java
package com.termlab.core.terminal.application;

import com.termlab.core.terminal.domain.TerminalSessionId;
import com.termlab.core.terminal.ports.TerminalLifecycleEvent;

import java.util.Objects;

public final class CloseTerminalUseCase {

    private final TerminalSessionLifecycleService lifecycle;

    public CloseTerminalUseCase(TerminalSessionLifecycleService lifecycle) {
        this.lifecycle = Objects.requireNonNull(lifecycle, "lifecycle");
    }

    public void execute(TerminalSessionId id, TerminalLifecycleEvent.ExitReason reason) {
        lifecycle.release(id, reason);
    }
}
```

- [ ] **Step 8: Implement `TerminalTabService`**

```java
package com.termlab.core.terminal.application;

import com.termlab.core.terminal.domain.OpenError;
import com.termlab.core.terminal.domain.Result;
import com.termlab.core.terminal.domain.TerminalSessionDescriptor;
import com.termlab.core.terminal.domain.TerminalSessionId;
import com.termlab.core.terminal.ports.TerminalLifecycleEvent;
import com.termlab.sdk.TerminalSessionProvider;

import java.util.Objects;
import java.util.Optional;

/**
 * Façade entry point for terminal-tab actions. Coordinates open/close use
 * cases and exposes the lifecycle service for adapters that need to query
 * active sessions.
 */
public final class TerminalTabService {

    private final OpenTerminalUseCase openUseCase;
    private final CloseTerminalUseCase closeUseCase;
    private final TerminalSessionLifecycleService lifecycle;

    public TerminalTabService(
        OpenTerminalUseCase openUseCase,
        CloseTerminalUseCase closeUseCase,
        TerminalSessionLifecycleService lifecycle
    ) {
        this.openUseCase = Objects.requireNonNull(openUseCase, "openUseCase");
        this.closeUseCase = Objects.requireNonNull(closeUseCase, "closeUseCase");
        this.lifecycle = Objects.requireNonNull(lifecycle, "lifecycle");
    }

    public Result<OpenResult, OpenError> open(
        TerminalSessionDescriptor descriptor,
        TerminalSessionProvider.SessionContext context
    ) {
        return openUseCase.execute(descriptor, context);
    }

    public void close(TerminalSessionId id, TerminalLifecycleEvent.ExitReason reason) {
        closeUseCase.execute(id, reason);
    }

    public Optional<com.termlab.core.terminal.ports.TerminalSessionHandle> handleOf(TerminalSessionId id) {
        return lifecycle.get(id);
    }
}
```

- [ ] **Step 9: Run tests via IntelliJ**

Run all `application/` tests + `HexagonalBoundariesTest`. Expected: pass.

- [ ] **Step 10: Commit**

```bash
git add core/src/com/termlab/core/terminal/application/ \
        core/test/com/termlab/core/terminal/application/ \
        core/test/com/termlab/core/terminal/testfixtures/StubTerminalSessionFactory.java
git commit -m "$(cat <<'EOF'
feat(core): add OpenTerminalUseCase, CloseTerminalUseCase, TerminalTabService

Plan #1 Task 13. Use cases are the testable seams; TabService is the
adapter-facing façade. All pure-JVM, no IntelliJ.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 14: Adapters/jediterm — relocate `OscTrackingTtyConnector` and add factory + handle

**Pattern:** Adapter (port implementation) + Decorator (existing). The JediTerm package owns everything that touches `JediTermWidget` and `TtyConnector`.

**Why:** the `application/` layer must never import JediTerm. The factory lives here, the handle lives here, and the existing OSC decorator is co-located.

**Files:**
- Move: `core/src/com/termlab/core/terminal/OscTrackingTtyConnector.java` → `core/src/com/termlab/core/terminal/adapters/jediterm/OscTrackingTtyConnector.java`
- Create: `core/src/com/termlab/core/terminal/adapters/jediterm/TerminalWidgetFactory.java`
- Create: `core/src/com/termlab/core/terminal/adapters/jediterm/JediTermSessionHandle.java`
- Create: `core/src/com/termlab/core/terminal/adapters/jediterm/JediTermSessionFactory.java`

- [ ] **Step 1: Relocate `OscTrackingTtyConnector`**

```bash
git mv core/src/com/termlab/core/terminal/OscTrackingTtyConnector.java \
       core/src/com/termlab/core/terminal/adapters/jediterm/OscTrackingTtyConnector.java
```

Edit the moved file: change the first line's package from `package com.termlab.core.terminal;` to `package com.termlab.core.terminal.adapters.jediterm;`. The class body is unchanged.

- [ ] **Step 2: Find all callers of `OscTrackingTtyConnector` and update imports**

```bash
grep -rln "import com.termlab.core.terminal.OscTrackingTtyConnector" core/ plugins/ 2>/dev/null
grep -rln "OscTrackingTtyConnector" core/ plugins/ 2>/dev/null | head
```

The current callers are inside `TermLabTerminalVirtualFile` (same package — uses unqualified name) and `TermLabTerminalEditor` (same package — same). After the move, both need explicit imports: `import com.termlab.core.terminal.adapters.jediterm.OscTrackingTtyConnector;`. Add those imports.

- [ ] **Step 3: Compile-check**

```bash
bash bazel.cmd build //termlab/core:core
```

Expected: BUILD SUCCESS.

- [ ] **Step 4: Create `TerminalWidgetFactory`**

`core/src/com/termlab/core/terminal/adapters/jediterm/TerminalWidgetFactory.java`:

```java
package com.termlab.core.terminal.adapters.jediterm;

import com.termlab.core.terminal.TermLabTerminalSettings;
import com.termlab.core.terminal.TermLabTerminalWidget;

/**
 * The single place {@link TermLabTerminalWidget} is constructed. Centralizing
 * widget creation keeps {@code application/} unaware of JediTerm types.
 *
 * <p>Note: {@code TermLabTerminalWidget} currently lives at the root of
 * {@code core/terminal/} and depends on IntelliJ's {@code Project}. A future
 * refactor may move it into {@code adapters/intellij/} (it bridges Swing UI
 * to JediTerm). For Plan #1 we leave it where it is and import it here.
 */
public final class TerminalWidgetFactory {

    private final TermLabTerminalSettings settings;
    // Project is required by TermLabTerminalWidget. We accept it through
    // a setter rather than the constructor so this factory can be instantiated
    // as an application service without a Project; callers (the IntelliJ
    // adapter) bind a Project per session.
    private com.intellij.openapi.project.Project project;

    public TerminalWidgetFactory(TermLabTerminalSettings settings) {
        this.settings = settings;
    }

    public void bindProject(com.intellij.openapi.project.Project project) {
        this.project = project;
    }

    public TermLabTerminalWidget create(com.intellij.openapi.fileEditor.impl.LightVirtualFile file) {
        if (project == null) {
            throw new IllegalStateException("Project must be bound before creating a widget");
        }
        return new TermLabTerminalWidget(settings, project, (com.termlab.core.terminal.TermLabTerminalVirtualFile) file);
    }
}
```

**Note (architectural compromise):** `TermLabTerminalWidget` imports `com.intellij.openapi.project.Project`, which means anyone who imports `TermLabTerminalWidget` transitively pulls in IntelliJ. This factory therefore imports IntelliJ — which would violate the `adapters/jediterm` rule against `com.intellij.*`. Resolution options:

1. **Keep `TerminalWidgetFactory` in `adapters/jediterm/` and add a *single* exception to ArchUnit** — `JediTermSessionFactory` and `TerminalWidgetFactory` may import `com.intellij.openapi.project.Project` because the current `TermLabTerminalWidget` requires it. Document the exception in `HexagonalBoundariesTest`.
2. **Move `TerminalWidgetFactory` and `JediTermSessionFactory` to `adapters/intellij/`** — accepts that "JediTerm widget creation needs IntelliJ Project" is fundamentally an intellij concern.

Choose **option 2** — it's cleaner. Move `TerminalWidgetFactory` to `adapters/intellij/` (revise the file path in this task's "Files" section accordingly):

`core/src/com/termlab/core/terminal/adapters/intellij/TerminalWidgetFactory.java`:

```java
package com.termlab.core.terminal.adapters.intellij;

import com.intellij.openapi.project.Project;
import com.termlab.core.terminal.TermLabTerminalSettings;
import com.termlab.core.terminal.TermLabTerminalVirtualFile;
import com.termlab.core.terminal.TermLabTerminalWidget;

/**
 * Constructs {@link TermLabTerminalWidget} instances. Lives in
 * {@code adapters/intellij/} (not {@code adapters/jediterm/}) because
 * {@code TermLabTerminalWidget} depends on {@code Project}.
 *
 * <p>The widget itself bridges Swing+JediTerm; the factory is where the
 * IntelliJ project context is bound.
 */
public final class TerminalWidgetFactory {

    private final TermLabTerminalSettings settings;
    private final Project project;

    public TerminalWidgetFactory(TermLabTerminalSettings settings, Project project) {
        this.settings = settings;
        this.project = project;
    }

    public TermLabTerminalWidget create(TermLabTerminalVirtualFile file) {
        return new TermLabTerminalWidget(settings, project, file);
    }
}
```

Adjust the section header file list above accordingly (this is the actual file location).

- [ ] **Step 5: Create `JediTermSessionHandle` (the live-session port impl)**

`core/src/com/termlab/core/terminal/adapters/jediterm/JediTermSessionHandle.java`:

```java
package com.termlab.core.terminal.adapters.jediterm;

import com.jediterm.terminal.TtyConnector;
import com.termlab.core.terminal.domain.TerminalSessionId;
import com.termlab.core.terminal.domain.TerminalTitle;
import com.termlab.core.terminal.ports.TerminalLifecycleEvent;
import com.termlab.core.terminal.ports.TerminalSessionHandle;

import javax.swing.JComponent;
import java.io.IOException;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.atomic.AtomicReference;
import java.util.function.Consumer;

/**
 * Implements {@link TerminalSessionHandle} by wrapping a {@link TtyConnector}
 * and a JediTerm widget. Additionally exposes {@link #swingComponent()} for
 * the IntelliJ FileEditor to embed — this is the single allowed cross-adapter
 * touchpoint between {@code adapters.jediterm} and {@code adapters.intellij}.
 */
public final class JediTermSessionHandle implements TerminalSessionHandle {

    private final TerminalSessionId sessionId;
    private final TtyConnector connector;
    private final JComponent component;
    private final AtomicReference<TerminalTitle> currentTitle;
    private final AtomicReference<String> currentCwd = new AtomicReference<>();
    private final List<Consumer<TerminalLifecycleEvent.ExitReason>> exitHandlers = new ArrayList<>();
    private volatile boolean manualOverride;
    private volatile boolean disposed;

    public JediTermSessionHandle(
        TerminalSessionId sessionId,
        TtyConnector connector,
        JComponent component,
        TerminalTitle initialTitle
    ) {
        this.sessionId = sessionId;
        this.connector = connector;
        this.component = component;
        this.currentTitle = new AtomicReference<>(initialTitle);
    }

    @Override public TerminalSessionId sessionId() { return sessionId; }

    @Override
    public void write(byte[] data) {
        try {
            connector.write(data);
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }

    @Override
    public void resize(int rows, int columns) {
        connector.resize(new com.jediterm.terminal.util.TermSize(columns, rows));
    }

    @Override public String currentWorkingDirectory() { return currentCwd.get(); }

    @Override public TerminalTitle currentTitle() { return currentTitle.get(); }

    @Override
    public void overrideTitle(TerminalTitle title) {
        currentTitle.set(title);
        manualOverride = true;
    }

    @Override public boolean hasManualTitleOverride() { return manualOverride; }

    public void updateCwdFromOsc(String cwd) {
        currentCwd.set(cwd);
    }

    public void updateTitleFromOsc(TerminalTitle title) {
        if (manualOverride) return;
        currentTitle.set(title);
    }

    @Override
    public synchronized void onExit(Consumer<TerminalLifecycleEvent.ExitReason> handler) {
        exitHandlers.add(handler);
    }

    void fireExit(TerminalLifecycleEvent.ExitReason reason) {
        List<Consumer<TerminalLifecycleEvent.ExitReason>> snapshot;
        synchronized (this) {
            snapshot = new ArrayList<>(exitHandlers);
        }
        for (var handler : snapshot) handler.accept(reason);
    }

    @Override
    public synchronized void dispose() {
        if (disposed) return;
        disposed = true;
        try {
            connector.close();
        } catch (Exception ignored) {
        }
    }

    @Override public boolean isDisposed() { return disposed; }

    /**
     * The single allowed cross-adapter accessor: lets
     * {@code adapters.intellij.TermLabTerminalEditor} mount the widget into
     * its {@code FileEditor.getComponent()}. ArchUnit explicitly permits this
     * one reference.
     */
    public JComponent swingComponent() {
        return component;
    }

    @Override
    public int awaitExit() throws InterruptedException {
        return connector.waitFor();
    }

    /** Package-private — the JediTerm-side OSC wrapper uses this; never exposed to intellij adapters. */
    TtyConnector connector() {
        return connector;
    }
}
```

- [ ] **Step 6: Create `JediTermSessionFactory` (port impl)**

This factory composes the widget (from `adapters/intellij/TerminalWidgetFactory`) with the OSC connector and wraps both in a `JediTermSessionHandle`. Because it depends on `TerminalWidgetFactory` (which lives in `adapters/intellij/`), this factory itself ALSO lives in `adapters/intellij/` — the cross-adapter dependency would otherwise be inverted. Adjust file path:

`core/src/com/termlab/core/terminal/adapters/intellij/JediTermSessionFactory.java`:

```java
package com.termlab.core.terminal.adapters.intellij;

import com.jediterm.terminal.TtyConnector;
import com.termlab.core.terminal.TermLabTerminalVirtualFile;
import com.termlab.core.terminal.adapters.jediterm.JediTermSessionHandle;
import com.termlab.core.terminal.adapters.jediterm.OscTrackingTtyConnector;
import com.termlab.core.terminal.domain.TerminalSessionDescriptor;
import com.termlab.core.terminal.domain.TerminalSessionId;
import com.termlab.core.terminal.domain.TerminalTitle;
import com.termlab.core.terminal.ports.TerminalEventPublisher;
import com.termlab.core.terminal.ports.TerminalLifecycleEvent;
import com.termlab.core.terminal.ports.TerminalSessionFactory;
import com.termlab.core.terminal.ports.TerminalSessionHandle;
import com.termlab.sdk.TerminalSessionProvider;

import java.util.Objects;

/**
 * Builds {@link JediTermSessionHandle} instances. Calls into
 * {@link TerminalWidgetFactory} for widget construction, wraps the provider's
 * raw connector in {@link OscTrackingTtyConnector}, and assembles the handle.
 *
 * <p>Lives in {@code adapters/intellij} (not {@code adapters/jediterm}) because
 * widget construction requires an IntelliJ {@code Project}.
 */
public final class JediTermSessionFactory implements TerminalSessionFactory {

    private final TerminalWidgetFactory widgetFactory;
    private final TerminalEventPublisher events;

    public JediTermSessionFactory(TerminalWidgetFactory widgetFactory, TerminalEventPublisher events) {
        this.widgetFactory = Objects.requireNonNull(widgetFactory, "widgetFactory");
        this.events = Objects.requireNonNull(events, "events");
    }

    @Override
    public TerminalSessionHandle create(
        TerminalSessionId sessionId,
        TerminalSessionDescriptor descriptor,
        TerminalSessionProvider provider,
        TerminalSessionProvider.SessionContext context
    ) {
        // The current TermLabTerminalWidget API expects a virtual-file backref.
        // For now, build a placeholder file; the full migration of the virtual-file
        // shape happens in Task 17 where the widget is updated to take a session id
        // instead. During Tasks 14–16, callers still pass the existing
        // TermLabTerminalVirtualFile constructor (descriptor-based).
        TermLabTerminalVirtualFile file =
            new TermLabTerminalVirtualFile(sessionId, descriptor);
        var widget = widgetFactory.create(file);

        TtyConnector raw = provider.createSession(context);
        if (raw == null) {
            return null; // user cancelled — handled by OpenTerminalUseCase
        }

        // The OSC connector pushes CWD/title updates into the handle.
        // Build the handle first with a placeholder, then wire the connector.
        // (The OSC connector wants a callback; we provide one that mutates the handle.)
        JediTermSessionHandle handle =
            new JediTermSessionHandle(sessionId, raw, widget.getComponent(), descriptor.displayTitle());
        TtyConnector wrapped = new OscTrackingTtyConnector(
            raw,
            handle::updateCwdFromOsc,
            newTitle -> {
                if (newTitle != null && !newTitle.isBlank()) {
                    handle.updateTitleFromOsc(TerminalTitle.of(newTitle));
                    events.publish(new TerminalLifecycleEvent.TerminalTitleChanged(
                        sessionId, handle.currentTitle()));
                }
            }
        );

        // Replace the connector reference inside the handle by reflection or
        // by extending the handle to support late binding. Cleanest: pass the
        // wrapped connector in directly. Refactor: build wrapped first, then handle.
        // The current ordering above is incorrect. Correct it:
        throw new UnsupportedOperationException(
            "Refactor pending — see comment. Implementing properly in Step 7.");
    }
}
```

The above intentionally throws to force you to read Step 7, which gives the corrected implementation. Replace the body with the version below.

- [ ] **Step 7: Replace `JediTermSessionFactory.create` body with the correct ordering**

The OSC connector needs callbacks that mutate the handle, *and* the handle needs the wrapped connector reference. Resolve by introducing the OSC callbacks first (capturing the future handle via `AtomicReference`), wrapping, building the handle with the wrapped connector, then publishing the handle into the AtomicReference for the OSC callbacks to find.

Replace `create()` body:

```java
@Override
public TerminalSessionHandle create(
    TerminalSessionId sessionId,
    TerminalSessionDescriptor descriptor,
    TerminalSessionProvider provider,
    TerminalSessionProvider.SessionContext context
) {
    TermLabTerminalVirtualFile file =
        new TermLabTerminalVirtualFile(sessionId, descriptor);
    var widget = widgetFactory.create(file);

    TtyConnector raw = provider.createSession(context);
    if (raw == null) {
        return null;
    }

    java.util.concurrent.atomic.AtomicReference<JediTermSessionHandle> handleRef =
        new java.util.concurrent.atomic.AtomicReference<>();

    TtyConnector wrapped = new OscTrackingTtyConnector(
        raw,
        cwd -> {
            JediTermSessionHandle h = handleRef.get();
            if (h != null) {
                h.updateCwdFromOsc(cwd);
                events.publish(new TerminalLifecycleEvent.TerminalCwdChanged(sessionId, cwd));
            }
        },
        newTitle -> {
            JediTermSessionHandle h = handleRef.get();
            if (h != null && newTitle != null && !newTitle.isBlank()) {
                h.updateTitleFromOsc(TerminalTitle.of(newTitle));
                events.publish(new TerminalLifecycleEvent.TerminalTitleChanged(
                    sessionId, h.currentTitle()));
            }
        }
    );

    JediTermSessionHandle handle = new JediTermSessionHandle(
        sessionId, wrapped, widget.getComponent(), descriptor.displayTitle());
    handleRef.set(handle);

    widget.createTerminalSession(wrapped);
    widget.start();

    return handle;
}
```

- [ ] **Step 8: Compile-check**

```bash
bash bazel.cmd build //termlab/core:core
```

Expected: BUILD SUCCESS. Any breakage means the existing `TermLabTerminalVirtualFile` constructor signature differs — see Task 17 where it gets the new `(TerminalSessionId, TerminalSessionDescriptor)` constructor. For now, **add a temporary new constructor** to `TermLabTerminalVirtualFile` (don't delete the old one yet — keep both):

In `core/src/com/termlab/core/terminal/TermLabTerminalVirtualFile.java`, add alongside the existing constructor:

```java
public TermLabTerminalVirtualFile(@NotNull TerminalSessionId sessionId,
                                  @NotNull TerminalSessionDescriptor descriptor) {
    super(descriptor.displayTitle().displayValue(), TermLabTerminalFileType.INSTANCE, "");
    this.sessionId = sessionId.asString();
    // Provider is resolved lazily through TerminalTabService now;
    // keep field non-null for compatibility with existing methods that
    // dereference it. The slim version in Task 17 removes this field.
    this.provider = null;
    this.descriptor = descriptor;
    putUserData(FileEditorManagerKeys.FORBID_PREVIEW_TAB, true);
    putUserData(FileEditorManagerKeys.FORBID_TAB_SPLIT, true);
}
```

Add `private final TerminalSessionDescriptor descriptor;` field (initialize to `null` in the old constructor for now), with a getter `public TerminalSessionDescriptor getDescriptor() { return descriptor; }`.

Add the imports:

```java
import com.termlab.core.terminal.domain.TerminalSessionDescriptor;
import com.termlab.core.terminal.domain.TerminalSessionId;
```

- [ ] **Step 9: Compile-check again**

```bash
bash bazel.cmd build //termlab/core:core
```

Expected: BUILD SUCCESS. The temporary dual-constructor virtual file compiles. Methods that use the old `provider` field still work because they're unreached during local-PTY-only operation when going through the new path.

- [ ] **Step 10: Commit**

```bash
git add core/src/com/termlab/core/terminal/adapters/ \
        core/src/com/termlab/core/terminal/TermLabTerminalVirtualFile.java
git rm core/src/com/termlab/core/terminal/OscTrackingTtyConnector.java 2>/dev/null || true
git commit -m "$(cat <<'EOF'
feat(core): add JediTermSessionHandle, JediTermSessionFactory, TerminalWidgetFactory

Plan #1 Task 14. Relocates OscTrackingTtyConnector to
adapters/jediterm/. JediTermSessionFactory + TerminalWidgetFactory live
in adapters/intellij/ because TermLabTerminalWidget requires Project.
TermLabTerminalVirtualFile gains a temporary dual constructor; the
slim version lands in Task 17.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 15: Adapters/intellij — `TerminalSessionMonitor`, `IntellijTerminalNotificationAdapter`, `IntellijTerminalEventPublisher`

**Pattern:** Observer (monitor watches connector for exit), Adapter (notification), and a simple in-process event bus implementation.

**Why:** these three are infrastructure utilities the IntelliJ side needs for the wiring done in Task 17. Building them up-front isolates the IntelliJ-specific logic (executor service, message bus) from the application services that consume them.

**Files:**
- Create: `core/src/com/termlab/core/terminal/adapters/intellij/TerminalSessionMonitor.java`
- Create: `core/src/com/termlab/core/terminal/adapters/intellij/IntellijTerminalNotificationAdapter.java`
- Create: `core/src/com/termlab/core/terminal/adapters/intellij/IntellijTerminalEventPublisher.java`

- [ ] **Step 1: Create `TerminalSessionMonitor`**

`core/src/com/termlab/core/terminal/adapters/intellij/TerminalSessionMonitor.java`:

```java
package com.termlab.core.terminal.adapters.intellij;

import com.intellij.openapi.application.ApplicationManager;
import com.intellij.openapi.diagnostic.Logger;
import com.termlab.core.terminal.adapters.jediterm.JediTermSessionHandle;
import com.termlab.core.terminal.application.TerminalSessionLifecycleService;
import com.termlab.core.terminal.domain.TerminalSessionId;
import com.termlab.core.terminal.ports.TerminalLifecycleEvent;

import java.util.Objects;

/**
 * Watches a JediTerm connector for exit and routes the result through
 * {@link TerminalSessionLifecycleService}. Replaces the exit-watcher thread
 * inside {@code TermLabTerminalVirtualFile.SharedTerminalSession}.
 *
 * <p>Pattern: Observer. The connector is the subject; the lifecycle service
 * is the observer (via the use case).
 */
public final class TerminalSessionMonitor {

    private static final Logger LOG = Logger.getInstance(TerminalSessionMonitor.class);

    private final TerminalSessionLifecycleService lifecycle;

    public TerminalSessionMonitor(TerminalSessionLifecycleService lifecycle) {
        this.lifecycle = Objects.requireNonNull(lifecycle, "lifecycle");
    }

    public void watch(TerminalSessionId sessionId,
                      com.termlab.core.terminal.ports.TerminalSessionHandle handle) {
        Thread watcher = new Thread(() -> {
            try {
                handle.awaitExit();
            } catch (InterruptedException ignored) {
                return;
            } catch (Throwable t) {
                LOG.error("Exit watcher failed for session " + sessionId, t);
                ApplicationManager.getApplication().invokeLater(
                    () -> lifecycle.release(sessionId, TerminalLifecycleEvent.ExitReason.MONITOR_FAILED));
                return;
            }
            ApplicationManager.getApplication().invokeLater(() -> {
                if (handle instanceof JediTermSessionHandle jt) {
                    jt.fireExit(TerminalLifecycleEvent.ExitReason.PROCESS_EXIT);
                }
                lifecycle.release(sessionId, TerminalLifecycleEvent.ExitReason.PROCESS_EXIT);
            });
        }, "TermLab-exit-watcher-" + sessionId.asString());
        watcher.setDaemon(true);
        watcher.start();
    }
}
```

- [ ] **Step 2: Create `IntellijTerminalNotificationAdapter`**

`core/src/com/termlab/core/terminal/adapters/intellij/IntellijTerminalNotificationAdapter.java`:

```java
package com.termlab.core.terminal.adapters.intellij;

import com.intellij.notification.Notification;
import com.intellij.notification.NotificationGroupManager;
import com.intellij.notification.NotificationType;
import com.intellij.openapi.project.Project;
import com.termlab.core.terminal.ports.TerminalNotificationPort;

import java.util.Objects;

public final class IntellijTerminalNotificationAdapter implements TerminalNotificationPort {

    private static final String GROUP_ID = "TermLab Terminal";

    private final Project project;

    public IntellijTerminalNotificationAdapter(Project project) {
        this.project = Objects.requireNonNull(project, "project");
    }

    @Override public void info(String message) { notify(message, NotificationType.INFORMATION); }
    @Override public void warn(String message) { notify(message, NotificationType.WARNING); }
    @Override public void error(String message) { notify(message, NotificationType.ERROR); }

    private void notify(String message, NotificationType type) {
        Notification n = NotificationGroupManager.getInstance()
            .getNotificationGroup(GROUP_ID)
            .createNotification(message, type);
        n.notify(project);
    }
}
```

**Note:** the notification group `TermLab Terminal` must be declared in `core/resources/META-INF/plugin.xml`. If it's not present yet, add to `<extensions defaultExtensionNs="com.intellij">`:

```xml
<notificationGroup id="TermLab Terminal" displayType="BALLOON"/>
```

Verify the file:

```bash
grep -n "notificationGroup" /Users/dustin/projects/TermLab/core/resources/META-INF/plugin.xml
```

If a similar group already exists with a different id, reuse that id and update the constant in the adapter.

- [ ] **Step 3: Create `IntellijTerminalEventPublisher`**

`core/src/com/termlab/core/terminal/adapters/intellij/IntellijTerminalEventPublisher.java`:

```java
package com.termlab.core.terminal.adapters.intellij;

import com.intellij.openapi.diagnostic.Logger;
import com.termlab.core.terminal.ports.TerminalEventPublisher;
import com.termlab.core.terminal.ports.TerminalLifecycleEvent;

import java.util.List;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.function.Consumer;

/**
 * Simple in-process event bus. Plan #1 has no subscribers in the production
 * graph (the subscriber port arrives in Plan #2); the bus exists so use
 * cases can publish events even if nothing listens.
 */
public final class IntellijTerminalEventPublisher implements TerminalEventPublisher {

    private static final Logger LOG = Logger.getInstance(IntellijTerminalEventPublisher.class);

    private final List<Consumer<TerminalLifecycleEvent>> subscribers = new CopyOnWriteArrayList<>();

    @Override
    public void publish(TerminalLifecycleEvent event) {
        for (var s : subscribers) {
            try {
                s.accept(event);
            } catch (Throwable t) {
                LOG.warn("Terminal event subscriber threw on event " + event, t);
            }
        }
    }

    /**
     * Adapter-only API for in-process subscribers. Plan #2 will introduce a
     * {@code TerminalEventSubscriberPort} that wraps this; Plan #1 leaves it
     * adapter-internal.
     */
    public void subscribe(Consumer<TerminalLifecycleEvent> handler) {
        subscribers.add(handler);
    }
}
```

- [ ] **Step 4: Compile-check**

```bash
bash bazel.cmd build //termlab/core:core
```

Expected: BUILD SUCCESS.

- [ ] **Step 5: Commit**

```bash
git add core/src/com/termlab/core/terminal/adapters/intellij/TerminalSessionMonitor.java \
        core/src/com/termlab/core/terminal/adapters/intellij/IntellijTerminalNotificationAdapter.java \
        core/src/com/termlab/core/terminal/adapters/intellij/IntellijTerminalEventPublisher.java \
        core/resources/META-INF/plugin.xml
git commit -m "$(cat <<'EOF'
feat(core): add IntelliJ adapter classes for monitor, notifications, events

Plan #1 Task 15. TerminalSessionMonitor extracts the exit-watcher thread.
IntellijTerminalNotificationAdapter wraps Notifications.Bus.
IntellijTerminalEventPublisher is a CopyOnWriteArrayList-backed bus
(no subscribers in Plan #1; subscriber port lands in Plan #2).

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 16: Adapters/intellij — relocate `LocalPtySessionProvider` and `TermLabTerminalEditorProvider`, declare local-PTY capabilities

**Pattern:** Adapter relocation — moving existing classes into their hexagonal home + updating their import surface.

**Files:**
- Move: `core/src/com/termlab/core/terminal/LocalPtySessionProvider.java` → `core/src/com/termlab/core/terminal/adapters/intellij/LocalPtySessionProvider.java`
- Move: `core/src/com/termlab/core/terminal/TermLabTerminalEditorProvider.java` → `core/src/com/termlab/core/terminal/adapters/intellij/TermLabTerminalEditorProvider.java`
- Modify: `core/resources/META-INF/plugin.xml` (update FQNs in `<fileEditorProvider>`, `<applicationService>`, etc. that reference these classes)

- [ ] **Step 1: Find references to these classes**

```bash
grep -rln "LocalPtySessionProvider" core/ plugins/ 2>/dev/null
grep -rln "TermLabTerminalEditorProvider" core/ plugins/ 2>/dev/null
grep -n "LocalPtySessionProvider\|TermLabTerminalEditorProvider" core/resources/META-INF/plugin.xml
```

- [ ] **Step 2: Relocate `LocalPtySessionProvider`**

```bash
git mv core/src/com/termlab/core/terminal/LocalPtySessionProvider.java \
       core/src/com/termlab/core/terminal/adapters/intellij/LocalPtySessionProvider.java
```

Edit the moved file's package: `package com.termlab.core.terminal.adapters.intellij;`.

Add the capability declaration. Override `capabilities()`:

```java
import com.termlab.sdk.TerminalCapabilities;
import com.termlab.sdk.TerminalCapability;
// ...
@Override
public @NotNull TerminalCapabilities capabilities() {
    return TerminalCapabilities.of(TerminalCapability.CLOSE_TAB_ON_SESSION_END);
}
```

Remove the `closeTabOnSessionEnd()` override if present — it now derives from the capability.

- [ ] **Step 3: Relocate `TermLabTerminalEditorProvider`**

```bash
git mv core/src/com/termlab/core/terminal/TermLabTerminalEditorProvider.java \
       core/src/com/termlab/core/terminal/adapters/intellij/TermLabTerminalEditorProvider.java
```

Edit the moved file's package.

- [ ] **Step 4: Update `core/resources/META-INF/plugin.xml`**

Find every reference like `implementation="com.termlab.core.terminal.LocalPtySessionProvider"` and `implementation="com.termlab.core.terminal.TermLabTerminalEditorProvider"` (or similar attributes — could be `serviceImplementation`, `class`, etc.). Update each to the new FQN:

- `com.termlab.core.terminal.LocalPtySessionProvider` → `com.termlab.core.terminal.adapters.intellij.LocalPtySessionProvider`
- `com.termlab.core.terminal.TermLabTerminalEditorProvider` → `com.termlab.core.terminal.adapters.intellij.TermLabTerminalEditorProvider`

- [ ] **Step 5: Compile-check + manually launch**

```bash
bash bazel.cmd build //termlab/core:core
bash bazel.cmd run //termlab:termlab_run
```

Expected: app launches, "New Terminal" still works (existing path; new TabService not yet wired).

- [ ] **Step 6: Commit**

```bash
git add core/src/com/termlab/core/terminal/adapters/intellij/LocalPtySessionProvider.java \
        core/src/com/termlab/core/terminal/adapters/intellij/TermLabTerminalEditorProvider.java \
        core/resources/META-INF/plugin.xml
git commit -m "$(cat <<'EOF'
refactor(core): relocate LocalPtySessionProvider and TermLabTerminalEditorProvider

Plan #1 Task 16. Both move to adapters/intellij/. Local PTY provider
declares CLOSE_TAB_ON_SESSION_END via capabilities(). plugin.xml
references updated to new FQNs.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 17: Wire `TerminalTabService` into IntelliJ — provider registration, action migration, slim virtual file

**Pattern:** Composition Root (TermLabStartupActivity) + Application Service registration. The IntelliJ-specific composition root assembles the production graph: registry, factories, lifecycle, use cases, façade, then registers each provider into the registry.

**Why:** without this task the new abstractions are dead code. After this task, "open new local PTY tab" routes through `TerminalTabService.open` rather than `new TermLabTerminalVirtualFile(...)`.

**Files:**
- Create: `core/src/com/termlab/core/terminal/adapters/intellij/TerminalSubsystem.java` (the composition root)
- Modify: `core/src/com/termlab/core/terminal/TermLabStartupActivity.java` *or equivalent registration site* (add provider registration)
- Modify: `core/src/com/termlab/core/terminal/TermLabTerminalVirtualFile.java` (slim down)
- Modify: `core/src/com/termlab/core/terminal/TermLabEditorTabTitleProvider.java` (read title from session handle, not virtual file)
- Modify: `core/src/com/termlab/core/actions/NewTerminalTabAction.java` (route via `TerminalTabService`)
- Modify: any other in-core caller of `new TermLabTerminalVirtualFile(...)` (find with grep)

- [ ] **Step 1: Inventory in-core callers of the old constructor**

```bash
grep -rn "new TermLabTerminalVirtualFile" core/ 2>/dev/null
```

Each match must be replaced with a `TerminalTabService.open(descriptor)` call. Plugins are disabled, so plugin callers are irrelevant for this branch (their code still compiles via the dual constructor; SSH plugin will adopt the new path during the SSH plan).

- [ ] **Step 2: Create `TerminalSubsystem` composition root**

`core/src/com/termlab/core/terminal/adapters/intellij/TerminalSubsystem.java`:

```java
package com.termlab.core.terminal.adapters.intellij;

import com.intellij.openapi.components.Service;
import com.intellij.openapi.project.Project;
import com.termlab.core.terminal.TermLabTerminalSettings;
import com.termlab.core.terminal.application.CloseTerminalUseCase;
import com.termlab.core.terminal.application.InMemoryTerminalSessionProviderRegistry;
import com.termlab.core.terminal.application.OpenTerminalUseCase;
import com.termlab.core.terminal.application.TerminalSessionLifecycleService;
import com.termlab.core.terminal.application.TerminalTabService;
import com.termlab.core.terminal.ports.TerminalEventPublisher;
import com.termlab.core.terminal.ports.TerminalNotificationPort;
import com.termlab.core.terminal.ports.TerminalSessionFactory;
import com.termlab.core.terminal.ports.TerminalSessionProviderRegistry;
import com.termlab.sdk.TerminalSessionProvider;

/**
 * IntelliJ project-level service that hosts the terminal subsystem composition.
 * Wires the production object graph: registry → factory → lifecycle → use
 * cases → façade. Adapters fetch this service to access {@code tabService()}.
 */
@Service(Service.Level.PROJECT)
public final class TerminalSubsystem {

    private final TerminalSessionProviderRegistry registry;
    private final TerminalEventPublisher events;
    private final TerminalNotificationPort notifications;
    private final TerminalSessionLifecycleService lifecycle;
    private final TerminalSessionFactory factory;
    private final OpenTerminalUseCase openUseCase;
    private final CloseTerminalUseCase closeUseCase;
    private final TerminalTabService tabService;
    private final TerminalSessionMonitor monitor;

    public TerminalSubsystem(Project project) {
        this.registry = new InMemoryTerminalSessionProviderRegistry();
        IntellijTerminalEventPublisher pub = new IntellijTerminalEventPublisher();
        this.events = pub;
        this.notifications = new IntellijTerminalNotificationAdapter(project);
        this.lifecycle = new TerminalSessionLifecycleService(events);
        var widgetFactory = new TerminalWidgetFactory(new TermLabTerminalSettings(), project);
        this.factory = new JediTermSessionFactory(widgetFactory, events);
        this.openUseCase = new OpenTerminalUseCase(registry, factory, lifecycle, events);
        this.closeUseCase = new CloseTerminalUseCase(lifecycle);
        this.tabService = new TerminalTabService(openUseCase, closeUseCase, lifecycle);
        this.monitor = new TerminalSessionMonitor(lifecycle);

        // Register the local PTY provider. Plugins are disabled; the SSH/etc.
        // providers will be registered by their own plugin startup activities
        // when they're re-enabled in Plan #2 verification.
        registry.register(new LocalPtySessionProvider());
    }

    public TerminalTabService tabService() { return tabService; }
    public TerminalSessionProviderRegistry registry() { return registry; }
    public TerminalSessionLifecycleService lifecycle() { return lifecycle; }
    public TerminalSessionMonitor monitor() { return monitor; }
    public TerminalNotificationPort notifications() { return notifications; }
    public IntellijTerminalEventPublisher eventPublisher() {
        return (IntellijTerminalEventPublisher) events;
    }
}
```

Register the service in `core/resources/META-INF/plugin.xml` under `<extensions defaultExtensionNs="com.intellij">`:

```xml
<projectService serviceImplementation="com.termlab.core.terminal.adapters.intellij.TerminalSubsystem"/>
```

- [ ] **Step 3: Slim down `TermLabTerminalVirtualFile`**

Replace the full contents of `core/src/com/termlab/core/terminal/TermLabTerminalVirtualFile.java` with:

```java
package com.termlab.core.terminal;

import com.intellij.openapi.fileEditor.FileEditorManagerKeys;
import com.intellij.testFramework.LightVirtualFile;
import com.termlab.core.terminal.domain.TerminalSessionDescriptor;
import com.termlab.core.terminal.domain.TerminalSessionId;
import org.jetbrains.annotations.NotNull;

/**
 * IntelliJ {@code VirtualFile} that carries a {@link TerminalSessionId} and
 * its opening {@link TerminalSessionDescriptor}. Owns no lifecycle, no widget,
 * no thread — those live in {@code TerminalSubsystem} / {@code TerminalSessionLifecycleService}.
 */
public final class TermLabTerminalVirtualFile extends LightVirtualFile {

    private final TerminalSessionId sessionId;
    private final TerminalSessionDescriptor descriptor;

    public TermLabTerminalVirtualFile(@NotNull TerminalSessionId sessionId,
                                      @NotNull TerminalSessionDescriptor descriptor) {
        super(descriptor.displayTitle().displayValue(), TermLabTerminalFileType.INSTANCE, "");
        this.sessionId = sessionId;
        this.descriptor = descriptor;
        putUserData(FileEditorManagerKeys.FORBID_PREVIEW_TAB, true);
        putUserData(FileEditorManagerKeys.FORBID_TAB_SPLIT, true);
    }

    public @NotNull TerminalSessionId getSessionId() { return sessionId; }
    public @NotNull TerminalSessionDescriptor getDescriptor() { return descriptor; }

    @Override public boolean isWritable() { return false; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        return o instanceof TermLabTerminalVirtualFile other && sessionId.equals(other.sessionId);
    }

    @Override
    public int hashCode() { return sessionId.hashCode(); }
}
```

The old constructor, all setters, the inner `SharedTerminalSession` class, and every helper method are gone. The class drops from 299 lines to ~30.

- [ ] **Step 4: Update `TermLabEditorTabTitleProvider` to read title from the handle**

```bash
cat core/src/com/termlab/core/terminal/TermLabEditorTabTitleProvider.java | head -40
```

The provider currently reads `file.getTerminalTitle()` (a getter on the old virtual file). After Step 3 that getter is gone. Replace with a query against `TerminalSubsystem`:

```java
package com.termlab.core.terminal;

import com.intellij.openapi.fileEditor.impl.EditorTabTitleProvider;
import com.intellij.openapi.project.Project;
import com.intellij.openapi.vfs.VirtualFile;
import com.termlab.core.terminal.adapters.intellij.TerminalSubsystem;
import org.jetbrains.annotations.Nullable;

public final class TermLabEditorTabTitleProvider implements EditorTabTitleProvider {

    @Override
    public @Nullable String getEditorTabTitle(Project project, VirtualFile file) {
        if (!(file instanceof TermLabTerminalVirtualFile termFile)) return null;
        var subsystem = project.getService(TerminalSubsystem.class);
        var handle = subsystem.lifecycle().get(termFile.getSessionId());
        return handle.map(h -> h.currentTitle().displayValue())
                     .orElseGet(() -> termFile.getDescriptor().displayTitle().displayValue());
    }
}
```

- [ ] **Step 5: Migrate `NewTerminalTabAction`**

Open the action file (find with `grep -rln "class NewTerminalTabAction" core/`). Replace the body that does `new TermLabTerminalVirtualFile(title, provider)` with a `TerminalTabService.open(descriptor)` call:

```java
import com.termlab.core.terminal.adapters.intellij.TerminalSubsystem;
import com.termlab.core.terminal.application.TerminalTabService;
import com.termlab.core.terminal.domain.TerminalSessionDescriptor;
import com.termlab.core.terminal.domain.TerminalTitle;
import com.termlab.core.terminal.TermLabTerminalVirtualFile;

@Override
public void actionPerformed(@NotNull AnActionEvent e) {
    Project project = e.getProject();
    if (project == null) return;

    TerminalSubsystem subsystem = project.getService(TerminalSubsystem.class);
    TerminalTabService tabService = subsystem.tabService();

    TerminalSessionDescriptor descriptor = TerminalSessionDescriptor.localPty(
        TerminalTitle.of("Local"),
        System.getProperty("user.home"),
        java.util.Map.of()
    );

    var result = tabService.open(descriptor, descriptor::workingDirectory);
    if (result.isFailure()) {
        subsystem.notifications().error("Could not open terminal: " + result.error());
        return;
    }

    var sessionId = result.value().sessionId();
    var virtualFile = new TermLabTerminalVirtualFile(sessionId, descriptor);
    subsystem.monitor().watch(sessionId, result.value().handle());
    com.intellij.openapi.fileEditor.FileEditorManager.getInstance(project).openFile(virtualFile, true);
}
```

- [ ] **Step 6: Update `TermLabTerminalEditor` to consume the handle from the lifecycle service**

The editor today calls `file.acquireSession(project)`. After the slim virtual file, that method is gone. Replace with a lookup:

In `core/src/com/termlab/core/terminal/TermLabTerminalEditor.java`, the constructor (or equivalent factory call) does:

```java
TerminalSubsystem subsystem = project.getService(TerminalSubsystem.class);
var handle = subsystem.lifecycle().get(file.getSessionId())
    .orElseThrow(() -> new IllegalStateException("No session for " + file.getSessionId()));
JediTermSessionHandle jt = (JediTermSessionHandle) handle;
JComponent component = jt.swingComponent();
```

Then the editor mounts `component` as its `getComponent()` return. The old code that called `acquireSession`, started the widget, and registered listeners is gone — those steps now happen inside `JediTermSessionFactory.create`. The editor becomes mostly a JComponent holder. Actual listener-extraction is Task 19 (controllers).

- [ ] **Step 7: Compile-check**

```bash
bash bazel.cmd build //termlab/core:core
```

Expected: BUILD SUCCESS. Many compilation errors are likely on first attempt — the old virtual-file API surface (`acquireSession`, `getProvider`, `getCurrentWorkingDirectory`, `setManualTitleOverride`, etc.) is gone. Each compilation error names a caller that needs migration:

- Callers reading CWD/title from the file → read from the handle via `subsystem.lifecycle().get(sessionId)`
- Callers calling `acquireSession`/`release` → use `subsystem.tabService()` instead
- Callers checking `closeTabOnSessionEnd()` on the file's provider → look up the provider via `subsystem.registry().find(descriptor.providerId())`

Work through each error. The diff for this step may be 200+ lines; that's expected.

- [ ] **Step 8: Run TermLab and manually verify**

```bash
bash bazel.cmd run //termlab:termlab_run
```

Verify:
1. App launches, no errors in IDE log.
2. New Terminal opens a working tab.
3. Type a command, output appears.
4. Tab title updates when shell sets it (e.g., `printf '\033]0;mytitle\007'`).
5. CWD updates when shell uses OSC 7.
6. Closing the tab releases (no zombie threads — verify via thread dump if suspicious).
7. Shell exit closes the tab.

If any step fails, fix before committing. Do not commit broken behavior.

- [ ] **Step 9: Commit**

```bash
git add core/
git commit -m "$(cat <<'EOF'
refactor(core): slim TermLabTerminalVirtualFile, route opens through TerminalTabService

Plan #1 Task 17. The 299-line god class drops to ~30 lines (metadata
only). TerminalSubsystem composition root assembles the production
graph and registers LocalPtySessionProvider. NewTerminalTabAction routes
through TerminalTabService.open(descriptor). TermLabEditorTabTitleProvider
reads from the live handle. Editor consumes the handle via the
lifecycle service.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 18: Step 3 verification and memory write-back

**Pattern:** Verification phase. Step 3 (Tasks 5–17) of the spec is now complete — `TermLabTerminalVirtualFile` is dismantled. Confirm the architecture before moving to editor splitting.

- [ ] **Step 1: Run all automated tests via IntelliJ**

Run every test class in `core/test/com/termlab/core/terminal/`:
- `domain/ResultTest`
- `domain/TerminalSessionIdTest`
- `domain/TerminalProviderIdTest`
- `domain/TerminalCapabilitiesTest`
- `domain/TerminalTitleTest`
- `domain/TerminalSessionDescriptorTest`
- `domain/OpenErrorTest`
- `application/InMemoryTerminalSessionProviderRegistryTest`
- `application/TerminalSessionLifecycleServiceTest`
- `application/OpenTerminalUseCaseTest`
- `application/CloseTerminalUseCaseTest`
- `architecture/HexagonalBoundariesTest`

Plus the existing core tests (e.g., `TermLabTerminalWidgetTest`, `LocalPtySessionProviderTest`, `TerminalDroppedPathFormatterTest`, `TerminalSessionProviderClosePolicyTest`) — they may need import-path updates after relocations. Fix any breakage now.

Expected: all green.

- [ ] **Step 2: Manual smoke test for Step 3**

```bash
bash bazel.cmd run //termlab:termlab_run
```

Verify the eight points in the spec's Step 3 behavior list:
1. App starts.
2. New Terminal opens.
3. Input/output works.
4. Tab close releases resources.
5. OSC 0/2 title update visible.
6. OSC 7 CWD update visible.
7. Process exit closes tab (`exit` in shell).
8. No errors in IDE log relating to terminal subsystem.

- [ ] **Step 3: Write Step-3 memory entry**

Create `/Users/dustin/.claude/projects/-Users-dustin-projects-TermLab/memory/project_core_rework_step3_complete.md`:

```markdown
---
name: Core rework Plan #1 Step 3 — TermLabTerminalVirtualFile dismantled
description: Plan #1 Step 3 (Tasks 5–17) complete on the core-rework branch. TermLabTerminalVirtualFile shrunk from 299 to ~30 lines; lifecycle/widget/monitor logic moved to hexagonal packages. Architecture pattern record for drift prevention.
type: project
---

Plan #1 Step 3 (Tasks 5–17 of `docs/superpowers/plans/2026-04-28-core-terminal-hexagonal-foundation.md`) landed on branch `core-rework`. `TermLabTerminalVirtualFile` is now a metadata-only adapter; lifecycle, widget creation, exit watcher, OSC tracking, and tab-close behavior moved to focused services.

**Files created (with pattern + role):**

`core/src/com/termlab/core/terminal/domain/`:
- `Result.java` — Result Object (sealed `Success<T,E>` | `Failure<T,E>`); used for use-case returns
- `TerminalSessionId.java` — Value Object (UUID wrapper); typed session identity
- `TerminalProviderId.java` — Value Object (String wrapper); typed provider identity
- `TerminalTitle.java` — Value Object (String wrapper); display title
- `TerminalSessionDescriptor.java` — Value Object (record); immutable opening intent
- `OpenError.java` — Sealed Type; `UserCancelled` | `UnknownProvider` | `CapabilityMissing`

`sdk/src/com/termlab/sdk/` (moved from core.domain to keep SDK self-contained):
- `TerminalCapability.java` — Enum
- `TerminalCapabilities.java` — Value Object holding `Set<TerminalCapability>`

`core/src/com/termlab/core/terminal/ports/`:
- `TerminalSessionHandle.java` — Port; live-session control surface
- `TerminalSessionFactory.java` — Factory port
- `TerminalSessionProviderRegistry.java` — Registry port
- `TerminalEventPublisher.java` — Observer port
- `TerminalNotificationPort.java` — Adapter port
- `TerminalLifecycleEvent.java` — Sealed event protocol (`TerminalOpened` | `TerminalClosed` | `TerminalCwdChanged` | `TerminalTitleChanged`)

`core/src/com/termlab/core/terminal/application/`:
- `InMemoryTerminalSessionProviderRegistry.java` — Application Service / Registry; in-memory port impl
- `TerminalSessionLifecycleService.java` — Application Service / Session Registry; owns active-session table
- `OpenTerminalUseCase.java` — Use Case; `Result<OpenResult, OpenError>` orchestration
- `CloseTerminalUseCase.java` — Use Case; symmetric, idempotent close
- `OpenResult.java` — Value Object for use-case success
- `TerminalTabService.java` — Application Service / Façade; adapter-facing entry point

`core/src/com/termlab/core/terminal/adapters/jediterm/`:
- `OscTrackingTtyConnector.java` — Decorator (relocated from `core/terminal/`)
- `JediTermSessionHandle.java` — Adapter (port impl); the *single* class with `swingComponent()` cross-adapter touchpoint

`core/src/com/termlab/core/terminal/adapters/intellij/`:
- `TerminalWidgetFactory.java` — Factory adapter (Project-bound widget creation)
- `JediTermSessionFactory.java` — Adapter (port impl); composes widget + OSC connector + handle
- `TerminalSessionMonitor.java` — Observer adapter (extracts the exit-watcher thread)
- `IntellijTerminalNotificationAdapter.java` — Adapter (Notifications.Bus wrapper)
- `IntellijTerminalEventPublisher.java` — Adapter (CopyOnWriteArrayList event bus)
- `TerminalSubsystem.java` — Composition Root (project-level service); assembles the production graph
- `LocalPtySessionProvider.java` — Adapter (relocated from `core/terminal/`); declares `CLOSE_TAB_ON_SESSION_END` capability
- `TermLabTerminalEditorProvider.java` — Factory (relocated from `core/terminal/`)

**Files deleted:** none. (`TermLabMultiExecManager` removal is Plan #2 Step 1.)

**Files slimmed:**
- `TermLabTerminalVirtualFile.java`: 299 → ~30 lines, holds only `TerminalSessionId` + `TerminalSessionDescriptor`
- `TermLabEditorTabTitleProvider.java`: reads title from `TerminalSubsystem.lifecycle().get(sessionId).currentTitle()`

**Behavior changes:** none. Manual smoke confirmed local-PTY parity (open, IO, OSC title/CWD, exit).

**ArchUnit boundary tests:** all 7 rules passing. New code respects the dependency arrows mechanically.

**What this enables:**
- Plan #1 Step 4 (Tasks 19–21) extracts `TermLabTerminalEditor` controllers using the `TerminalSessionHandle` port now exposed.
- Plan #2 Step 1 wires extension-point ports (`TerminalInputWriter`, `EditorLayoutPort`, lifecycle subscriber) on top of the existing port infrastructure.
- The SSH plugin rework (post-core) replaces SSH's `SshSessionProvider` with a thin adapter that consumes `TerminalTabService.open(descriptor)`.

**Compromises noted:**
- `TerminalWidgetFactory` and `JediTermSessionFactory` live in `adapters/intellij/` (not `adapters/jediterm/`) because `TermLabTerminalWidget` requires `Project`. ArchUnit accepts this — `adapters/intellij/` is allowed to import IntelliJ.
```

Then add a pointer in MEMORY.md:

```markdown
- [Core rework Plan #1 Step 3 — TermLabTerminalVirtualFile dismantled](project_core_rework_step3_complete.md) — Plan #1 Step 3 done on core-rework branch; pattern record of every class created/moved/deleted, with role.
```

- [ ] **Step 4: Commit memory write**

The memory directory is outside the repo. The `memory_*.md` write IS the commit — no `git commit` needed for it. But add a tag or note to `MEMORY.md` (also outside the repo). The repo has nothing to commit at this step.

If the user wants a repo-side record too, append to `docs/architecture.md`:

```markdown
## Refactor history

- 2026-04-28 — Plan #1 Step 3: `TermLabTerminalVirtualFile` dismantled into `domain/application/ports/adapters` packages. See `docs/superpowers/plans/2026-04-28-core-terminal-hexagonal-foundation.md` Tasks 5–17.
```

If you appended to `docs/architecture.md`, commit:

```bash
git add docs/architecture.md
git commit -m "$(cat <<'EOF'
docs: record Plan #1 Step 3 completion in architecture history

Plan #1 Task 18. TermLabTerminalVirtualFile dismantled into hexagonal
packages with all 7 ArchUnit boundaries clean.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Tasks 19–23: Step 4 — Dismantle `TermLabTerminalEditor` into controllers

Step 4 of the spec extracts five controllers from the 326-line editor. Each controller owns one listener concern that the editor currently mixes. Pattern: **MVC Controller / Single-Responsibility class**, plus **Composition over Inheritance** at the editor level.

The five controllers are independent — they can be implemented in any order, but reviewing them as five separate commits makes the architectural intent obvious.

For each controller below: **read the corresponding section of the existing `TermLabTerminalEditor`**, identify the listener body / setup code, extract it into the controller class, and replace the inline code with a controller method call.

### Task 19: Extract `TerminalAppearanceController`

**Pattern:** MVC Controller. Owns appearance listener registration and theme/font sync into the JediTerm widget.

**Files:**
- Create: `core/src/com/termlab/core/terminal/adapters/intellij/controller/TerminalAppearanceController.java`
- Modify: `core/src/com/termlab/core/terminal/TermLabTerminalEditor.java` (remove appearance code, call controller)

- [ ] **Step 1: Identify the appearance code in `TermLabTerminalEditor.java`**

Search for any of the following patterns:

```bash
grep -n "Appearance\|theme\|font\|colorScheme" core/src/com/termlab/core/terminal/TermLabTerminalEditor.java
```

The relevant block typically:
- Subscribes to `LafManager.getInstance().addLafManagerListener(...)`, `EditorColorsManager.getInstance().getGlobalScheme()`, or similar
- Pushes new font/colors into the JediTerm widget when the listener fires

- [ ] **Step 2: Create the controller**

```java
package com.termlab.core.terminal.adapters.intellij.controller;

import com.intellij.ide.ui.LafManager;
import com.intellij.ide.ui.LafManagerListener;
import com.intellij.openapi.Disposable;
import com.intellij.openapi.application.ApplicationManager;
import com.intellij.openapi.editor.colors.EditorColorsListener;
import com.intellij.openapi.editor.colors.EditorColorsScheme;
import com.intellij.util.messages.MessageBusConnection;
import com.termlab.core.terminal.TermLabTerminalWidget;
import org.jetbrains.annotations.NotNull;

/**
 * Controller for terminal appearance (theme + colors + font) sync.
 *
 * <p>Pattern: MVC Controller. Subscribes to LAF and editor-color listeners;
 * pushes updates into the JediTerm widget. Replaces inline appearance
 * listener bodies inside {@code TermLabTerminalEditor}.
 *
 * <p>Disposed via the editor's {@link Disposable}.
 */
public final class TerminalAppearanceController implements Disposable {

    private final TermLabTerminalWidget widget;
    private final MessageBusConnection connection;

    public TerminalAppearanceController(@NotNull TermLabTerminalWidget widget) {
        this.widget = widget;
        this.connection = ApplicationManager.getApplication().getMessageBus().connect(this);
        connection.subscribe(LafManagerListener.TOPIC, (LafManagerListener) source -> applyAppearance());
        connection.subscribe(EditorColorsListener.TOPIC,
            (EditorColorsListener) (EditorColorsScheme s) -> applyAppearance());
        applyAppearance();
    }

    private void applyAppearance() {
        // Push current LAF/colors into the widget. Implementation mirrors
        // whatever the existing TermLabTerminalEditor did — copy that body
        // verbatim into this method.
        widget.applyAppearanceFromGlobalScheme();
    }

    @Override
    public void dispose() {
        connection.disconnect();
    }
}
```

If `TermLabTerminalWidget.applyAppearanceFromGlobalScheme()` doesn't exist, add it as a method on the widget that encapsulates the body of the original listener. (The widget already knows how to apply appearance; this just exposes it as a single call.)

- [ ] **Step 3: Replace the inline appearance code in the editor**

In `TermLabTerminalEditor.java`, replace the inline LAF/colors listener block with:

```java
this.appearanceController = new TerminalAppearanceController(widget);
Disposer.register(this, appearanceController);
```

Add a field: `private TerminalAppearanceController appearanceController;`.

- [ ] **Step 4: Compile + manually launch**

```bash
bash bazel.cmd build //termlab/core:core
bash bazel.cmd run //termlab:termlab_run
```

Verify: theme switch (Settings → Appearance → Theme) propagates into the open terminal tab. Open terminal, switch theme, terminal rerenders correctly.

- [ ] **Step 5: Commit**

```bash
git add core/src/com/termlab/core/terminal/adapters/intellij/controller/TerminalAppearanceController.java \
        core/src/com/termlab/core/terminal/TermLabTerminalEditor.java \
        core/src/com/termlab/core/terminal/TermLabTerminalWidget.java
git commit -m "$(cat <<'EOF'
refactor(core): extract TerminalAppearanceController

Plan #1 Task 19. MVC Controller for theme/font sync. Editor now
delegates appearance listener bodies to the controller.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

### Task 20: Extract `TerminalResizeController`, `TerminalDropPathController`, `TerminalFocusController`

These three controllers follow the same pattern as Task 19. Implement each, replace the inline editor code, manual-verify, commit. One per sub-task.

**Files (all under `core/src/com/termlab/core/terminal/adapters/intellij/controller/`):**
- `TerminalResizeController.java`
- `TerminalDropPathController.java`
- `TerminalFocusController.java`

- [ ] **Step 1: `TerminalResizeController`**

```java
package com.termlab.core.terminal.adapters.intellij.controller;

import com.intellij.openapi.Disposable;
import com.termlab.core.terminal.ports.TerminalSessionHandle;

import javax.swing.JComponent;
import java.awt.event.ComponentAdapter;
import java.awt.event.ComponentEvent;
import java.awt.event.ComponentListener;

/**
 * Controller for terminal resize. Listens on the editor component; pushes
 * (rows, columns) into the session handle when size changes.
 */
public final class TerminalResizeController implements Disposable {

    private final JComponent component;
    private final TerminalSessionHandle handle;
    private final ComponentListener listener;

    public TerminalResizeController(JComponent component, TerminalSessionHandle handle) {
        this.component = component;
        this.handle = handle;
        this.listener = new ComponentAdapter() {
            @Override public void componentResized(ComponentEvent e) { pushResize(); }
            @Override public void componentShown(ComponentEvent e) { pushResize(); }
        };
        component.addComponentListener(listener);
        pushResize();
    }

    private void pushResize() {
        // Mirror whatever the existing TermLabTerminalEditor resize logic does
        // (typically: convert pixel size to (rows, columns) using widget metrics
        // and call connector.resize). The handle.resize takes (rows, columns).
        // If the existing editor has helper code for this, move it here.
        int rows = Math.max(2, component.getHeight() / 16);
        int cols = Math.max(2, component.getWidth() / 8);
        handle.resize(rows, cols);
    }

    @Override
    public void dispose() {
        component.removeComponentListener(listener);
    }
}
```

Replace the inline resize code in `TermLabTerminalEditor` with:

```java
this.resizeController = new TerminalResizeController(rootComponent, handle);
Disposer.register(this, resizeController);
```

Compile + verify (resize the IDE window; terminal reflows without artifacts) + commit:

```bash
git add core/src/com/termlab/core/terminal/adapters/intellij/controller/TerminalResizeController.java \
        core/src/com/termlab/core/terminal/TermLabTerminalEditor.java
git commit -m "refactor(core): extract TerminalResizeController

Plan #1 Task 20.1.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

- [ ] **Step 2: `TerminalDropPathController`**

```java
package com.termlab.core.terminal.adapters.intellij.controller;

import com.intellij.openapi.Disposable;
import com.termlab.core.terminal.TerminalDroppedPathFormatter;
import com.termlab.core.terminal.ports.TerminalSessionHandle;

import javax.swing.JComponent;
import javax.swing.TransferHandler;
import java.awt.datatransfer.DataFlavor;
import java.io.File;
import java.nio.charset.StandardCharsets;
import java.util.List;

/**
 * Controller for file-path drag-and-drop into the terminal.
 *
 * <p>Pattern: MVC Controller wrapping a Swing {@link TransferHandler}.
 * Reuses {@link TerminalDroppedPathFormatter} for shell-aware quoting.
 */
public final class TerminalDropPathController implements Disposable {

    private final JComponent component;
    private final TransferHandler previousHandler;

    public TerminalDropPathController(JComponent component, TerminalSessionHandle handle) {
        this.component = component;
        this.previousHandler = component.getTransferHandler();
        component.setTransferHandler(new TransferHandler() {
            @Override public boolean canImport(TransferSupport s) {
                return s.isDataFlavorSupported(DataFlavor.javaFileListFlavor);
            }
            @Override public boolean importData(TransferSupport s) {
                if (!canImport(s)) return false;
                try {
                    @SuppressWarnings("unchecked")
                    List<File> files = (List<File>) s.getTransferable().getTransferData(DataFlavor.javaFileListFlavor);
                    StringBuilder sb = new StringBuilder();
                    for (File f : files) {
                        if (sb.length() > 0) sb.append(' ');
                        sb.append(TerminalDroppedPathFormatter.format(f.getAbsolutePath()));
                    }
                    handle.write(sb.toString().getBytes(StandardCharsets.UTF_8));
                    return true;
                } catch (Exception e) {
                    return false;
                }
            }
        });
    }

    @Override
    public void dispose() {
        component.setTransferHandler(previousHandler);
    }
}
```

Replace inline drop code in editor with controller construction. Verify drag-a-file-into-terminal pastes the formatted path. Commit:

```bash
git add core/src/com/termlab/core/terminal/adapters/intellij/controller/TerminalDropPathController.java \
        core/src/com/termlab/core/terminal/TermLabTerminalEditor.java
git commit -m "refactor(core): extract TerminalDropPathController

Plan #1 Task 20.2.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

- [ ] **Step 3: `TerminalFocusController`**

```java
package com.termlab.core.terminal.adapters.intellij.controller;

import com.intellij.openapi.Disposable;
import com.termlab.core.terminal.TermLabTerminalWidget;

import javax.swing.JComponent;
import java.awt.event.FocusAdapter;
import java.awt.event.FocusEvent;
import java.awt.event.FocusListener;

/**
 * Controller that forwards focus from the editor's root component into the
 * JediTerm widget so keystrokes reach the terminal immediately.
 */
public final class TerminalFocusController implements Disposable {

    private final JComponent component;
    private final FocusListener listener;

    public TerminalFocusController(JComponent component, TermLabTerminalWidget widget) {
        this.component = component;
        this.listener = new FocusAdapter() {
            @Override public void focusGained(FocusEvent e) {
                widget.requestFocusInTerminal();
            }
        };
        component.addFocusListener(listener);
    }

    @Override
    public void dispose() {
        component.removeFocusListener(listener);
    }
}
```

If `TermLabTerminalWidget.requestFocusInTerminal()` doesn't exist, add a delegate method that calls into JediTerm's `getCurrentSession().getTerminal().getTerminalPanel().requestFocusInWindow()` or whatever the current editor inlined. Replace inline editor focus code, verify (clicking the editor focuses the terminal), commit:

```bash
git add core/src/com/termlab/core/terminal/adapters/intellij/controller/TerminalFocusController.java \
        core/src/com/termlab/core/terminal/TermLabTerminalEditor.java \
        core/src/com/termlab/core/terminal/TermLabTerminalWidget.java
git commit -m "refactor(core): extract TerminalFocusController

Plan #1 Task 20.3.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

### Task 21: Extract `TerminalTabActionsFactory`

**Pattern:** Factory. Builds the list of `AnAction` shown in the tab's overflow/context menu. Currently inlined inside the editor constructor.

**Files:**
- Create: `core/src/com/termlab/core/terminal/adapters/intellij/controller/TerminalTabActionsFactory.java`
- Modify: `core/src/com/termlab/core/terminal/TermLabTerminalEditor.java`

- [ ] **Step 1: Identify the tab-action code**

```bash
grep -n "AnAction\|ActionGroup\|setTabActions" core/src/com/termlab/core/terminal/TermLabTerminalEditor.java
```

The relevant block builds an `ActionGroup` or `List<AnAction>` for the tab.

- [ ] **Step 2: Create the factory**

```java
package com.termlab.core.terminal.adapters.intellij.controller;

import com.intellij.openapi.actionSystem.AnAction;
import com.intellij.openapi.actionSystem.DefaultActionGroup;
import com.intellij.openapi.project.Project;
import com.termlab.core.terminal.ports.TerminalSessionHandle;

import java.util.List;

/**
 * Factory for the tab-bar action group shown on each terminal tab.
 *
 * <p>Pattern: Factory. Centralizes the actions in one place so adding /
 * removing actions doesn't require touching the editor constructor.
 */
public final class TerminalTabActionsFactory {

    public static DefaultActionGroup buildTabActions(Project project, TerminalSessionHandle handle) {
        DefaultActionGroup group = new DefaultActionGroup();
        for (AnAction action : actions(project, handle)) {
            group.add(action);
        }
        return group;
    }

    private static List<AnAction> actions(Project project, TerminalSessionHandle handle) {
        // Move whatever the existing editor inlined here. Typical contents:
        // rename, copy text, clear, close. Match the existing list verbatim.
        return List.of(
            // new RenameTerminalTabAction(project, handle),
            // new CopyTerminalSelectionAction(handle),
            // new ClearTerminalAction(handle),
            // new CloseTerminalTabAction(project, handle)
        );
    }
}
```

- [ ] **Step 3: Replace inline editor code**

```java
DefaultActionGroup tabActions = TerminalTabActionsFactory.buildTabActions(project, handle);
// pass tabActions wherever the editor previously set them
```

- [ ] **Step 4: Compile + verify**

Open the tab overflow/context menu — actions still present and functional.

- [ ] **Step 5: Commit**

```bash
git add core/src/com/termlab/core/terminal/adapters/intellij/controller/TerminalTabActionsFactory.java \
        core/src/com/termlab/core/terminal/TermLabTerminalEditor.java
git commit -m "refactor(core): extract TerminalTabActionsFactory

Plan #1 Task 21.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

### Task 22: Slim `TermLabTerminalEditor` and relocate to `adapters/intellij/`

**Pattern:** Composition over Inheritance. The editor now does *only* composition — wires the controllers and exposes the JComponent to IntelliJ.

**Files:**
- Move: `core/src/com/termlab/core/terminal/TermLabTerminalEditor.java` → `core/src/com/termlab/core/terminal/adapters/intellij/TermLabTerminalEditor.java`

- [ ] **Step 1: Relocate the file**

```bash
git mv core/src/com/termlab/core/terminal/TermLabTerminalEditor.java \
       core/src/com/termlab/core/terminal/adapters/intellij/TermLabTerminalEditor.java
```

Update package declaration. Update `core/resources/META-INF/plugin.xml` if the FQN is referenced there.

- [ ] **Step 2: Replace the body with the slim version**

```java
package com.termlab.core.terminal.adapters.intellij;

import com.intellij.openapi.fileEditor.FileEditor;
import com.intellij.openapi.fileEditor.FileEditorState;
import com.intellij.openapi.project.Project;
import com.intellij.openapi.util.Disposer;
import com.intellij.openapi.util.UserDataHolderBase;
import com.termlab.core.terminal.TermLabTerminalVirtualFile;
import com.termlab.core.terminal.adapters.intellij.controller.TerminalAppearanceController;
import com.termlab.core.terminal.adapters.intellij.controller.TerminalDropPathController;
import com.termlab.core.terminal.adapters.intellij.controller.TerminalFocusController;
import com.termlab.core.terminal.adapters.intellij.controller.TerminalResizeController;
import com.termlab.core.terminal.adapters.jediterm.JediTermSessionHandle;
import com.termlab.core.terminal.ports.TerminalSessionHandle;
import org.jetbrains.annotations.NotNull;
import org.jetbrains.annotations.Nullable;

import javax.swing.JComponent;
import java.beans.PropertyChangeListener;

/**
 * IntelliJ {@link FileEditor} for terminal tabs. Composition glue only:
 * looks up the live {@link TerminalSessionHandle}, mounts its widget, and
 * wires controllers. No business logic.
 */
public final class TermLabTerminalEditor extends UserDataHolderBase implements FileEditor {

    private final TermLabTerminalVirtualFile file;
    private final JComponent root;

    public TermLabTerminalEditor(@NotNull Project project,
                                 @NotNull TermLabTerminalVirtualFile file) {
        this.file = file;

        TerminalSubsystem subsystem = project.getService(TerminalSubsystem.class);
        TerminalSessionHandle handle = subsystem.lifecycle().get(file.getSessionId())
            .orElseThrow(() -> new IllegalStateException(
                "No live session for " + file.getSessionId() + " — did the open path use TerminalTabService?"));
        JediTermSessionHandle jt = (JediTermSessionHandle) handle;
        this.root = jt.swingComponent();

        // Wire controllers; each registers itself for disposal with this editor.
        Disposer.register(this, new TerminalAppearanceController(jt /* widget reference; see Task 19 */));
        Disposer.register(this, new TerminalResizeController(root, handle));
        Disposer.register(this, new TerminalDropPathController(root, handle));
        Disposer.register(this, new TerminalFocusController(root, jt /* widget reference; see Task 20 */));
        // Tab actions are wired separately via TerminalTabActionsFactory wherever the
        // FileEditorManager attaches them (this varies by IntelliJ-platform version).
    }

    @Override public @NotNull JComponent getComponent() { return root; }
    @Override public @Nullable JComponent getPreferredFocusedComponent() { return root; }
    @Override public @NotNull String getName() { return "TermLab Terminal"; }
    @Override public @NotNull FileEditorState getState(@NotNull com.intellij.openapi.fileEditor.FileEditorStateLevel level) { return FileEditorState.INSTANCE; }
    @Override public void setState(@NotNull FileEditorState state) {}
    @Override public boolean isModified() { return false; }
    @Override public boolean isValid() { return file.isValid(); }
    @Override public void addPropertyChangeListener(@NotNull PropertyChangeListener listener) {}
    @Override public void removePropertyChangeListener(@NotNull PropertyChangeListener listener) {}
    @Override public void dispose() {}
    @Override public @NotNull com.intellij.openapi.vfs.VirtualFile getFile() { return file; }
}
```

The `TerminalAppearanceController` and `TerminalFocusController` constructors above take a widget, not a handle — adjust their signatures to accept whatever widget reference they need (the `JediTermSessionHandle` exposes the widget via the `swingComponent`/internals; or extend the handle with `widget()` access for adapter-side use).

- [ ] **Step 3: Update `TermLabTerminalEditorProvider` if needed**

The provider's `createEditor(Project, VirtualFile)` returns `new TermLabTerminalEditor(project, (TermLabTerminalVirtualFile) file)`. Confirm the import path matches the new location.

- [ ] **Step 4: Compile**

```bash
bash bazel.cmd build //termlab/core:core
```

Resolve any remaining errors. The diff vs. the pre-Step-4 state should be ~290 lines removed from the editor (326 → ~35).

- [ ] **Step 5: Commit**

```bash
git add core/
git commit -m "$(cat <<'EOF'
refactor(core): slim TermLabTerminalEditor to composition glue, relocate to adapters/intellij/

Plan #1 Task 22. Editor goes from 326 lines to ~35 — pure composition.
Listener bodies live in controllers; widget mounting goes through the
JediTermSessionHandle.swingComponent() cross-adapter accessor.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 23: Step 4 verification + memory write-back

- [ ] **Step 1: Run all tests via IntelliJ**

Same set as Task 18 Step 1, plus any tests added during the controller extraction (probably none — controllers are tiny adapters and IntelliJ light-platform tests are deferred).

Expected: all green.

- [ ] **Step 2: Manual smoke test for Step 4**

```bash
bash bazel.cmd run //termlab:termlab_run
```

Verify the spec's Step 4 Section 5 manual checklist:
1. Branch builds.
2. App starts.
3. New Terminal opens.
4. IO works.
5. Closing tab releases resources (no zombie threads).
6. Theme/font changes propagate.
7. Resize works.
8. Drag-drop file path → formatted path pasted.
9. Focus behavior unchanged.

- [ ] **Step 3: Write Step-4 memory entry**

`/Users/dustin/.claude/projects/-Users-dustin-projects-TermLab/memory/project_core_rework_step4_complete.md`:

```markdown
---
name: Core rework Plan #1 Step 4 — TermLabTerminalEditor dismantled into controllers
description: Plan #1 Step 4 (Tasks 19–22) complete. Five controllers extracted (Appearance, Resize, DropPath, Focus, TabActions); editor shrunk from 326 to ~35 lines; relocated to adapters/intellij/.
type: project
---

Plan #1 Step 4 (Tasks 19–22) landed on branch `core-rework`. `TermLabTerminalEditor` is now composition glue.

**Files created:**
- `core/src/com/termlab/core/terminal/adapters/intellij/controller/TerminalAppearanceController.java` — MVC Controller; theme + font sync
- `core/src/com/termlab/core/terminal/adapters/intellij/controller/TerminalResizeController.java` — MVC Controller; ComponentListener → handle.resize
- `core/src/com/termlab/core/terminal/adapters/intellij/controller/TerminalDropPathController.java` — MVC Controller; TransferHandler → handle.write
- `core/src/com/termlab/core/terminal/adapters/intellij/controller/TerminalFocusController.java` — MVC Controller; FocusListener → widget focus
- `core/src/com/termlab/core/terminal/adapters/intellij/controller/TerminalTabActionsFactory.java` — Factory; tab-bar AnAction list

**Files moved:**
- `TermLabTerminalEditor.java`: `core/src/com/termlab/core/terminal/` → `adapters/intellij/`

**Files slimmed:**
- `TermLabTerminalEditor.java`: 326 → ~35 lines

**Behavior:** unchanged. All 9 manual smoke checks passed.

**ArchUnit:** all 7 boundaries clean.

**What this enables:**
- Plan #2 can wire `EditorLayoutPort` (used by future SSH-side MultiExec) without touching the editor's listener bodies.
- Future appearance/resize/drop changes require touching only one controller, not the editor.
```

Append to MEMORY.md:

```markdown
- [Core rework Plan #1 Step 4 — TermLabTerminalEditor dismantled into controllers](project_core_rework_step4_complete.md) — Plan #1 Step 4 done; five controllers extracted, editor shrunk from 326 to ~35 lines.
```

- [ ] **Step 4: Append to `docs/architecture.md` history**

```markdown
- 2026-04-28 — Plan #1 Step 4: `TermLabTerminalEditor` dismantled into five controllers (Appearance, Resize, DropPath, Focus, TabActions). See `docs/superpowers/plans/2026-04-28-core-terminal-hexagonal-foundation.md` Tasks 19–22.
```

Commit:

```bash
git add docs/architecture.md
git commit -m "$(cat <<'EOF'
docs: record Plan #1 Step 4 completion in architecture history

Plan #1 Task 23.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 24: Plan #1 final smoke test and wrap-up

- [ ] **Step 1: Final manual smoke (spec Step 5 checklist)**

```bash
bash bazel.cmd run //termlab:termlab_run
```

Walk through every item in `docs/superpowers/specs/2026-04-28-core-terminal-hexagonal-foundation-design.md` § "Manual smoke test (end of Plan #1)":

1. Branch builds.
2. App starts with only core enabled.
3. New Terminal opens a local PTY tab.
4. Terminal accepts input, echoes output, runs commands.
5. Closing the tab releases resources (no zombie threads).
6. Theme/font changes take effect.
7. Resize works.
8. Drag-and-drop file path → formatted path pasted.
9. Focus behavior unchanged.

Any failure re-opens the responsible task. Do not declare Plan #1 complete with unresolved failures.

- [ ] **Step 2: Run every test class in IntelliJ one final time**

Including `HexagonalBoundariesTest`. All green.

- [ ] **Step 3: Inventory pre-existing core/terminal files that were NOT touched**

```bash
ls core/src/com/termlab/core/terminal/
```

Files that should still be at the root (not yet relocated; deferred to Plan #2 or out-of-scope):
- `TermLabMultiExecManager.java` — Plan #2 Step 1 deletes
- `TermLabTabBarManager.java`, `TermLabTabNumberSupport.java`, `TermLabTerminalOnlyFileEditorListener.java`, `TermLabKeymapChangeListener.java`, `TermLabDistractionFreeModeListener.java` — orthogonal to terminal session lifecycle; leave at root for now (could be relocated in a later cleanup, not this plan)
- `TermLabTerminalFileType.java` — file type descriptor; leave at root
- `TermLabTerminalSettings.java` — settings model; leave at root for now
- `TermLabTerminalStarter.java` — JediTerm-specific starter wrapper; review and relocate to `adapters/jediterm/` if it imports JediTerm directly
- `TermLabTerminalWidget.java` — JediTerm widget subclass; relocate to `adapters/jediterm/` (see optional Step 4)
- `TerminalDroppedPathFormatter.java` — pure utility; leave at root or move to `domain/`
- `TerminalShellKind.java` — pure value type; could move to `domain/`
- `TermLabEditorTabTitleProvider.java` — IntelliJ adapter; relocate to `adapters/intellij/`

- [ ] **Step 4 (optional cleanup): Relocate `TermLabTerminalWidget`, `TermLabTerminalStarter`, `TermLabEditorTabTitleProvider` to their hexagonal homes**

If energy permits, move:
- `TermLabTerminalWidget.java` → `adapters/jediterm/` (note: imports `Project`, so might better fit `adapters/intellij/` — read the imports and decide)
- `TermLabTerminalStarter.java` → `adapters/jediterm/`
- `TermLabEditorTabTitleProvider.java` → `adapters/intellij/`

Update plugin.xml FQNs if affected. Compile + smoke. Commit each individually.

If energy doesn't permit, defer to a follow-up cleanup task on the same branch — these don't block Plan #2.

- [ ] **Step 5: Final wrap-up commit if architectural history was appended**

```bash
git status
git log --oneline core-rework ^main | head -30
```

Verify the commit history reads as a coherent refactor narrative. If anything looks off (missing commits, weird ordering), don't try to rewrite history — just note in the next task what's worth tidying.

- [ ] **Step 6: Branch ready for Plan #2**

Plan #1 is done. Plan #2 (extension ports → MultiExec deletion → workspace provider-driven restore → re-enable plugins) is written *after* Plan #1 lands so it can reflect what we learned. The branch stays open; do not merge to main until Plan #2 also lands.

Notify the user:

> Plan #1 complete on branch `core-rework`. All ArchUnit boundaries clean; manual smoke passed. Ready to draft Plan #2 (extension ports + MultiExec removal + workspace refactor + plugin re-enable).
