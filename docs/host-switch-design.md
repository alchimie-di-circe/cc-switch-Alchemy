# Host-Switch Design — Step 5 of the Alchemy Refactor

> **Status:** planning only. No application code changes in this PR.
> The mayor must explicitly start the staged convoy before any of the
> design decisions below are implemented.
>
> **Prerequisite:** this step depends on
> [`docs/agent-matrix.md`](./agent-matrix.md) (Step 2),
> [`docs/removal-plan.md`](./removal-plan.md) (Step 3), and
> `docs/ia-design.md` (Step 4, not yet written). The vocabulary in
> `AGENTS.md §2` is the public contract.
>
> **Scope:** introduce a Settings-level **Host Switcher** that lets the
> user point the whole app at one of three host types — `local`,
> `ssh`, `tailscale-ssh` — and route every file / process / agent-CLI
> call through a new `HostAdapter` abstraction so the agent tabs from
> Step 4 work transparently against a remote VPS.

---

## 1. Goals and non-goals

### 1.1 Goals

- Add a **Settings → Host** section exposing a single "Active host"
  selector with three options: `local` (this machine), `ssh` (a remote
  host reachable over plain SSH), `tailscale-ssh` (the same SSH
  transport, but the connection goes over the Tailscale network and
  uses a Tailscale MagicDNS name). All other tabs in the app operate
  against the active host.
- Introduce a `HostAdapter` trait with a `Local` and an `Ssh`
  implementation. The Step-4 agent tab command layer calls the
  adapter; the agent tabs themselves have no awareness of where the
  host runs.
- Persist the host selection in the existing key/value `settings`
  table (`src-tauri/src/database/dao/settings.rs`) so it survives
  restarts and follows the same migration discipline as every other
  persisted preference.
- Provide a clear security model: SSH credentials (key path, optional
  passphrase, control-socket path) are stored in the OS keyring /
  Tauri stronghold, **never** in the SQLite `settings` table and
  **never** logged.

### 1.2 Non-goals (this PR does not)

- Do not implement container or WSL targets. `local` already covers
  WSL2 via the existing `linux_fix.rs` path; remote containers are a
  future host type.
- Do not add per-agent host overrides. The active host is global.
  Per-agent overrides are a follow-up if the user research demands
  it.
- Do not retrofit the legacy proxy/failover subsystem — that is
  removed in `docs/removal-plan.md` (Step 3).
- Do not silently re-implement direct `std::fs` / `std::process::Command`
  call sites that should now go through the adapter. New code routes
  through the adapter; the migration sweep is a separate bead.

---

## 2. Host types

| Host id         | Transport              | Auth                          | When to use                                                                                                                                 |
| --------------- | ---------------------- | ----------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| `local`         | direct std / process   | n/a (current user)            | The cc-switch desktop app is running on the same machine that owns the agent config dirs.                                                   |
| `ssh`           | `ssh -p <port> …`      | SSH key file (or `ssh-agent`) | A reachable remote VPS over the public internet or a LAN.                                                                                   |
| `tailscale-ssh` | `ssh …` over Tailscale | SSH key file (same as `ssh`)  | **SPECULATIVE**: a Tailscale-network peer; this assumes the Tailscale daemon is installed on both ends and MagicDNS resolves the peer name. |

The `Ssh` and `TailscaleSsh` impls share 100% of the runtime code; the
only difference is the default `ProxyCommand` / host alias used when
spawning the SSH control master. From the `HostAdapter`'s point of
view they are the same transport — see §3.4.

---

## 3. `HostAdapter` abstraction

### 3.1 Module layout

```
src-tauri/src/host/
    mod.rs                    # pub trait HostAdapter + the global accessor
    local.rs                  # Local adapter (wraps std::fs / Command)
    ssh.rs                    # Ssh + TailscaleSsh adapters (wrap ssh)
    credentials.rs            # keychain-backed credential store
    error.rs                  # HostError + From<...> impls
```

### 3.2 Trait surface (Rust pseudocode)

```rust
// src-tauri/src/host/mod.rs
pub trait HostAdapter: Send + Sync {
    fn id(&self) -> HostId;

    // --- filesystem ---
    fn read_text_file(&self, path: &HostPath) -> Result<String, HostError>;
    fn write_text_file(&self, path: &HostPath, contents: &str) -> Result<(), HostError>;
    fn read_dir(&self, path: &HostPath) -> Result<Vec<DirEntry>, HostError>;
    fn exists(&self, path: &HostPath) -> Result<bool, HostError>;
    fn metadata(&self, path: &HostPath) -> Result<FileMetadata, HostError>;
    fn remove(&self, path: &HostPath) -> Result<(), HostError>;
    fn create_dir_all(&self, path: &HostPath) -> Result<(), HostError>;

    // --- process ---
    fn run_command(&self, spec: CommandSpec) -> Result<CommandOutput, HostError>;

    // --- path translation ---
    /// Translate a path on this host into a path the agent tab can show
    /// the user. For `Local` this is a no-op; for `Ssh` it is `ssh://user@host/path`.
    fn display(&self, path: &HostPath) -> String;
    /// Inverse of `display` — used when the user types/pastes a remote path
    /// into a CCS form field.
    fn parse_display(&self, s: &str) -> Result<HostPath, HostError>;
}
```

`HostPath` is a new tagged enum (not a raw `PathBuf`) so that
`SshAdapter::write_text_file` can refuse to operate on a
`HostPath::Local` by construction:

```rust
pub enum HostPath {
    Local(PathBuf),
    Remote { user: String, host: String, port: Option<u16>, path: PathBuf },
}
```

### 3.3 Why an enum, not a wrapper

Putting `user/host/port` on the path keeps the call sites uniform
("read this file, please") and prevents accidental cross-host writes
("write the local config to a remote path") — the type system
guarantees a path belonging to host A cannot be passed to host B
without a deliberate conversion.

### 3.4 The `Ssh` adapter — implementation outline

- **Connection model:** maintain a single long-lived SSH control
  master per remote host using `ssh -o ControlMaster=auto -o
ControlPath=~/.ssh/ccs-%r@%h:%p -o ControlPersist=10m`. The control
  socket is created on first use and reused for the rest of the
  session. This avoids paying the SSH handshake cost on every file
  read.
- **File ops:** use `ssh <host> -- cat <path>` for reads and `ssh
<host> -- sh -c 'cat > <path>'` for writes. For directory listing,
  `ssh <host> -- ls -1 -- <path>`. **SPECULATIVE**: we assume the
  remote shell is POSIX. If a Windows-VPS target is ever requested,
  this adapter gains a `posix | powershell` discriminator.
- **Process ops:** use `ssh <host> -- <argv...>` directly. The remote
  exit code is propagated back through the `CommandOutput` return
  value. Stderr is captured but **redacted** — see §6.
- **Path translation:** all paths returned to the frontend are
  rendered as `ssh://user@host[:port]/abs/path` (matches the `scp://`
  URI scheme users already recognise). The frontend uses this string
  for display only; it never parses it back without going through
  `parse_display`.
- **Tailscale-SSH:** the only difference from `Ssh` is the host
  alias. We use `ssh <tailscale-magicdns-name>` instead of an
  IP / FQDN. The `HostId` carries the chosen alias, but the runtime
  code is shared. If the Tailscale daemon is not running on the
  desktop machine, the `Ssh` adapter still works (it just resolves
  the name via system DNS or fails). **SPECULATIVE**: Tailscale
  availability and MagicDNS resolution are not guaranteed; the
  adapter must not block on `tailscale status` and must not auto-install
  Tailscale.

### 3.5 The `Local` adapter

Wraps the existing `std::fs` and `std::process::Command` calls used
in `*_config.rs` modules today. Initially we can provide
straightforward `impl`s; once the migration sweep lands, every
existing `std::fs::read_to_string(&path)` in an agent-config module is
replaced with `app.host().read_text_file(&HostPath::Local(path))`.
The legacy code paths remain in place until the per-agent modules
are touched by their respective beads.

### 3.6 Global accessor

A new field on `AppState` holds the active adapter, swapped in place
when the user changes the host:

```rust
// src-tauri/src/lib.rs (additive — no public surface change in this PR)
pub struct AppState {
    pub host: Arc<dyn HostAdapter>,
    // ... existing fields
}
```

The Tauri command layer from Step 4 (`agent_command(app, op, payload)`)
takes `&AppState` and forwards `app.host` to the per-agent backend
modules. The Step-4 design specifies that the per-agent backend takes
`&dyn HostAdapter` instead of the implicit `std::env::home_dir()` it
uses today.

---

## 4. Settings UI integration

### 4.1 New `Settings → Host` panel

A new `src/components/settings/HostSettingsPanel.tsx` is added next
to `AppVisibilitySettings`, `SkillStorageLocationSettings`, etc., in
`SettingsPage.tsx`. The panel shows:

- A "Active host" radio / segmented control with the three options.
- For `ssh` / `tailscale-ssh`:
  - Host alias (or Tailscale MagicDNS name)
  - Port (default 22)
  - Username
  - SSH key path (defaults to `~/.ssh/id_ed25519`; can be `ssh-agent`
    identity name)
  - "Test connection" button that calls a new
    `host_test_connection` Tauri command and surfaces a toast
- A "Connected as `user@host`" status line and a "Disconnect" /
  "Reset to local" button.

The panel is the **only** place in the UI where the user enters host
details. Every other panel (per-agent tabs, prompts, providers, usage
dashboard) reads the active host from `useSettings().host` and does
not show its own host controls.

### 4.2 State management

- `useSettings()` (`src/hooks/useSettings.ts`) gains a `host` field
  and `setHost(hostId)` / `setSshConfig(ssh: SshConfig)` setters.
- On mount, the panel calls a new `host_get_state` Tauri command
  which returns the active host plus the **redacted** SSH config
  (see §6.3).
- On change, the panel calls `host_set_state({ hostId, sshConfig })`.
  The Rust side validates the SSH config (parses the key path, pings
  the host if reachable) and updates `app.host` atomically — readers
  either see the old adapter or the new one, never a half-applied
  state.

### 4.3 Persistence

The active host is persisted in the existing `settings` table
(`src-tauri/src/database/dao/settings.rs`) under two new keys:

- `host.active_id` → `"local" | "ssh" | "tailscale-ssh"`
- `host.ssh_config_ref` → a UUID that points to the credentials
  store entry. The actual SSH config (user, host, port, key path)
  is **not** in SQLite.

The credentials are stored via a new
`src-tauri/src/host/credentials.rs` module that wraps
`keyring-rs` (or Tauri's secret plugin). On first launch with no
stored credentials, the panel prompts the user. There is no
"remember password" path for SSH passphrases — passphrases are
forwarded to the running `ssh-agent` only and never persisted by
cc-switch.

---

## 5. Mapping onto the agent command layer (Step 4)

The Step-4 design introduces `agent_command(app, op, payload)`. With
this design in place, every backend module that today does:

```rust
let home = dirs::home_dir().unwrap();
let path = home.join(".claude").join("claude.json");
let s = std::fs::read_to_string(&path)?;
```

becomes:

```rust
let path = HostPath::Local(get_claude_config_dir());
let s = app.host.read_text_file(&path)?;
```

For agents that the matrix says have a **Local (project)** scope
(see `docs/agent-matrix.md`), the backend additionally resolves the
project root through the adapter:

```rust
let projects = app.host.list_projects(&agent_id)?; // walks ~/.claude/projects/ etc.
```

`list_projects` and the related `resolve_project_dir(agent_id, hint)`
methods are part of the trait surface — they live next to
`read_text_file` and are the only operations the per-agent backend
modules need in addition to generic file / process ops.

The "every agent tab works transparently against a remote VPS" goal
falls out of the fact that:

- All file ops go through `HostPath`.
- All process ops go through `run_command`.
- All path resolution goes through `list_projects` /
  `resolve_project_dir`.

The frontend `AgentTab` component (Step 4) never sees a path that
isn't pre-displayed through `host.display(path)`; it never spawns a
process directly; it never walks the filesystem. The same
`<AgentTab agentId="claude">` renders identically against `local`,
`ssh`, or `tailscale-ssh`.

---

## 6. Security model

### 6.1 Credential storage

- SSH key **paths** are stored in the keyring entry referenced by
  `host.ssh_config_ref`. The key file itself is never read into
  memory by cc-switch — it is passed to `ssh -i <path>` which handles
  the key natively.
- SSH key **passphrases** are never persisted. If the key is
  encrypted, the user must add it to `ssh-agent` (`ssh-add
~/.ssh/id_ed25519`) before pointing cc-switch at the remote host.
  cc-switch surfaces a clear error if the SSH control master fails
  to authenticate.
- The control-master socket is created with `0600` permissions in
  `$XDG_RUNTIME_DIR/cc-switch/ssh-%r@%h:%p` (Linux) or
  `$TMPDIR/cc-switch/...` (macOS) — never in a world-readable
  location.

### 6.2 What is logged

- **Never** log: key paths, key contents, passphrases, the SSH
  username, the host name, the host port, the content of any file
  read from the remote host, or the stderr of any remote command.
- **Always** log: the active `hostId`, the `HostId::Remote` opaque
  connection id (a UUID, not the host name), the size in bytes of
  files read / written, and the exit code of remote commands.
- Log redaction is enforced in one place:
  `src-tauri/src/host/credentials.rs::redact(s: &str)` is called on
  every log statement that may include remote identifiers. The
  redaction list is the union of the SSH config fields and any
  string that matches `ssh://[^\s]+`.

### 6.3 What the frontend can see

- The active `hostId` (`"local" | "ssh" | "tailscale-ssh"`).
- The SSH config with sensitive fields stripped: the user sees
  `user@host:port` as a single display string; the key path is
  shown only as a basename (e.g. `id_ed25519`); the passphrase is
  never returned from the backend.
- File paths for display, via `host.display(path)`, rendered as
  `ssh://user@host/abs/path`.

### 6.4 Fail-closed behavior

- If the SSH control master cannot be opened (key missing, host
  unreachable, permission denied), the active host stays at its
  previous value. The panel shows the error; the rest of the app
  does not crash and does not silently fall back to `local`.
- If a single `read_text_file` / `run_command` call fails because
  the connection dropped, the adapter attempts one reconnect via
  the control master, and returns a typed `HostError::Disconnected`
  to the caller. The Tauri command surface translates this into a
  user-visible toast; the agent tab marks the affected section as
  "unavailable" rather than showing stale data.
- The host switcher is **not** a sandbox. It assumes the remote
  host is one the user already trusts (their own VPS, not a shared
  host). The UI does not advertise this as a security boundary; a
  follow-up design doc covers multi-tenant trust.

---

## 7. Open questions and risks

### 7.1 Open questions

- **Q1** — Should the host switcher support per-agent hosts (e.g.
  "Claude on local, Codex on my VPS") or stay global? This design
  says **global** for the first cut. Per-agent hosts are a follow-up
  if the user research demands it.
- **Q2** — Should the SSH config be exportable (so a user can move
  their cc-switch profile between machines)? Recommendation: yes,
  but re-import should re-prompt for the key path and never bundle
  the key file itself.
- **Q3** — Do we need a Tailscale-specific "is the daemon running"
  check, or do we trust the user? **SPECULATIVE**: recommend
  opportunistic — if `tailscale status` returns 0 within 1s, show a
  green "Tailscale ready" badge; otherwise show a yellow "Tailscale
  not detected" badge but do not block the host from being selected.
- **Q4** — Where do connection-level timeouts live? Recommend: 5s
  connect, 30s command, 60s file transfer. Per-call overrides via
  `CommandSpec::timeout`.
- **Q5** — How does the Ssh adapter behave on Windows? **SPECULATIVE**:
  cc-switch's Tauri build already supports Windows for local; the
  Ssh adapter would shell out to `ssh.exe` (OpenSSH) and the
  control-master path lives in `%LOCALAPPDATA%`. No
  Windows-specific assumptions are baked into the trait.

### 7.2 Risks

- **R1 — Path encoding.** SSH forces a UTF-8 boundary on paths; the
  Local adapter does not. A `HostPath` containing non-UTF-8 bytes
  must error cleanly on `Ssh` rather than panic. Mitigation:
  `HostPath::Remote` stores the path as `String` and rejects
  non-UTF-8 at construction time.
- **R2 — Partial writes.** If the SSH control master dies mid-write,
  the remote file may be truncated. Mitigation: write to a sibling
  `*.tmp` file and `mv` atomically, mirroring the pattern in
  `services/skill.rs`.
- **R3 — Performance.** Every config read is a separate `ssh cat` if
  we are naive. Mitigation: a short-lived batch API on the trait,
  `run_command_batched(specs: Vec<CommandSpec>) -> Vec<CommandOutput>`,
  plus a transparent read cache (5s TTL) for the most-frequently
  read files (the per-agent MCP config and the agent's
  `settings.json`).
- **R4 — DB migration.** The two new `settings` keys (`host.active_id`
  and `host.ssh_config_ref`) require no schema change — the
  `settings` table is already a free-form key/value store. Migration
  risk is therefore low, but the change must be documented in
  CHANGELOG.
- **R5 — Agent that has a different config layout on the remote
  host.** **SPECULATIVE**: agents like `~/.config/opencode/opencode.json`
  are Linux/macOS paths; on a Windows remote the path differs. For
  the first cut we restrict the Ssh target to Linux/macOS remotes
  and surface a clear error otherwise.
- **R6 — Tailscale availability.** As noted throughout, Tailscale is
  a hard assumption for `tailscale-ssh` to be useful. If the user
  does not have Tailscale installed the host is functionally
  identical to `ssh` (just with a MagicDNS name). Acceptable.
- **R7 — Per-agent project resolution on a remote host.** The agent
  tabs ask the backend "where are the project dirs for Claude?". On
  a remote host this becomes `ls ~/.claude/projects/`. The
  `list_projects` and `resolve_project_dir` methods are the only
  new trait surface beyond generic file / process ops. Their
  implementation must be conservative — do not recurse, do not
  follow symlinks.

---

## 8. Implementation sequencing

This is a planning doc; the implementation lands in the follow-up
convoy. The recommended slicing:

1. **PR-5a — Adapter + Local impl only.** Land the trait, the
   `Local` impl, the global accessor on `AppState`, and the
   `host_get_state` / `host_set_state` commands. No UI change. No
   backend module is migrated. Gating: `cargo build && cargo test`.
2. **PR-5b — Ssh impl + credentials module.** Land `Ssh`,
   `TailscaleSsh`, and `credentials.rs`. The active host can be
   switched via the new commands; no UI change. Add unit tests for
   control-master lifecycle and redaction. Gating: `cargo test`.
3. **PR-5c — Settings panel.** Land `HostSettingsPanel.tsx` and
   the wiring in `SettingsPage.tsx`. Gating: `pnpm typecheck &&
pnpm test:unit`.
4. **PR-5d — Backend migration sweep.** Replace direct `std::fs` /
   `Command` calls in the per-agent backend modules with
   `app.host.*`. This is the only PR that touches the agent tabs'
   backend and must be sequenced after Step 4's
   `agent_command(app, op, payload)` is in place. Gating: every
   pre-existing test still passes plus a new integration test
   "agent tab against a mock `Ssh` adapter".

Each PR keeps `cargo build && cargo test && pnpm typecheck &&
pnpm test:unit` green and lands behind a feature flag
(`CCS_HOST_SWITCHER`) so the rollout is reversible.

---

## 9. Cross-references

- Vocabulary and column names: `AGENTS.md §2`.
- Agent capabilities and per-agent paths: [`docs/agent-matrix.md`](./agent-matrix.md).
- Subsystem retirements: [`docs/removal-plan.md`](./removal-plan.md).
- Agent tab IA (Step 4): `docs/ia-design.md` — **to be written**;
  this doc assumes its `agent_command(app, op, payload)` surface.
- Existing settings DAO: `src-tauri/src/database/dao/settings.rs`
  (free-form `key/value` table; no schema change needed).
- Existing settings UI mount point:
  `src/components/settings/SettingsPage.tsx` (new panel slots in
  next to `AppVisibilitySettings` and `SkillStorageLocationSettings`).
