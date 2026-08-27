# Host-Switch Architecture Plan (Step 5)

> Status: **PLAN ONLY**. No application code changes land in this bead. This document is a design contract for the host-switch refactor; code work ships in a follow-up bead after this plan is merged.

> Prerequisite docs (read first):
>
> - [AGENTS.md](../AGENTS.md) — vocabulary, column/tab names, hand-off contract.
> - [Agent Capability Matrix](./agent-matrix.md) — authoritative column definitions.
> - [Removal Plan](./removal-plan.md) — what subsystems are being retired (this design assumes Gemini / Grok Build / provider-routing are gone or going).
> - [IA Design](./ia-design.md) — the agent tabs that must run transparently against any host.

---

## 1. Goals and non-goals

### 1.1 Goals

1. Add a **Settings-level host switcher** with three host kinds: `local`, `ssh`, `tailscale-ssh`.
2. Define a `HostAdapter` trait with at least `Local` and `Ssh` implementations, exposing the operations the Step 4 agent tabs need.
3. Persist the host selection in the existing local settings file (`src-tauri/src/settings.rs`) — **device-level, never synced** (mirroring `claude_config_dir` / `codex_config_dir` / etc.).
4. Route every file path and shell invocation that the agent tabs need through `HostAdapter`, so that a single tab UI works against `local` and a remote VPS without code change.
5. Keep consistent with the **reusable Rust command layer** from Step 4: the command layer is host-aware via `HostAdapter`, not by branching on host kind per command.
6. Document the security posture: SSH credential handling, no secret logging, fail-closed defaults.

### 1.2 Non-goals

- We are **not** shipping a full SSH terminal emulator or an interactive shell experience. The host switcher covers **config-file and command-execution primitives only**.
- We are **not** adding a new "remote" or "ssh" provider type. Remote hosts are a transport concern, not an agent.
- We are **not** retrofitting every existing Tauri command in this plan. The plan identifies the seams; the refactor lands incrementally.
- We are **not** designing a multi-host fan-out. Only one host is active at a time.

---

## 2. Host kinds and selection semantics

| Kind            | Wire target                                                                                                                                      | When to use                                                                        | Pre-conditions (enforced by Settings UI)                                                                                                                                                            |
| --------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `local`         | The machine running the cc-switch Tauri process.                                                                                                 | Default. Same behavior as today.                                                   | None.                                                                                                                                                                                               |
| `ssh`           | A user-supplied host reachable over plain `ssh(1)`.                                                                                              | When the user manages agent configs on a remote VPS they already SSH into.         | `ssh` / `ssh-keygen` available on `$PATH`; configured key with non-interactive auth.                                                                                                                |
| `tailscale-ssh` | Same as `ssh` but resolved through the Tailscale node hostname (`<node>.<tailnet>.ts.net`) and (by convention) tagged as a Tailscale SSH target. | When the user manages agent configs on a remote VPS only reachable over Tailscale. | **SPECULATIVE** — assumes Tailscale is installed and the node is routable from the local machine. We do not bundle a Tailscale client; we just resolve the target string and let `ssh` do the work. |

### 2.1 Why `ssh` and `tailscale-ssh` are the same adapter

The transport is the same: an `ssh(1)` invocation. The only difference is the **target string** the user enters (and, optionally, which key file is preferred). Tailscale is not a wire protocol we re-implement; it is a DNS / ACL layer on top of SSH. Keeping `ssh` and `tailscale-ssh` as separate _kinds_ but a single `Ssh` adapter:

- Lets the UI label them clearly so the user knows "this connection will go over Tailscale".
- Lets us store a per-kind default key path / default identity if we want to (not required in v1).
- Avoids inventing a `TailscaleAdapter` whose only job is to call `ssh` with a different host.

### 2.2 What "active host" means

There is **exactly one active host** at any moment. The active host applies to:

- Reading the **global** config dir of the selected agent (e.g. `~/.claude/`, `~/.codex/`).
- Reading the **project** config dir (e.g. `<cwd>/.claude/`).
- Running the agent CLI for Version/Update and any tab that shells out.
- Listing skill / hook / rule / MCP / plugin artifacts.

When the user switches hosts, every open agent tab should re-load against the new host. We do **not** silently merge local and remote state; the active host is the only source of truth for the UI.

---

## 3. Where the host selection lives in Settings

### 3.1 Frontend: `src/components/settings/SettingsPage.tsx`

The current `SettingsPage` already has a `Tabs` with values `general`, `proxy`, `auth`, `advanced`, `usage`, `about` (`SettingsPage.tsx:222`). The host switcher belongs as a new dedicated tab value `host` inserted between `general` and `proxy`:

```
TabsList order: general | host | proxy | auth | advanced | usage | about
```

Rationale: host is a device-level routing decision that the user should set once and rarely change, but it is not "advanced" — it gates every other tab, so it should sit near the top.

The new `HostTabContent` (new file `src/components/settings/HostTabContent.tsx`) owns:

- A radio group for host kind (`local` | `ssh` | `tailscale-ssh`).
- A form for the SSH / Tailscale target, identity file path, port, username.
- A "Test connection" button that invokes a new Tauri command `host_test_connection`.
- A read-only "Active host summary" panel that shows the resolved target (and a hostname banner at the top of every agent tab so the user always sees _which_ host they are editing).

### 3.2 Frontend: typing

Add a new shared type in `src/types/host.ts` (or wherever the other settings types live — keep it adjacent to the settings type module if one exists):

```ts
export type HostKind = "local" | "ssh" | "tailscale-ssh";

export interface SshHostConfig {
  // Display label only; never logged.
  label: string;
  // user@host:port or user@host (no scheme).
  target: string;
  // Absolute path to the identity file on the LOCAL machine.
  identityFile?: string;
  // Optional port override; defaults to 22.
  port?: number;
  // Optional strict-host-key-checking override; default "accept-new".
  // We refuse "no" at the UI level (see §7).
  knownHostsPolicy?: "accept-new" | "yes";
}

export type HostConfig =
  | { kind: "local" }
  | { kind: "ssh"; ssh: SshHostConfig }
  | { kind: "tailscale-ssh"; ssh: SshHostConfig };

export interface HostSettings {
  active: HostConfig;
  // Recent target list (no secrets), capped at 5.
  recent: SshHostConfig[];
}
```

`HostSettings` is **never** returned to the frontend with secrets; the Rust side is responsible for stripping anything sensitive (we don't store any in v1 — see §7.1).

### 3.3 Backend: persistence in `src-tauri/src/settings.rs`

Persist via the same `AppSettings` struct used today. Add a single field:

```rust
#[serde(default, skip_serializing_if = "Option::is_none")]
pub host: Option<HostSettings>,
```

Constraints, mirroring the existing `*_config_dir` fields:

- `host` lives on the device-local `~/.cc-switch/settings.json` (see `settings.rs:565` `settings_path()`). It is **not** in `database/dao/settings.rs` because that path is for synced settings and host selection must be per-device.
- `host` is gated by the same `get_settings_for_frontend` / `update_settings` / `mutate_settings` helpers in `settings.rs:757-795`. We do not invent a new persistence path.
- Add `normalize_paths`-style cleanup: trim strings, reject empty `target` on `ssh` / `tailscale-ssh` kinds, cap `recent` at 5.

### 3.4 New Tauri commands

Add a `host` submodule under `src-tauri/src/commands/` (parallel to `settings.rs`, `proxy.rs`, `webdav_sync.rs`):

| Command                | Purpose                                                                                                                                                                                                                                        |
| ---------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `host_get_settings`    | Return the current `HostSettings` (no secrets, since there are none in v1).                                                                                                                                                                    |
| `host_set_settings`    | Persist a new `HostSettings`. Validates, normalizes, writes via `mutate_settings`.                                                                                                                                                             |
| `host_test_connection` | Synchronous-ish probe: run `ssh -o BatchMode=yes -o ConnectTimeout=5 <target> true`. Returns `{ ok: bool, latency_ms?: number, error?: string }`. **Never** includes the identity file contents or any stdout beyond what `ssh` itself prints. |
| `host_active_kind`     | Cheap getter for the frontend banner; returns just the `HostKind` string.                                                                                                                                                                      |
| `host_list_recent`     | Returns the `recent` list (display only).                                                                                                                                                                                                      |

These are deliberately small. Anything that _uses_ the adapter goes through the command layer (§5), not through new `host_*` commands.

---

## 4. `HostAdapter` design

### 4.1 Trait surface

The adapter must cover what every per-agent tab needs. Mapping each tab to the primitive it requires (vocabulary from the [Agent Capability Matrix](./agent-matrix.md)):

| Tab            | File ops needed                                                                 | Command exec needed                                                    |
| -------------- | ------------------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| Context Files  | read / write / list a single text file in global or project scope.              | —                                                                      |
| Skills         | list a directory of skill folders; read/write `SKILL.md` and any sibling files. | —                                                                      |
| Version/Update | read a version string from a config file.                                       | run `<cli> --version`; optionally run the agent's self-update command. |
| Hooks          | read/write a JSON / TOML / YAML config file.                                    | —                                                                      |
| Rules          | read/write a rules file in project scope.                                       | —                                                                      |
| MCP            | read/write the MCP section of the agent config.                                 | —                                                                      |
| Plugins        | list a plugin directory; read `plugin.json` / manifest.                         | install / remove plugin (some agents need this).                       |

That collapses to four primitives, plus a path resolver:

```rust
// src-tauri/src/host/mod.rs

use std::path::{Path, PathBuf};
use async_trait::async_trait;
use serde::{Deserialize, Serialize};

/// A path on the active host. Local paths are normal `PathBuf`; remote
/// paths are tagged with the host id so we never mix them.
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(tag = "scope", rename_all = "snake_case")]
pub enum HostPath {
    /// Resolved against the local machine.
    Local(PathBuf),
    /// Resolved against the active remote host; the path is relative to
    /// the user's home dir on that host (or absolute if absolute).
    Remote { host_id: String, path: String },
}

#[derive(Debug, thiserror::Error)]
pub enum HostError {
    #[error("io error on {path}: {source}")]
    Io { path: String, source: std::io::Error },
    #[error("ssh command failed: {0}")]
    Ssh(String),
    #[error("host not configured: {0}")]
    Unconfigured(String),
    #[error("path escapes host root: {0}")]
    EscapesRoot(String),
}

#[async_trait]
pub trait HostAdapter: Send + Sync {
    /// Stable identifier for the host ("local", or "<user>@<host>").
    fn id(&self) -> &str;

    /// Display label for the UI banner ("Local", "vps-user@box-1").
    fn label(&self) -> &str;

    /// Read a UTF-8 text file.
    async fn read_text(&self, path: &HostPath) -> Result<String, HostError>;

    /// Write a UTF-8 text file, creating parents as needed.
    async fn write_text(&self, path: &HostPath, contents: &str) -> Result<(), HostError>;

    /// List immediate children of a directory (names only, no recursion).
    async fn list_dir(&self, path: &HostPath) -> Result<Vec<String>, HostError>;

    /// Check existence.
    async fn exists(&self, path: &HostPath) -> Result<bool, HostError>;

    /// Run a command, returning stdout. Stderr is captured but never
    /// returned in a way that could echo secrets — see §7.
    async fn run(&self, cmd: HostCommand) -> Result<HostCommandOutput, HostError>;

    /// Resolve a logical "global config dir" for an agent on this host.
    /// This is the single source of truth that replaces the per-agent
    /// `get_<agent>_dir()` overrides (see AGENTS.md §4) when the host
    /// is remote. For local, it falls back to the existing helpers.
    fn resolve_agent_global_dir(&self, agent_id: &str) -> HostPath;

    /// Resolve a "project config dir" for an agent given a project root
    /// on the active host.
    fn resolve_agent_project_dir(&self, agent_id: &str, project: &HostPath) -> HostPath;
}

#[derive(Debug, Clone)]
pub struct HostCommand {
    pub program: String,
    pub args: Vec<String>,
    /// Hard cap on runtime; default 30s, max 5 min.
    pub timeout: std::time::Duration,
    /// If true, the command runs in an interactive allocation (only for
    /// agents that need a TTY — we do not use this in v1; see §7.5).
    pub interactive: bool,
}

#[derive(Debug, Clone)]
pub struct HostCommandOutput {
    pub exit_code: i32,
    pub stdout: String,
    pub stderr: String,
}
```

### 4.2 `Local` implementation

A thin wrapper over `tokio::fs` and `tokio::process::Command`. Every `HostPath::Local(p)` becomes a direct `p` call. `resolve_agent_global_dir` defers to the existing `get_<agent_id>_dir()` helpers (e.g. `get_claude_override_dir` at `settings.rs:901`) so today's behavior is preserved.

```rust
pub struct LocalAdapter;

#[async_trait]
impl HostAdapter for LocalAdapter { /* tokio::fs + tokio::process */ }
```

### 4.3 `Ssh` implementation

A wrapper over `tokio::process::Command` that invokes the system `ssh(1)` binary. We intentionally do **not** vendor an in-process SSH library in v1:

- All supported platforms ship OpenSSH.
- Key handling stays where the OS / `ssh-agent` already manages it.
- We avoid taking on the burden of validating host keys ourselves.

```rust
pub struct SshAdapter {
    pub target: SshHostConfig,
}

#[async_trait]
impl HostAdapter for SshAdapter {
    async fn read_text(&self, path: &HostPath) -> Result<String, HostError> {
        let remote = require_remote(path)?;
        let out = self.run_ssh_capture(&["cat", "--", &remote.path]).await?;
        if out.exit_code != 0 { return Err(HostError::Ssh(out.stderr)); }
        Ok(out.stdout)
    }
    // write_text:  `ssh ... -- sh -c 'cat > PATH'`
    // list_dir:    `ssh ... -- ls -1 PATH`
    // exists:      `ssh ... -- test -e PATH; echo $?`
    // run:         `ssh ... -- <program> <args...>`
}
```

All `ssh` invocations pass through a single helper that:

1. Refuses to include the identity file contents in any log.
2. Always passes `-o BatchMode=yes` so a missing key fails fast instead of prompting on a TTY we don't have.
3. Passes `-o ConnectTimeout=5` and `-o ServerAliveInterval=15` by default.
4. Strips the password-less `-o` flags the user didn't set; we never pass `PreferredAuthentications=password` (see §7.2).

The `SshAdapter` is **stateless**; the per-host config (target, identity, port) is owned by a `HostRegistry` (next section) that hands out the right adapter.

### 4.4 `HostRegistry` and the active host

```rust
// src-tauri/src/host/registry.rs

pub struct HostRegistry {
    local: LocalAdapter,
    active: Arc<RwLock<Box<dyn HostAdapter>>>,
    cached_ssh: Arc<RwLock<Option<SshAdapter>>>,
}

impl HostRegistry {
    /// Get the adapter for the currently active host. Always returns
    /// `Local` if no SSH host is configured or if the configured host
    /// failed to initialize (fail-closed; see §7.6).
    pub async fn active(&self) -> Box<dyn HostAdapter> { ... }

    /// Rebuild the active adapter from `AppSettings.host`. Called from
    /// `host_set_settings` and at startup.
    pub fn reload(&self, settings: &AppSettings) -> Result<(), HostError> { ... }
}
```

The registry is a `OnceLock<HostRegistry>` so the rest of the codebase can pull the active adapter without plumbing through every layer:

```rust
pub fn host_registry() -> &'static HostRegistry { ... }
```

### 4.5 Routing file paths and commands

Replace direct `std::fs` and `std::process::Command` calls in the agent backends with adapter calls. Concretely:

| Today                                                              | After                                                                                                   |
| ------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------- |
| `std::fs::read_to_string(get_claude_dir()?.join("settings.json"))` | `host_registry().active().await.read_text(&host_path).await`                                            |
| `std::process::Command::new("claude").arg("--version").output()`   | `host_registry().active().await.run(HostCommand { program: "claude", args: ["--version".into()], .. })` |
| `<project>/.claude/CLAUDE.md` resolved by hand                     | `active.resolve_agent_project_dir("claude-code", &project_path)`                                        |

This is the only change the agent-tab code needs to be host-agnostic. The tab UI does not branch on host kind; it calls the same command regardless.

**SPECULATIVE** — we may discover, when the Step 4 code lands, that some agent backends cache resolved paths at struct-construction time. We will need a small refactor pass to make those lazy. Out of scope to enumerate in this plan; the refactor is mechanical and lands with the adapter.

---

## 5. Reusable command layer integration (Step 4 contract)

The Step 4 IA introduces a single Rust command layer that dispatches to the per-agent backend modules. The shape (per [Removal Plan](./removal-plan.md) and the matrix) is roughly:

```rust
// src-tauri/src/commands/agent.rs (proposed by Step 4)
pub async fn agent_command(
    agent_id: &str,
    scope: AgentScope,        // Global | Project(HostPath)
    op: AgentOp,              // ReadConfig | WriteConfig | ListSkills | ...
    payload: serde_json::Value,
) -> Result<serde_json::Value, AppError>;
```

This plan adds one new argument:

```rust
pub async fn agent_command(
    agent_id: &str,
    host: HostContext,        // <-- new
    scope: AgentScope,
    op: AgentOp,
    payload: serde_json::Value,
) -> Result<serde_json::Value, AppError>;
```

Where `HostContext` is either `HostContext::Implicit` (use the active host from the registry) or `HostContext::Override(Box<dyn HostAdapter>)` (used in tests and by the `host_test_connection` command, which should never depend on the active host).

The default is `Implicit`. The frontend never specifies a host — it gets the active one for free. Tests inject an `Override` with a fake adapter to avoid touching the filesystem or shelling out.

### 5.1 Why the command layer is host-aware, not host-branching

`agent_command` does **not** dispatch `if host.kind == "ssh" { ... } else { ... }`. It resolves the adapter once, then routes every file / process call through it. The per-agent backend modules do not need to know whether the host is local or remote — they call `adapter.read_text(...)` and get back bytes from the right place.

This is the contract that makes Step 4 and Step 5 compose. If Step 4 ships with any code that calls `std::fs` directly, it will silently fail to work over SSH; the adapter-aware refactor must land alongside, or immediately after, the command-layer consolidation.

---

## 6. Path handling invariants

These are the rules every refactor under this plan must follow. They exist to keep the abstraction honest.

1. **A `HostPath` never crosses a host boundary.** A path produced for `Local` cannot be passed to `Ssh`, and vice versa. The `SshAdapter` reads/writes only `HostPath::Remote`; the `LocalAdapter` reads/writes only `HostPath::Local`. Mixing is a programmer error caught at the type level.
2. **Remote paths are strings, not `PathBuf`.** A `PathBuf` on the local machine is not a valid `PathBuf` on the remote. Treating it as a string forces every consumer to think about it.
3. **No absolute path the user types into a Settings field is used as-is on a remote host.** We always resolve `~` against the _remote_ home dir, computed by `ssh ... -- sh -c 'echo $HOME'`. This is cached per session.
4. **Project dir resolution is lazy.** When the user picks a project in the UI, the path is resolved through `adapter.resolve_agent_project_dir(agent_id, project)`. The agent backend never sees the raw project path; it sees the host-resolved child.
5. **The registry is the only place that knows about the active host.** No module may import `AppSettings` and read `host` directly. They must call `host_registry().active()`.

---

## 7. Security notes

### 7.1 What we store

In v1 we store **no secrets** in `HostSettings`:

- The identity file **path** is stored, not the key.
- We do not support password auth (see §7.2), so there is no password to store.
- We do not support `ProxyCommand` or custom `ssh` configs; the user is expected to have `~/.ssh/config` set up if they want anything fancy.

The `recent` list contains only `SshHostConfig` (label, target, identity path, port, policy). Nothing in it is a secret.

If we later add secret-bearing fields (e.g. an encrypted key passphrase), they will live in the OS keychain via `keyring-rs` (SPECULATIVE — not in v1). They will **not** be written to `settings.json`.

### 7.2 Authentication: key-only, fail-closed

- We refuse to send `PreferredAuthentications=password` or `keyboard-interactive`. SSH is invoked with the system default auth methods, which means whatever the user has configured (typically `publickey`).
- The "Test connection" probe uses `BatchMode=yes`, so a host that requires password auth fails immediately instead of hanging.
- If the configured identity file does not exist or is not readable by the current user, `host_set_settings` rejects the save with a clear error.

### 7.3 Host key handling

- Default policy: `accept-new` (TOFU). The user is shown a one-time prompt the first time they connect to a new host, just like vanilla `ssh`.
- We do **not** ship a `StrictHostKeyChecking=no` path. There is no UI affordance for it. This is intentional and fail-closed: a user who really wants to disable it can edit `~/.ssh/config` themselves; the application will not do it for them.
- On a `HOST KEY VERIFICATION FAILED` error, we surface the message verbatim from `ssh` and refuse to retry.

### 7.4 Secret hygiene in logs

- `HostSettings` is logged with `#[serde(skip_serializing)]` on any future secret field. In v1 there is nothing to skip.
- `HostCommand` and `HostCommandOutput` are **never** serialized into log lines. Logging of the adapter is limited to `tracing::debug!` with `host.id()` and `op` kind; never the path, never the args, never the stdout/stderr. Where stderr is needed for error messages, we truncate to 1 KiB and re-check for obvious secret markers (`PRIVATE KEY`, `BEGIN `, `token=`) before including it in a user-facing error.
- The "Test connection" command returns the exit code and a sanitized error string. It does **not** include the identity file path or any `ssh` arguments.

### 7.5 Interactive commands

- `HostCommand::interactive` exists in the trait for future use but is not used by any v1 caller. The agent tabs in Step 4 do not need a TTY.
- If a future caller sets `interactive: true`, the `SshAdapter` must add `-tt` and route stdin/stdout through the Tauri window. That work is out of scope for v1.

### 7.6 Fail-closed defaults

- On startup, if the persisted `host` is `ssh` / `tailscale-ssh` but the configured `target` fails a connection probe, we **fall back to `Local`** and log a warning. The UI banner shows "Local (fallback — remote host unreachable)" so the user is never silently working on the wrong host.
- `host_set_settings` is atomic: validation failures do not partially apply.
- The `SshAdapter` never silently retries on auth failure. One attempt, one error, user-visible.

### 7.7 SPECULATIVE items

The following are not confirmed against an authoritative source and must be re-checked before code lands:

- **Tailscale availability** — we assume the local machine has the `tailscale` CLI installed and the remote node is routable. We do not auto-install or auto-configure Tailscale. If the user picks `tailscale-ssh` and `tailscale status` does not show the node, the test-connection command fails with a clear "node not visible to Tailscale" error.
- **SSH key passphrase via keychain** — listed in §7.1 as future work. Not in v1.
- **Cross-platform `ssh(1)` location** — assumed to be on `$PATH` on macOS, Linux, and Windows 10+ (OpenSSH is bundled). If a future Windows build runs on a stripped image, `SshAdapter` must report a clear "ssh not found" error.
- **Latency budgets** — assumed to be acceptable for synchronous tab loads. If a remote `read_text` is too slow for the existing UI loading patterns, we will need to add a "loading remote host" affordance and possibly a short-lived cache. Out of scope for v1; track as a follow-up.

---

## 8. Migration path and what this plan does NOT touch

This plan is intentionally additive.

- **Adds**: `src-tauri/src/host/`, the `HostSettings` field on `AppSettings`, the `HostTabContent` UI, the new Tauri commands, and the `HostAdapter` plumbing.
- **Does not change**: any existing per-agent backend module, the database schema, the provider-routing subsystem being removed in [Removal Plan](./removal-plan.md), or the IA from [IA Design](./ia-design.md).
- **Assumes**: Steps 2–4 are merged or merging. The `agent_command` shape in §5 is a proposal; if Step 4 lands with a different name for the same thing, we follow Step 4's naming and only add the `host` argument.

The refactor lands incrementally after this plan is merged:

1. Land the registry + `Local` adapter behind a feature flag. Nothing changes for existing callers.
2. Land the `Ssh` adapter and the `host_*` commands. Still nothing changes for existing callers.
3. Migrate one agent backend (suggest Claude Code) to use the adapter. Validate end-to-end against a real SSH host.
4. Migrate the rest. Each migration is one small PR.
5. Turn the feature flag on by default; remove the legacy `std::fs` / `Command` call sites in a final sweep.

---

## 9. Open questions for the convoy owner

1. **Tailscale auth**: do we want to surface a "use `tailscale ssh` instead of `ssh`" toggle, or always use plain `ssh` against the `<node>.ts.net` hostname? v1 assumes the latter; the former would require a separate adapter.
2. **Multi-host**: out of scope for v1, but worth confirming. If a user wants to manage two VPSes from one cc-switch instance, do we want a "host profiles" picker (à la SSH config) or one global active host? Plan assumes the latter.
3. **Connection multiplexing**: `ssh` can multiplex sessions over a single connection. Should the registry hold a long-lived `ssh -M` control socket? v1 says no (one command per invocation, simpler failure model); a follow-up could add it for latency.
4. **Audit log**: should every `HostAdapter::run` invocation be written to a per-user audit log for review? v1 says no, but the trait is shaped to allow it later (add a `&AuditContext` argument in a v2).

---

## 10. Acceptance criteria for this plan

This plan is **done** when:

1. `docs/host-switch-design.md` is committed to the bead's branch and pushed.
2. It cross-references the [Agent Matrix](./agent-matrix.md), the [Removal Plan](./removal-plan.md), and the [IA Design](./ia-design.md) (even though IA Design is the next bead, the contract for the command layer is here).
3. A PR is opened against `convoy/cc-switch-alchemy-refactor-plan-steps-2-/12603eab/head`.
4. `gt_done` is called with the PR URL.

The plan is **not** a code change. No `src-tauri/src/**` or `src/**` file is modified by this bead.
