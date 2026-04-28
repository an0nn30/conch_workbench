# Core Terminal — Hexagonal Foundation (Plan #1) — Design Spec

**Date:** 2026-04-28
**Branch:** dedicated core-rework branch (named in Step 1; not `main`; reused for Plan #2)
**Source review:** `docs/termlab-clean-architecture-plan.md`
**Plan series:** Plan #1 of the TermLab core clean-architecture rework; Plan #2 follows on the same branch.

---

## Goal

Refactor `core/terminal/` from its current flat layout (where `TermLabTerminalVirtualFile` and `TermLabTerminalEditor` mix domain, application, lifecycle, IntelliJ adapter, and JediTerm adapter concerns into single classes) into a clean hexagonal architecture with strict dependency direction, so that:

1. Domain and application logic become unit-testable with plain JUnit, no IntelliJ test framework.
2. The plugin SPI surface is the literal `ports/` package — small, explicit, no IntelliJ surface bleed.
3. IntelliJ Platform churn is contained to `adapters/intellij/`.
4. Future architectural drift is mechanically detectable (ArchUnit boundary tests).

## Non-goals (explicit)

- **Not** refactoring SSH, Vault, SFTP, or any other plugin. They stay disabled across both Plan #1 and Plan #2; their refactors are subsequent plans on subsequent branches.
- **Not** changing the test infrastructure. Tests run via IntelliJ as today; CI uses its existing pipeline. No `make test`, no new Bazel runners.
- **Not** building portability. TermLab is permanently on the IntelliJ Platform; hexagonal is justified by testability, SPI clarity, drift prevention, and IntelliJ-API churn containment — never by "future port."
- **Not** building MultiExec functionality. MultiExec is a plugin (SSH-owned) feature; its current core implementation is deleted in Plan #2 Step 1. The extension points it will need are designed in Plan #2 Step 1; the SSH plugin rework reintroduces the feature later.
- **Not** introducing a logging port, cancellation token, or other speculative abstractions. They appear when consumers exist.

## Constraints

1. **Approach B — big-bang per file.** A "step" is one file fully dismantled into its new hexagonal home. The app may be temporarily broken between steps; manual testing happens at the end of the plan, not at every micro-edit.
2. **Plugins disabled** for the entire duration of Plan #1 and Plan #2. Only `core` and `sdk` (and the local PTY provider) load. Plugins are re-enabled in Plan #2 Step 3.
3. **Pattern documentation per step.** Every step explicitly states: design pattern in use, why it fits this dismantling, how it works concretely, and whether it's a precursor to a grander pattern. Memory write-back follows the step landing — see `feedback_pattern_documentation.md`.
4. **Existing test infrastructure unchanged.** New test files mirror the existing convention (`core/test/com/termlab/core/terminal/...`).

---

## Architecture — Hexagonal layout for `core/terminal/`

### Package structure (target end state of Plan #1)

```
core/src/com/termlab/core/terminal/
  domain/
    TerminalSessionId
    TerminalProviderId
    TerminalSessionDescriptor
    TerminalCapability              (enum naming individual capabilities)
    TerminalCapabilities            (immutable Set<TerminalCapability> wrapper)
    TerminalTitle
    Result                          (in-house sealed Result<T,E>)
    OpenError                       (sealed: UserCancelled | UnknownProvider | CapabilityMissing)
  application/
    TerminalTabService              (façade entry point)
    OpenTerminalUseCase
    CloseTerminalUseCase
    TerminalSessionLifecycleService
  ports/
    TerminalSessionProviderRegistry
    TerminalSessionFactory
    TerminalSessionHandle           (port: live-session control surface; write/resize/onExit/dispose)
    TerminalEventPublisher
    TerminalNotificationPort
  adapters/
    intellij/
      TermLabTerminalVirtualFile    (slimmed to metadata-only adapter)
      TermLabTerminalEditor         (composition glue after Step 4)
      TermLabTerminalEditorProvider
      TerminalSessionMonitor        (exit-watcher; observer adapter)
      IntellijTerminalNotificationAdapter
      LocalPtySessionProvider       (existing — relocated)
      TerminalAppearanceController  (Step 4)
      TerminalResizeController      (Step 4)
      TerminalDropPathController    (Step 4)
      TerminalFocusController       (Step 4)
      TerminalTabActionsFactory     (Step 4)
    jediterm/
      TerminalWidgetFactory
      JediTermSessionFactory        (implements TerminalSessionFactory port)
      OscTrackingTtyConnector       (existing — relocated)
```

### Dependency rule (mechanically enforced via ArchUnit)

| Package | May import | May NOT import |
|---|---|---|
| `domain/` | JDK, `sdk/` value types | `com.intellij.*`, `javax.swing.*`, `com.jediterm.*`, `org.apache.sshd.*` |
| `application/` | `domain/`, `ports/` | same as above |
| `ports/` | `domain/` (interfaces only — no logic, no state) | everything else |
| `adapters/intellij/` | `application/`, `domain/`, `ports/`, `com.intellij.*`, `javax.swing.*`, `adapters/jediterm/` (limited — see below) | `com.jediterm.*` directly |
| `adapters/jediterm/` | `application/`, `domain/`, `ports/`, `com.jediterm.*`, `javax.swing.*` (for `JComponent` exposure) | `com.intellij.*` |

The dependency arrow points strictly inward. `domain/` has no imports out. ArchUnit rules in `core/test/com/termlab/core/terminal/architecture/HexagonalBoundariesTest.java` verify this on every test run.

### Adapter cross-dependency (one allowed exception)

`adapters/intellij/TermLabTerminalEditor` must embed a JediTerm widget into IntelliJ's `FileEditor.getComponent()`. The widget itself is a `com.jediterm.*` type, which `adapters/intellij/` is forbidden to import. Resolution: `adapters/jediterm/JediTermSessionHandle` exposes a `swingComponent(): JComponent` accessor; `adapters/intellij/` imports `JediTermSessionHandle` (our class) and calls that accessor. The IntelliJ adapter sees a `JComponent` (Swing, allowed) without ever importing `com.jediterm.*`. This is the *single* allowed touchpoint between adapter packages; ArchUnit explicitly permits `adapters/intellij/` to reference `adapters/jediterm/JediTermSessionHandle` and forbids any other `adapters/jediterm/*` import from `adapters/intellij/*`.

---

## Components — what lands where, what pattern, why

### `domain/` — pure value types, zero infrastructure dependencies

| Class | Pattern | Role |
|---|---|---|
| `TerminalSessionId` | Value Object | Stable identity for a session; opaque wrapper around UUID so call sites can't pass a raw String |
| `TerminalProviderId` | Value Object | Stable identity for a provider (e.g., `"com.termlab.local-pty"`); typed for type-safe registry lookup |
| `TerminalSessionDescriptor` | Value Object (Java `record`) | Serializable intent: `(providerId, displayTitle, workingDirectory, providerState)`. Replaces the field-bag inside `TermLabTerminalVirtualFile` today |
| `TerminalCapability` | Enum | Names individual capabilities: `SUPPORTS_MULTI_EXEC`, `CLOSE_TAB_ON_SESSION_END`, `SUPPORTS_BROADCAST`, … |
| `TerminalCapabilities` | Value Object | Immutable `Set<TerminalCapability>` wrapper with `has(...)`, `union(...)`, `intersect(...)` semantics. Replaces the hardcoded `SSH_PROVIDER_ID` check |
| `TerminalTitle` | Value Object | Display title (raw + decorated); explicit type so title-update logic stops manipulating mutable strings |
| `Result<T, E>` | Result Object (sealed type) | In-house ~30-line sealed class: `Success(T)` \| `Failure(E)`. Used by use cases that have expected failures |
| `OpenError` | Sealed Type | `UserCancelled` \| `UnknownProvider(TerminalProviderId)` \| `CapabilityMissing(TerminalCapability)` |

Pattern note: idiomatic Java records for the data carriers; tiny class with private constructor + static factory for the IDs. No hand-rolled `equals`/`hashCode`.

### `ports/` — interfaces only

| Interface | Pattern | Role |
|---|---|---|
| `TerminalSessionProviderRegistry` | Registry / Service Locator scoped to a single SPI | Lookup by `TerminalProviderId` → `TerminalSessionProvider`. Insulates `application/` from IntelliJ's `ExtensionPointName` API |
| `TerminalSessionFactory` | Factory port | Given a descriptor + provider, produce a `TerminalSessionHandle`. Implemented by `adapters/jediterm/JediTermSessionFactory` |
| `TerminalSessionHandle` | Port (live-session control surface) | Domain-flavored operations on a running session: `write(byte[])`, `resize(rows, cols)`, `onExit(Consumer<ExitReason>)`, `dispose()`. The `application/` layer holds this reference; never sees `TtyConnector` or `JediTermWidget` |
| `TerminalEventPublisher` | Observer / Publisher port | Publish lifecycle events (`TerminalOpened`, `TerminalClosed`, `TerminalCwdChanged`). Plan #1 emits only; subscriber port arrives in Plan #2 |
| `TerminalNotificationPort` | Adapter port | Surface user-facing notifications without `application/` knowing about IntelliJ's `Notifications.Bus` |

Pattern note: interfaces with no default methods carrying logic, no static state, no fields. Every method takes domain types in, returns domain types or `Result` out.

### `application/` — orchestration, no infrastructure

| Class | Pattern | Role |
|---|---|---|
| `TerminalTabService` | Application Service / Façade | Public entry point: `open(descriptor) → Result<OpenResult, OpenError>`, `close(sessionId)`. Coordinates registry → factory → lifecycle service. Replaces direct `new TermLabTerminalVirtualFile(...)` calls from actions |
| `OpenTerminalUseCase` | Use Case | Single-method class: `execute(descriptor) → Result<...>`. The verb "open a terminal." Splitting it from the service lets future actions invoke the use case directly with their own coordination |
| `CloseTerminalUseCase` | Use Case | Symmetric; idempotent — closing an unknown session id is a silent no-op (forgiving close semantics) |
| `TerminalSessionLifecycleService` | Application Service / Session Registry | Owns the active-session table `Map<TerminalSessionId, TerminalSessionHandle>`. Replaces the lifecycle responsibilities currently inside `TermLabTerminalVirtualFile`. Fail-fast on duplicate registration (programmer error) |

### `adapters/jediterm/` — JediTerm-specific glue

| Class | Pattern | Role |
|---|---|---|
| `TerminalWidgetFactory` | Factory adapter | Creates `JediTermWidget` from a `TtyConnector`. The *only* place `JediTermWidget` is constructed |
| `JediTermSessionFactory` | Factory adapter | Implements `TerminalSessionFactory` port. Builds a `JediTermSessionHandle` wrapping widget + connector + OSC tracking |
| `JediTermSessionHandle` | Adapter (port impl) | Implements `TerminalSessionHandle` port. Wraps `JediTermWidget` + `TtyConnector`. **Additionally** exposes a package-private (or `adapters`-scoped) accessor `swingComponent()` returning `JComponent`, used by `adapters/intellij/TermLabTerminalEditor` to embed the widget. The accessor is the single allowed cross-adapter touchpoint between `adapters/jediterm/` and `adapters/intellij/` (see "Adapter cross-dependency" below) |
| `OscTrackingTtyConnector` | Decorator (existing — relocated) | Already exists; moved without code change |

### `adapters/intellij/` — IntelliJ-specific glue (anti-corruption layer)

| Class | Pattern | Role |
|---|---|---|
| `TermLabTerminalVirtualFile` | Adapter (anti-corruption layer) | Shrinks from 299 lines to ~50. Carries `TerminalSessionId` + `TerminalSessionDescriptor`. Implements `VirtualFile` for `FileEditorManager` routing. Owns *no* lifecycle, widget, or thread |
| `TermLabTerminalEditor` | Adapter | Bridges `FileEditor` to `TerminalSessionHandle` (cast to `JediTermSessionHandle` for the `swingComponent()` accessor). After Step 4: ~30 lines of composition glue, no listener bodies |
| `TermLabTerminalEditorProvider` | Factory (existing — relocated) | Already exists; moves to `adapters/intellij/` |
| `TerminalSessionMonitor` | Observer adapter | Watches `TtyConnector` for exit; republishes via `TerminalEventPublisher`. Replaces the exit-watcher thread inside `TermLabTerminalVirtualFile` |
| `IntellijTerminalNotificationAdapter` | Adapter | Implements `TerminalNotificationPort` using IntelliJ's `Notifications` API |
| `LocalPtySessionProvider` | Adapter (existing — relocated) | Already exists in `core/terminal/`; relocated; implementation unchanged |
| `TerminalAppearanceController` | MVC Controller (Step 4) | Theme/font sync into the JediTerm widget |
| `TerminalResizeController` | MVC Controller (Step 4) | Editor resize listener → connector resize call |
| `TerminalDropPathController` | MVC Controller (Step 4) | Drop-target installation; file-path-to-input formatting |
| `TerminalFocusController` | MVC Controller (Step 4) | Focus listener → request focus into widget |
| `TerminalTabActionsFactory` | Factory (Step 4) | Builds the tab-bar `AnAction` list (currently inlined in editor) |

### Files deleted in Plan #1

None. (`TermLabMultiExecManager` deletion is Plan #2 Step 1.)

### SDK change

`sdk/src/com/termlab/sdk/TerminalSessionProvider.java` gains a default-method `capabilities()` returning empty/no-caps. Non-breaking — existing implementations compile unchanged. Existing `closeTabOnSessionEnd()` becomes a flag inside `TerminalCapabilities` (the default method on `TerminalSessionProvider` continues to delegate, preserving compile compatibility).

---

## Data flow

### Flow 1 — User opens a new local PTY tab

```
User clicks "New Terminal" menu item
        │
        ▼
[adapters/intellij] NewTerminalTabAction.actionPerformed
                    builds TerminalSessionDescriptor.localPty(cwd, title)
        ▼
[application]       TerminalTabService.open(descriptor)
        ▼
[application]       OpenTerminalUseCase.execute(descriptor)
        │  1. registry lookup
        ▼
[ports]             TerminalSessionProviderRegistry.find(descriptor.providerId)
        │  returns LocalPtySessionProvider
        │  2. provider creates the TtyConnector
        │  3. factory builds TerminalSessionHandle
        ▼
[ports]             TerminalSessionFactory.create(descriptor, connector)
        ▼
[adapters/jediterm] JediTermSessionFactory
                    builds JediTermSessionHandle = (widget + connector + OSC)
        ▼
[application]       TerminalSessionLifecycleService.register(sessionId, handle)
        ▼
[ports]             TerminalEventPublisher.publish(TerminalOpened(sessionId, descriptor))
        │  back up the stack:
        ▼
[adapters/intellij] FileEditorManager.openFile(new TermLabTerminalVirtualFile(sessionId, descriptor))
        ▼
[adapters/intellij] TermLabTerminalEditorProvider.createEditor → TermLabTerminalEditor
                    (composes session handle + controllers; no business logic)
```

`OpenTerminalUseCase` never imports `com.intellij.*` or `com.jediterm.*`. The IntelliJ side bookends entry and exit; the middle is pure.

### Flow 2 — Session process exits, tab closes

```
Local PTY process exits
        ▼
[adapters/intellij] TerminalSessionMonitor (observing TtyConnector)
        ▼
[application]       CloseTerminalUseCase.execute(sessionId, reason=PROCESS_EXIT)
        ▼
[application]       TerminalSessionLifecycleService.release(sessionId)
        ▼
[ports]             TerminalEventPublisher.publish(TerminalClosed(sessionId, reason))
        │
        ▼
[adapters/intellij] (subscriber, future Plan #2) → FileEditorManager.closeFile(virtualFile)
```

`closeTabOnSessionEnd` policy lives in `application/` (consults `TerminalCapabilities`); the actual `closeFile` call is the adapter's reaction to the event. Currently both are conflated inside the exit-watcher thread.

### Flow 3 — Provider registration at startup

```
IntelliJ loads the core plugin
        ▼
[adapters/intellij] TermLabStartupActivity (existing)
                    for each provider in IntelliJ extension point:
        ▼
[ports]             TerminalSessionProviderRegistry.register(provider)
                    Plan #1: only LocalPtySessionProvider registers (plugins disabled)
```

Adding a future provider is one registration call. The registry doesn't change.

---

## Error handling

### Two categories of failure

**Expected (modeled as Result):**

| Failure | Type |
|---|---|
| User cancels (Plan #1 has no prompts; type exists for Plan #2 + SSH) | `OpenError.UserCancelled` |
| Provider id not in registry | `OpenError.UnknownProvider(TerminalProviderId)` |
| Provider doesn't support a requested capability | `OpenError.CapabilityMissing(TerminalCapability)` |

Use case signature:

```java
public Result<OpenResult, OpenError> execute(TerminalSessionDescriptor descriptor);
```

**Unexpected (exceptions, propagate to adapter layer):**

| Failure | Handling |
|---|---|
| `JediTermWidget` constructor throws | Propagates; adapter catches `RuntimeException` → notification |
| `LocalPtySessionProvider.createConnector` IOException | Wrapped as `RuntimeException` for now; promote to typed failure later if it becomes a real user-facing scenario |
| Exit-watcher thread blows up | `Logger.error` + emit `TerminalClosed(reason=MONITOR_FAILED)`; never silent death |

### Single notification port at the boundary

`TerminalNotificationPort` is the only path errors reach the user. Domain and application code never call `Notifications.Bus` directly. Adapters format user-facing strings; domain doesn't know what an "error notification" is.

### Logging

`com.intellij.openapi.diagnostic.Logger` — adapter layer only. Domain and application don't log; they return `Result`. No `LogPort` in Plan #1 (YAGNI).

### Cancellation

Local PTY opens are sub-millisecond; no cancellation token in Plan #1. The pattern arrives when a long-running operation needs it (Plan #2 / SSH plan).

---

## Testing

### Layout

```
core/test/com/termlab/core/terminal/
  domain/                           pure value-object tests
  application/                      use case + service tests with fakes
  architecture/
    HexagonalBoundariesTest         ArchUnit dependency rule tests
  testfixtures/                     reusable Fake* implementations
    FakeTerminalSessionProviderRegistry
    FakeTerminalSessionFactory
    FakeTerminalEventPublisher
```

### Plan #1 active coverage

**Pure JUnit (domain):**
- `TerminalSessionDescriptor` — equality, null-rejection, copy-with semantics
- `TerminalCapabilities` — flag combinators
- `TerminalSessionId` / `TerminalProviderId` — equality, opacity (no accidental construction from arbitrary string)
- `Result<T, E>` — success/failure factories, `map`, `flatMap`, `isSuccess`

**JUnit + Fakes (application):**
- `OpenTerminalUseCase`:
  - Happy path → returns `OpenResult`, lifecycle has handle, `TerminalOpened` published
  - Unknown provider → returns `OpenError.UnknownProvider`, no side effects
  - Factory throws → `RuntimeException` propagates (no swallow)
- `CloseTerminalUseCase`:
  - Existing session → handle released, `TerminalClosed` published
  - Unknown session id → idempotent no-op
- `TerminalSessionLifecycleService`:
  - register-then-get round-trip
  - register-twice with same id → `IllegalStateException`
  - release removes the handle
- `TerminalSessionProviderRegistry` (default in-memory impl):
  - register-then-find round-trip
  - find unknown id → empty
  - register-twice → fail-fast

### ArchUnit boundary tests (Step 2)

Adopted: ArchUnit is a JUnit-runnable library, no infrastructure change. Rules in `HexagonalBoundariesTest.java`:

```
classes in domain            must not import com.intellij..*, javax.swing..*, com.jediterm..*
classes in application       must not import com.intellij..*, javax.swing..*, com.jediterm..*
classes in ports             must be interfaces (no fields, no instance state)
classes in adapters.intellij must not import com.jediterm..*
classes in adapters.intellij may import adapters.jediterm.JediTermSessionHandle ONLY (single allowed
                              cross-adapter touchpoint for widget embedding)
classes in adapters.jediterm must not import com.intellij..*
```

Drift becomes a build failure, not a code-review hope.

### Deferred from Plan #1

- IntelliJ light-platform tests for `TermLabTerminalVirtualFile`, `TermLabTerminalEditor`, `TermLabTerminalEditorProvider` — added when bugs prove they're needed
- `JediTermSessionFactory` test — defer; mostly mechanical assembly
- `TerminalSessionMonitor` test — threading-heavy; defer until behavior gets harder

### Existing tests during Plan #1

`plugins/ssh/test/...` and other plugin tests keep passing — Plan #1 doesn't touch plugin source. The only SDK change is the non-breaking default-method `capabilities()` addition.

### Manual smoke test (end of Plan #1)

After Step 4 lands:
1. Branch builds.
2. App starts with only core enabled.
3. New Terminal action opens a local PTY tab.
4. Terminal accepts input, echoes output, runs commands.
5. Closing the tab releases resources (no zombie threads).
6. Theme/font changes take effect.
7. Resize works.
8. Drag-and-drop file path → formatted path pasted.
9. Focus behavior unchanged.

Any failure re-opens the responsible step.

---

## Step structure (Plan #1)

Each step follows the pattern documentation requirement: explicit pattern + why + how + precursor-or-not, plus a memory write-back after the step lands.

### Step 1 — Setup

Create dedicated branch off `main`. Disable all non-core plugins (mechanism: modify root `BUILD.bazel` to comment out plugin deps with a `// CORE-REWORK` marker for easy revert; verify against `core/resources/META-INF/plugin.xml` and any registered extension points). Scaffold the empty hexagonal package skeleton. Confirm the app boots with local PTY only.

**Patterns:** none yet — environmental setup.
**Memory write:** branch name; plugin disablement mechanism used; verification result.

### Step 2 — Architecture guardrails

Write `docs/architecture.md` defining:
- Bounded contexts (terminal, workspace, future plugin contexts)
- The dependency rule table from this spec
- Where IntelliJ/Swing/JediTerm/MINA/PasswordSafe imports are allowed
- The pattern catalog (Value Object, Use Case, Adapter, Anti-Corruption Layer, Result, Registry, Factory)

Add ArchUnit dependency. Write `HexagonalBoundariesTest.java` rules. (Tests pass trivially in Step 2 because nothing has been moved yet; they begin to enforce in Step 3.)

**Patterns:** documentation-as-contract; ArchUnit as Architecture Conformance pattern.
**Memory write:** doc location; ArchUnit rules in place; test class location.

### Step 3 — Dismantle `TermLabTerminalVirtualFile`

Single big-bang step: introduce all needed `domain/`, `application/`, `ports/` types; build `JediTermSessionFactory` and `TerminalSessionMonitor` adapters; relocate `OscTrackingTtyConnector` and `LocalPtySessionProvider`; slim `TermLabTerminalVirtualFile` to metadata-only adapter; migrate `NewTerminalTabAction` (and any other in-core caller) to call `TerminalTabService`. SDK gains default-method `capabilities()`.

**Patterns:** Value Object (descriptors, IDs, capabilities), Application Service / Use Case (TabService, Open/Close), Registry (provider lookup), Factory (session factory), Port (`TerminalSessionHandle`), Observer (session monitor), Adapter / Anti-Corruption Layer (slim virtual file), Result (use case return).
**Why:** the virtual file was conflating domain identity, IntelliJ adapter duty, JediTerm widget construction, and lifecycle ownership. The split assigns each role to a single class with a single import surface.
**Precursor to:** Plan #2's extension-point ports (TerminalInputWriter, EditorLayoutPort) hang off this same architecture; the SSH plugin's eventual rework copies this layout.
**Memory write:** every class created, with package path + pattern + role; deletions; relocations; behavior changes; manual verification of "open new local PTY tab still works."

### Step 4 — Dismantle `TermLabTerminalEditor`

Extract `TerminalAppearanceController`, `TerminalResizeController`, `TerminalDropPathController`, `TerminalFocusController`, `TerminalTabActionsFactory` into `adapters/intellij/`. Editor becomes ~30 lines of composition glue.

**Patterns:** MVC Controller / Single-Responsibility Principle, Composition over inheritance, Factory (tab actions).
**Why:** today the editor constructor mixes view construction, message-bus subscription, listener wiring, drop-target installation, and tab-action building — five concerns at three abstraction levels. Each concern has different change drivers; splitting them follows SRP.
**Precursor to:** future appearance/resize/drop behavior changes touch one controller, not a 326-line editor.
**Memory write:** controllers created, with role; editor composition shape; manual verification of "appearance/resize/drop/focus all still work."

### Step 5 — Manual smoke test

The 9-item checklist above. Any failure → re-open the responsible step. Plan #1 lands when the checklist passes.

**Patterns:** none — verification phase.
**Memory write:** smoke test results; any deferred follow-ups.

---

## Out of scope — explicit deferrals

| Concern | Where it lands |
|---|---|
| Workspace restore provider-driven | Plan #2 Step 2 |
| Schema-versioned workspace persistence | Plan #2 Step 2 |
| `TerminalInputWriter` extension port | Plan #2 Step 1 |
| `EditorLayoutPort` + `IntellijEditorLayoutGateway` | Plan #2 Step 1 |
| Lifecycle event subscriber port | Plan #2 Step 1 |
| `TermLabMultiExecManager` deletion | Plan #2 Step 1 |
| Re-enable plugins + smoke test | Plan #2 Step 3 |
| SSH plugin refactor (incl. MultiExec reintroduction) | Subsequent SSH plan, separate branch |
| Other plugin refactors | Subsequent plans, separate branches |
| Test infrastructure changes | Permanently out of scope |
| Logging port | When a domain-side consumer needs it |
| Cancellation token | When a long-running operation needs it |

## Cross-references

- Architecture review source: `docs/termlab-clean-architecture-plan.md`
- Plan series memory: `project_core_rework_plan_split.md`
- Pattern documentation requirement: `feedback_pattern_documentation.md`
- MultiExec extraction record: `project_multiexec_extraction.md`
- IntelliJ binding record: `project_termlab_intellij_bound.md`
- Hexagonal preference: `feedback_hexagonal_architecture.md`
