# TermLab Codebase Cleanup and Clean Architecture Plan

Repository reviewed: `termlab-io/TermLab`

This document exports the complete architecture review and cleanup plan previously provided, preserving the structure, recommendations, code blocks, and source references.

## Review scope

I reviewed representative files across `core`, `sdk`, `plugins/ssh`, `plugins/vault`, `plugins/sftp`, the Bazel build files, and the architecture docs. I did not inspect every file line-by-line, but I looked at enough of the main flows to identify the structural issues.

## Overall verdict

TermLab has the right *top-level* idea: `core`, `sdk`, and first-party plugins are already separated, and the product direction is explicitly plugin-first. The README describes `core`, `sdk`, and `plugins` as separate high-level areas, with SSH, Vault, SFTP, tunnels, editor, runner, and other capabilities packaged as product plugins.

The design spec also says the core should be thin: extension points, JediTerm wiring, workspace state, and plugin contracts, while SSH/SFTP/Vault/Tunnels live outside core.

The implementation is not yet clean architecture. It is more accurately a working IntelliJ-platform product with growing feature code. That is normal at this stage, but it needs cleanup before adding more major plugins.

The biggest problems are:

1. `core` owns too much product/platform behavior.
2. Runtime lifecycle, UI, persistence, and domain concerns are mixed.
3. Actions and tool windows directly construct implementation classes.
4. Services often act as mutable stores plus repositories plus event sources.
5. IntelliJ internals leak into core behavior in places that will be fragile.
6. Tests exist, but the build structure suggests many tests are compile-only rather than executed.

I would not rewrite the app wholesale. I would refactor behind the current behavior, starting with the terminal/session boundary.

---

## What is already good

The module split is a strong start. `core`, `sdk`, and each plugin have separate Bazel targets. The root `BUILD.bazel` wires first-party plugins independently into the runtime, including SSH, Vault, Tunnels, Share, SFTP, Search, Sysinfo, Proxmox, Editor, and Runner.

The SDK is intentionally small. `TerminalSessionProvider` is a clear plugin contract for terminal backends. That is the right direction.

Some domain models are already sane. `SshHost` is an immutable record with stable identity, auth strategy, proxy settings, and copy-style mutation methods. That is a good DDD starting point.

There are useful unit tests around `HostStore`, including persistence, reload, snapshots, update/remove, and listener behavior.

The Vault code shows serious thought around state, encryption, device binding, and secret lifetime. `LockManager` is explicit about lock/unlock/seal states and cached credentials.

---

## Main architectural problems

### 1. Core is not thin enough

`core/resources/META-INF/plugin.xml` is doing a lot: extension points, settings pages, editor providers, startup activities, platform suppressors, search customization, action stripping, theme sync, tool window stripping, file watcher disabling, Defender notifier suppression, terminal actions, and more.

That may be necessary for a custom IntelliJ product, but it should be split conceptually:

```text
core/domain          pure TermLab concepts
core/application     use cases
core/platform        IntelliJ-specific adapters
core/customization   product stripping / IntelliJ surgery
```

Right now “core” means both “TermLab core domain” and “hack IntelliJ into not being an IDE.” Those are different responsibilities.

### 2. `TermLabTerminalVirtualFile` owns too much

`TermLabTerminalVirtualFile` is currently not just a virtual file. It owns session identity, provider, CWD, title state, session context, shared session lifecycle, widget creation, connector creation, OSC tracking, exit watcher thread, tab close behavior, disconnected rendering, and title updates.

That is the clearest cleanup target.

A `VirtualFile` should be a lightweight adapter object. It should not own terminal session lifecycle. Move that logic into:

```text
TerminalSessionDescriptor     serializable intent: provider id, title, cwd, host id, etc.
TerminalSessionHandle         runtime connector/widget/session state
TerminalSessionService        create/acquire/release/close session lifecycle
TerminalPresentationService   title, CWD, disconnected display
TerminalEditorAdapter         IntelliJ FileEditor bridge only
```

### 3. `TermLabTerminalEditor` mixes view, lifecycle, event subscription, resizing, file drop, appearance, and multi-exec integration

The editor constructor acquires a shared session, gets widget/connector, connects to the message bus, sets tab actions, builds Swing UI, installs appearance listeners, file-drop handler, resize listeners, and focus listeners.

This should be split into controllers:

```text
TerminalEditorView
TerminalAppearanceController
TerminalResizeController
TerminalDropPathController
TerminalFocusController
TerminalTabActionsFactory
```

The editor should become mostly composition glue.

### 4. SSH connection flow mixes domain, UI, threading, retry, and infrastructure

`SshSessionProvider` validates context type, shows Swing dialogs, resolves credentials, resolves bastion credentials, opens modal progress, manages cancellation, shuts down the SSH client on cancel, handles retry, classifies errors, and returns the `TtyConnector`.

That class should become a thin adapter:

```java
public final class SshSessionProvider implements TerminalSessionProvider {
    private final ConnectSshSessionUseCase connectSshSession;

    @Override
    public TtyConnector createSession(SessionRequest request) {
        return connectSshSession.connect(request).connectorOrNull();
    }
}
```

Move the current behavior into application services and ports:

```text
ssh/domain
  SshHost
  SshEndpoint
  ProxyJump
  SshAuthMethod
  KnownHostEntry

ssh/application
  ConnectSshSessionUseCase
  ResolveSshCredentialUseCase
  VerifyHostKeyUseCase
  RetrySshConnectionPolicy

ssh/ports
  SshClientPort
  CredentialResolverPort
  HostKeyRepository
  UserPromptPort
  ProgressPort

ssh/adapters
  mina/ApacheMinaSshClientAdapter
  intellij/IntellijUserPromptAdapter
  intellij/IntellijProgressAdapter
  json/JsonHostRepository
```

### 5. Credential resolution is also mixed

`HostCredentialBundle` does too much. It dispatches auth variants, creates resolvers/pickers, accesses `HostStore` through `ApplicationManager`, shows UI dialogs, inspects key files, resolves bastion credentials, and owns secret cleanup.

The idea is good. The placement is wrong.

It should become a use case with injected dependencies:

```java
public final class ResolveSshCredentialsUseCase {
    private final CredentialProviderPort credentials;
    private final KeyFileInspectorPort keyFiles;
    private final HostRepository hosts;
    private final UserPromptPort prompts;

    public CredentialBundle resolve(SshHost host) { ... }
}
```

No `ApplicationManager`, no Swing dialogs, no static factory wiring in the application logic.

### 6. `HostStore` is a store, repository, event source, and service all in one

`HostStore` loads from disk in its no-arg service constructor, keeps mutable in-memory state, delegates JSON persistence to `HostsFile`, exposes CRUD, exposes listeners, and requires callers to remember to call `save()`.

This is workable for early development but not ideal for contributors. Split it:

```text
HostRepository
  List<SshHost> findAll()
  Optional<SshHost> findById(HostId id)
  void save(SshHost host)
  void delete(HostId id)

JsonHostRepository
  disk persistence only

HostCatalogService
  validates commands
  owns mutation workflow
  publishes HostAdded/HostUpdated/HostDeleted events
```

The UI should call `HostCatalogService.addHost(...)`, not mutate a list and call `saveAndRefresh()`.

### 7. Tool windows contain application logic

`HostsToolWindow` directly manages `HostStore`, dialogs, menu actions, save/reload, duplicate, delete, and connect behavior.

That should become:

```text
HostsToolWindow               Swing rendering only
HostsViewModel                observable list + selected host
HostCommands                  add/edit/delete/duplicate/connect use cases
HostCatalogService            domain/application service
```

This is especially important for contributors. Tool window code gets messy fast if every feature is implemented directly inside Swing panels.

### 8. `ConnectToHostAction` bypasses the architecture

`ConnectToHostAction` constructs `TermLabTerminalVirtualFile`, directly instantiates `SshSessionProvider`, stashes `SshSessionContext`, and opens the file.

That should go through a core terminal service:

```java
public final class ConnectToHostAction {
    public static void run(Project project, SshHost host) {
        TerminalTabService.getInstance(project).open(
            TerminalSessionDescriptor.ssh(host.id(), host.label())
        );
    }
}
```

Core should resolve the provider by ID. SSH should not manually build core virtual files.

### 9. MultiExec is powerful but fragile

`TermLabMultiExecManager` reaches into IntelliJ editor internals, uses `EditorWindow`, `EditorComposite`, `EditorsSplitters`, `FileEditorOpenOptions`, and reflection methods discovered by prefix. It also hardcodes `SSH_PROVIDER_ID = "com.termlab.ssh"`.

Do not delete this. It is valuable product behavior. But isolate it aggressively.

Create:

```text
MultiExecService              pure state machine: active sessions, excluded sessions, source session
TerminalBroadcastService      writes input to selected sessions
EditorLayoutGateway           IntelliJ-specific layout capture/restore
IntellijEditorLayoutGateway   all reflection and EditorWindow internals live here
```

Also replace the SSH hardcode with a provider capability:

```java
interface TerminalSessionProvider {
    default boolean supportsMultiExec() {
        return false;
    }
}
```

or session metadata:

```java
TerminalCapabilities capabilities();
```

### 10. Workspace restore is not extension-driven

`WorkspaceManager` captures terminal tabs, writes JSON, and restores only local PTY sessions by checking `"com.termlab.local-pty"`. It constructs `LocalPtySessionProvider` directly.

`WorkspaceState` has a `connectionId` field, but the restore path is not provider-based yet. Serialization is also plain Gson with no schema version or migration strategy.

You need a provider persistence SPI:

```java
public interface TerminalSessionProvider {
    ProviderId id();

    TerminalSessionDescriptor describe(TerminalSessionHandle session);

    TerminalSessionRestoreResult restore(TerminalSessionDescriptor descriptor);
}
```

Then workspace restore can be provider-neutral:

```java
for (TerminalSessionDescriptor descriptor : workspace.sessions()) {
    terminalTabService.restore(descriptor);
}
```

SSH can restore a disconnected host tab. SFTP can restore a file browser context. Local PTY can restore CWD.

### 11. Tests are not yet where they need to be

The plugin build files define test libraries, but I did not see a clear runnable test target pattern. The Vault build file explicitly says the test sources compile as a separate library and that an executable test target can be added later. SSH and SFTP also define `*_test_lib` targets. The Makefile has build and perf targets, but no obvious `make test`.

Compiling tests is not the same as running tests. This should be fixed early.

---

## Recommended clean architecture

Use a modular-monolith, hexagonal architecture. Do not overdo enterprise DDD ceremony, but enforce boundaries.

### Dependency rule

```text
domain        -> JDK only, maybe sdk value types
application   -> domain + ports
ports         -> interfaces only
adapters      -> application + domain + external APIs
intellij UI   -> adapters/application only
```

### Core package shape

```text
core/src/com/termlab/core/
  terminal/
    domain/
      TerminalSessionId
      TerminalSessionDescriptor
      TerminalTitle
      TerminalCapabilities
      TerminalLifecycleEvent
    application/
      TerminalTabService
      OpenTerminalUseCase
      CloseTerminalUseCase
      RestoreTerminalUseCase
      TerminalSessionLifecycleService
    ports/
      TerminalSessionProviderRegistry
      TerminalSessionFactory
      TerminalEventPublisher
      TerminalNotificationPort
      EditorLayoutPort
    adapters/
      intellij/
        TermLabTerminalVirtualFile
        TermLabTerminalEditor
        TermLabTerminalEditorProvider
        IntellijTerminalNotificationAdapter
        IntellijEditorLayoutGateway
      jediterm/
        TerminalWidgetFactory

  workspace/
    domain/
      Workspace
      WorkspaceId
      WorkspaceSession
      WorkspaceSchemaVersion
    application/
      SaveWorkspaceUseCase
      RestoreWorkspaceUseCase
      ListWorkspacesUseCase
    ports/
      WorkspaceRepository
      WorkspaceMigration
    adapters/
      json/
        JsonWorkspaceRepository
```

### SSH package shape

```text
plugins/ssh/src/com/termlab/ssh/
  domain/
    SshHost
    HostId
    SshEndpoint
    SshPort
    Username
    ProxyJump
    SshAuthMethod
    KnownHostEntry
  application/
    HostCatalogService
    ConnectSshSessionUseCase
    ResolveSshCredentialUseCase
    VerifyHostKeyUseCase
  ports/
    HostRepository
    KnownHostsRepository
    SshClientPort
    CredentialPromptPort
    ProgressPort
  adapters/
    intellij/
      NewSshSessionAction
      HostsToolWindow
      HostEditDialog
      IntellijProgressAdapter
      IntellijPromptAdapter
    mina/
      MinaSshClientAdapter
    json/
      JsonHostRepository
      KnownHostsFileRepository
    termlab/
      SshTerminalSessionProvider
```

### Vault package shape

```text
plugins/vault/src/com/termlab/vault/
  domain/
    Vault
    CredentialEntry
    VaultState
    SecretBytes
  application/
    UnlockVaultUseCase
    LockVaultUseCase
    AddCredentialUseCase
    ResolveCredentialUseCase
  ports/
    VaultRepository
    DeviceSecretStore
    CryptoService
    Clock
  adapters/
    intellij/
      VaultDialog
      VaultStatusBarWidget
      PasswordSafeDeviceSecretStore
    crypto/
      AesGcmVaultCryptoService
    file/
      EncryptedVaultFileRepository
```

### SFTP package shape

```text
plugins/sftp/src/com/termlab/sftp/
  domain/
    RemotePath
    FileTransfer
    SftpSessionId
  application/
    OpenSftpSessionUseCase
    BrowseRemoteDirectoryUseCase
    UploadFileUseCase
    DownloadFileUseCase
  ports/
    SftpClientPort
    SftpSessionRegistry
    FileTransferProgressPort
  adapters/
    intellij/
      SftpToolWindow
      SftpVirtualFileSystem
    sshd/
      MinaSftpClientAdapter
```

---

## Event-driven architecture recommendation

Use EDA inside the app, but keep it modest. Do not introduce Kafka-style complexity into a desktop app.

Use typed in-process events for decoupling:

```java
public sealed interface TermLabEvent permits
    TerminalOpened,
    TerminalClosed,
    TerminalCwdChanged,
    TerminalTitleChanged,
    HostAdded,
    HostUpdated,
    HostDeleted,
    VaultUnlocked,
    VaultLocked,
    SftpSessionOpened,
    SftpSessionClosed {}
```

Use cases publish events. UI components subscribe.

Good uses:

```text
HostAdded              refresh Hosts palette/tool window
VaultLocked            disable credential-dependent actions
TerminalCwdChanged     update workspace state and file explorer
TerminalClosed         update MultiExec state
SftpSessionClosed      refresh SFTP UI and VFS state
```

Avoid using events for direct request/response workflows. For example, “connect to SSH host” should remain a use case call, not a fire-and-pray event.

---

## Design patterns that fit TermLab

Use these deliberately:

| Pattern | Where it fits |
|---|---|
| Repository | Hosts, workspaces, known_hosts, vault file |
| Application Service / Use Case | Open terminal, connect SSH, add host, unlock vault, upload file |
| Factory | terminal widgets, session handles, provider descriptors |
| Strategy | SSH auth modes, credential resolution, host-key decisions |
| Adapter | IntelliJ APIs, Apache MINA, PasswordSafe, JSON files |
| Observer / Domain Events | UI refresh, palette refresh, status bar state |
| Anti-corruption Layer | IntelliJ editor internals and reflection-heavy layout handling |
| Result object | expected failures instead of throwing/showing dialogs from domain code |
| State machine | vault lock state, SFTP session state, terminal lifecycle |

Avoid these for now:

```text
CQRS everywhere
event sourcing
abstract factories for every small object
generic "manager" classes
global service locators in domain/application code
```

---

## Testing plan

### 1. Add a real test command

Add Bazel test targets and a Makefile target:

```make
test:
    $(BAZEL) test //termlab/core:core_tests \
                  //termlab/plugins/ssh:ssh_tests \
                  //termlab/plugins/vault:vault_tests \
                  //termlab/plugins/sftp:sftp_tests
```

The exact Bazel rule depends on the IntelliJ repo conventions, but the key is: tests must execute, not just compile.

### 2. Add architecture boundary tests

Even simple package-boundary tests would help:

```text
domain must not import com.intellij.*
domain must not import javax.swing.*
application must not import com.intellij.openapi.ui.Messages
adapters may import IntelliJ/Swing/MINA/PasswordSafe
```

ArchUnit would be ideal if it fits the build. If not, a simple source scanner test is good enough.

### 3. Unit tests to add first

```text
TerminalSessionDescriptor serialization
Workspace schema versioning and migration
Workspace restore with unknown provider
Host validation: blank host, bad port, blank username
HostCatalogService add/update/delete persistence
Ssh credential resolution strategy per auth mode
Vault lock/unlock state transition ordering
SFTP session refcount release behavior
MultiExec pure state machine: activate, exclude, close, broadcast target selection
```

### 4. Integration tests

```text
JsonHostRepository with @TempDir
JsonWorkspaceRepository with @TempDir
Vault file roundtrip / wrong password / tamper detection
Embedded Apache MINA SSH server connection test
Embedded SFTP server browse/upload/download test
IntelliJ light-platform tests for actions/editors/tool windows only
```

### 5. Security tests

For Vault and SSH:

```text
wrong password fails
tampered vault file fails
host key mismatch fails hard
credential arrays are closed/zeroed where possible
no accidental secret serialization in host/workspace JSON
```

---

## Refactor plan

### Phase 0: Put guardrails in place

Create `docs/architecture.md` that defines:

```text
bounded contexts
package layout
dependency rules
testing rules
where IntelliJ code is allowed
where Swing code is allowed
where Apache MINA code is allowed
```

Add runnable tests before doing heavy refactors.

### Phase 1: Extract terminal session descriptors

Create:

```java
TerminalSessionDescriptor
TerminalProviderId
TerminalSessionId
TerminalTabService
TerminalSessionProviderRegistry
```

Then change `NewTerminalTabAction` and `ConnectToHostAction` so they call `TerminalTabService`, not constructors. This removes direct coupling from actions to files/providers.

### Phase 2: Split `TermLabTerminalVirtualFile`

Target end state:

```text
TermLabTerminalVirtualFile = metadata only
SharedTerminalSession = moved to TerminalSessionLifecycleService
Widget creation = TerminalWidgetFactory
Exit watcher = TerminalSessionMonitor
Tab close/disconnected handling = TerminalPresentationService
```

This is the highest-value cleanup.

### Phase 3: Refactor SSH connection flow

Turn `SshSessionProvider` into an adapter. Move credential resolution, retry, host key verification, progress, and prompt behavior into use cases and ports.

This makes SSH testable without IntelliJ UI.

### Phase 4: Replace `HostStore` with repository + catalog service

Keep `HostStore` temporarily as a compatibility wrapper if needed, but route new code through:

```java
HostCatalogService
HostRepository
HostEvents
```

Tool windows and palette contributors should observe host events rather than owning persistence refresh behavior.

### Phase 5: Make workspace restore provider-driven

Add descriptor support to the terminal provider SPI.

Workspace should not know about local PTY vs SSH directly. It should persist provider descriptors and ask providers to restore them.

Add:

```text
schemaVersion
providerId
providerState
displayTitle
lastKnownCwd
restoreMode: CONNECTED | DISCONNECTED_STUB
```

### Phase 6: Isolate MultiExec platform internals

Move all reflection and `EditorWindow` manipulation into:

```java
IntellijEditorLayoutGateway
```

Then test the MultiExec rules separately from the IntelliJ layout code.

### Phase 7: Contributor hardening

Add:

```text
CONTRIBUTING.md
architecture rules
test command
plugin authoring guide
extension point examples
sample plugin
```

This matters if you want other contributors to work safely without understanding every IntelliJ platform hack.

---

## Practical priority order

Do these first:

1. Add runnable tests.
2. Extract `TerminalSessionDescriptor` and `TerminalTabService`.
3. Stop actions from constructing `TermLabTerminalVirtualFile` and providers directly.
4. Move session lifecycle out of `TermLabTerminalVirtualFile`.
5. Split SSH connection logic into use cases and ports.
6. Introduce repository interfaces for hosts/workspaces/known_hosts.
7. Isolate MultiExec IntelliJ reflection.
8. Add workspace schema versioning and provider-driven restore.

The current codebase is not bad. It is at the point where feature velocity has started to outgrow the first implementation structure. The cleanup should focus less on abstract “DDD purity” and more on one rule: domain/application behavior should be testable without IntelliJ, Swing, Apache MINA, or the filesystem unless that specific adapter is under test.

---

## Source references from review

These are the repository areas and files referenced during the review:

| Area | File / Path |
|---|---|
| Product overview and repository layout | `README.md` |
| Active runtime composition | `BUILD.bazel` |
| Core plugin descriptor | `core/resources/META-INF/plugin.xml` |
| Core build target | `core/BUILD.bazel` |
| SDK build target | `sdk/BUILD.bazel` |
| Terminal provider SPI | `sdk/src/com/termlab/sdk/TerminalSessionProvider.java` |
| Terminal virtual file/session lifecycle | `core/src/com/termlab/core/terminal/TermLabTerminalVirtualFile.java` |
| Terminal editor | `core/src/com/termlab/core/terminal/TermLabTerminalEditor.java` |
| Workspace manager | `core/src/com/termlab/core/workspace/WorkspaceManager.java` |
| Workspace state model | `core/src/com/termlab/core/workspace/WorkspaceState.java` |
| Workspace serializer | `core/src/com/termlab/core/workspace/WorkspaceSerializer.java` |
| MultiExec manager | `core/src/com/termlab/core/terminal/TermLabMultiExecManager.java` |
| SSH plugin descriptor | `plugins/ssh/resources/META-INF/plugin.xml` |
| SSH build target | `plugins/ssh/BUILD.bazel` |
| SSH session provider | `plugins/ssh/src/com/termlab/ssh/provider/SshSessionProvider.java` |
| SSH host store | `plugins/ssh/src/com/termlab/ssh/model/HostStore.java` |
| SSH host model | `plugins/ssh/src/com/termlab/ssh/model/SshHost.java` |
| SSH credential bundle/resolution | `plugins/ssh/src/com/termlab/ssh/credentials/HostCredentialBundle.java` |
| SSH host tool window | `plugins/ssh/src/com/termlab/ssh/toolwindow/HostsToolWindow.java` |
| SSH host tool window factory | `plugins/ssh/src/com/termlab/ssh/toolwindow/HostsToolWindowFactory.java` |
| SSH connect action helper | `plugins/ssh/src/com/termlab/ssh/actions/ConnectToHostAction.java` |
| SSH host store tests | `plugins/ssh/test/com/termlab/ssh/model/HostStoreTest.java` |
| Vault plugin descriptor | `plugins/vault/resources/META-INF/plugin.xml` |
| Vault build target | `plugins/vault/BUILD.bazel` |
| Vault lock manager | `plugins/vault/src/com/termlab/vault/lock/LockManager.java` |
| SFTP plugin descriptor | `plugins/sftp/resources/META-INF/plugin.xml` |
| SFTP build target | `plugins/sftp/BUILD.bazel` |
| SFTP session manager | `plugins/sftp/src/com/termlab/sftp/session/SftpSessionManager.java` |
| Product design spec | `docs/specs/2026-04-08-termlab-workstation-design.md` |
| Makefile build targets | `Makefile` |
