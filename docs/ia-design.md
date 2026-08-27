# IA Design — Step 4 of the Alchemy Refactor

> **Status:** planning only. Do **not** start any code work in this PR. The
> mayor must explicitly start the staged convoy before any of the design
> changes below are implemented.
>
> **Scope:** replace the current `AppSwitcher` + `ProviderList` + per-agent
> provider-form subsystem with an **agent-first** information architecture:
> a single AGENT selector, a fixed set of shared per-agent tabs, one
> reusable Rust command layer, and one reusable frontend tab component.
>
> **Inputs (already committed):**
>
> - `AGENTS.md` — vocabulary and hand-off contract.
> - `docs/agent-matrix.md` (Step 2) — per-agent capability table that
>   drives the registry data structures and the visible tab set.
> - `docs/removal-plan.md` (Step 3) — defines which subsystems the IA
>   replaces (`AppSwitcher`, `ProviderList`, `components/providers/`,
>   `proxy/`, `failover`, `session_manager`, the deleted agents Gemini
>   and GrokBuild).
> - `docs/host-switch-design.md` (Step 5) — supplies the `HostAdapter`
>   abstraction this doc references but does not redesign.
>
> **Output:** when this bead closes, the implementation bead will have
> a single design contract — this document — to land in one or more
> code PRs.

---

## 0. Design goals

1. **One agent at a time.** A user picks an agent (Claude Code, Codex,
   OpenCode, …) and sees the same tab set for every agent, with each
   tab populated from the per-agent backend module for that agent.
2. **Tabs are data, not branches.** The same `<AgentTab>` component
   renders for every agent and every tab key. Per-agent differences are
   expressed in the central capability matrix (`AGENT_REGISTRY` /
   `agent_matrix()`), not in `if (appId === "pi") { … }` branches
   inside the component tree.
3. **One Rust command layer, one keyed dispatch.** Every per-agent
   operation is reachable through a single command interface
   (`agent_command(agent_id, op, payload)`) that keys into the same
   per-agent backend modules that exist today (`<agent>_config.rs`,
   `mcp/<agent>.rs`, `services::skill::SkillService`, `services::prompt`).
   No new per-agent command files.
4. **Consolidate, don't multiply.** Components that already operate
   agent-agnostically (`UnifiedMcpPanel`, `UnifiedSkillsPanel`, the
   shared `AppToggleGroup`) are the primitives; panels that hard-code
   one agent (`WorkspaceFilesPanel`, per-agent form components) are
   retired or generalized.
5. **Aligned with AGENTS.md vocabulary.** Tab labels, scope names, and
   per-agent column names are exactly the ones defined in AGENTS.md §2
   and reused in `docs/agent-matrix.md`.

---

## 1. The AGENT selector

### 1.1 What it replaces

- `src/components/AppSwitcher.tsx` — kept as the **visual** selector
  (icon + label + active highlight) but wired to the new
  `AGENT_REGISTRY` and the new `selectedAgent` state model (see §1.3).
- `src/components/providers/ProviderList.tsx` — retired as a
  page-level concept. The "list of providers for this agent" is folded
  into the new **Provider** tab (see §2.4).

### 1.2 Agent registry (the single source of truth)

The registry replaces the scattered `APP_IDS`, `MCP_APP_IDS`,
`SKILLS_APP_IDS`, `PROXY_APP_IDS`, `ADDITIVE_APP_IDS`, `APP_ICON_MAP`
constants in `src/config/appConfig.tsx:19–200` with one typed table
keyed by `AppId`.

#### Frontend shape (TypeScript)

```ts
// src/config/agentRegistry.ts  (new)

import type { AppId } from "@/lib/api/types";

export type Scope = "global" | "local";

export type TabKey =
  | "context-files"
  | "skills"
  | "version"
  | "hooks"
  | "rules"
  | "mcp"
  | "plugins"
  | "prompts"
  | "providers";

export interface TabSupport {
  /** Whether the tab is rendered at all for this agent. */
  enabled: boolean;
  /** Which scopes are editable. Empty array = view-only. */
  scopes: Scope[];
  /**
   * If true, the tab is rendered as a "Coming Soon" placeholder.
   * Used during the gradual rollout while the per-agent backend
   * module is being wired up (matrix cells marked SPECULATIVE).
   */
  placeholder?: boolean;
}

export interface AgentDescriptor {
  /** Stable id, kebab-case, matches src-tauri/src/app_config.rs::AppType::as_str. */
  id: AppId;
  /** Human label, matches the existing APP_ICON_MAP label. */
  label: string;
  /** CLI binary name (matrix column "CLI binary"). */
  cliBinary: string;
  /** Path resolvers used by the <ContextFiles> and other filesystem tabs. */
  paths: {
    /** User-wide config dir (matrix "Global config"). */
    globalConfigDir: string;
    /** Project-local config dir or "." if the agent is global-only. */
    projectConfigDir: string;
  };
  /** Visual identity — keeps AppSwitcher working without a parallel icon table. */
  iconKey: string;
  activeClass: string;
  badgeClass: string;
  /** Which tabs this agent supports, in render order. */
  tabs: Record<TabKey, TabSupport>;
}

export const AGENT_REGISTRY: Record<AppId, AgentDescriptor> = {
  claude: { … },
  "claude-desktop": { … },
  codex: { … },
  opencode: { … },
  openclaw: { … },
  hermes: { … },
  pi: { … },
};

/** Stable render order for the selector and tab list. */
export const AGENT_ORDER: AppId[] = [
  "claude", "codex", "opencode", "openclaw", "hermes", "pi",
];

/** Visible-by-default flags (moves DEFAULT_VISIBLE_APPS here). */
export const DEFAULT_VISIBLE_AGENTS: Record<AppId, boolean> = { … };
```

The descriptors are hand-written today (this is a planning doc) and
each row is checked against `docs/agent-matrix.md` Part A & Part B. In
a follow-up implementation step the registry can be generated by
reading the matrix at build time, but that automation is **out of
scope** for this bead.

#### Backend shape (Rust)

```rust
// src-tauri/src/agent.rs  (new)

use serde::Serialize;
use crate::app_config::AppType;

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize)]
#[serde(rename_all = "kebab-case")]
pub enum Scope { Global, Local }

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize)]
#[serde(rename_all = "kebab-case")]
pub enum TabKey {
    ContextFiles, Skills, Version, Hooks, Rules, Mcp, Plugins, Prompts, Providers,
}

#[derive(Debug, Clone, Serialize)]
pub struct TabSupport {
    pub enabled: bool,
    pub scopes: Vec<Scope>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub placeholder: Option<bool>,
}

#[derive(Debug, Clone, Serialize)]
pub struct AgentDescriptor {
    pub id: &'static str,
    pub label: &'static str,
    pub cli_binary: &'static str,
    pub global_config_dir: &'static str,
    pub project_config_dir: &'static str,
    pub icon_key: &'static str,
    pub tabs: std::collections::BTreeMap<TabKey, TabSupport>,
}

/// Single source of truth for the capability matrix on the backend.
/// Mirrors src/config/agentRegistry.ts on the frontend.
pub fn agent_matrix() -> Vec<AgentDescriptor> { … }

pub fn descriptor_for(app: AppType) -> AgentDescriptor { … }
```

The descriptor is the **same** data on both sides of the IPC. The
frontend **does not** hard-code the matrix; it calls
`agent_matrix()` once on startup, caches the result in a React
context, and the `<AgentTab>` component reads the cached descriptor for
the currently selected agent.

> **Vocabulary alignment (AGENTS.md §2):** `tabs` keys use the same
> exact labels as the matrix columns: **Context Files**, **Skills**,
> **Version/Update** (typed `version` in the enum), **Hooks**,
> **Rules**, **MCP**, **Plugins**. The two extra tabs (**Prompts** and
> **Providers**) are explicitly called out in §2.4 and §2.5 below.

### 1.3 Selection state model

Replace the `activeApp` boolean state in `src/App.tsx:180` with a
`selectedAgent: AppId` and a `selectedTab: TabKey` (default: the first
enabled tab in `AGENT_REGISTRY[selectedAgent].tabs`).

- The selector persists `selectedAgent` in `localStorage["cc-switch-last-agent"]`
  (replaces the existing `cc-switch-last-app` key with a one-shot
  migration in the IA landing PR).
- Per-agent `selectedTab` is **not** persisted in v1. The default tab
  is always the first enabled one; this is a deliberate
  simplification to avoid hiding "what does each agent actually offer".
- The current "visible apps" toggle UI in Settings →
  `DEFAULT_VISIBLE_APPS` is renamed to `DEFAULT_VISIBLE_AGENTS` and
  its key set is `AppId`.

---

## 2. The shared per-agent tabs

Every agent renders the same set of tab buttons. The set of visible
tabs is the intersection of `Object.keys(AGENT_REGISTRY[id].tabs)`
(filtered by `enabled`) and the user's "tabs I want to see" filter
(out of scope for v1 — every enabled tab is shown).

### 2.1 Tab: Context Files

- **Matrix mapping:** `docs/agent-matrix.md` rows
  "Context Files (Global)" / "Context Files (Local)".
- **Scope support:** per agent; the tab renders two sub-panels
  (Global / Local) when both are present, one sub-panel otherwise.
- **Backed by:** a new `op_list_context_files` /
  `op_read_context_file` / `op_write_context_file` / `op_delete_context_file`
  set of `AgentOp` variants (see §3.2). The implementation dispatches
  to a small helper per agent (e.g. `qwen::list_context_files(scope)`,
  `nanobot::list_context_files(scope)`) — these helpers wrap the
  filesystem reads/writes the matrix already documents.
- **Reuses:** the file-editor UI in
  `src/components/workspace/WorkspaceFileEditor.tsx` and the
  `read_workspace_file` / `write_workspace_file` /
  `read_daily_memory_file` / `write_daily_memory_file` commands under
  `src-tauri/src/commands/workspace.rs` (which are themselves
  generic — they take a directory and a filename, not an `AppType`).
- **Retires:** `src/components/workspace/WorkspaceFilesPanel.tsx` —
  its hard-coded list of 9 filenames (`AGENTS.md`, `SOUL.md`, `USER.md`,
  `IDENTITY.md`, `TOOLS.md`, `MEMORY.md`, `HEARTBEAT.md`,
  `BOOTSTRAP.md`, `BOOT.md`) becomes a per-agent override table inside
  `agentRegistry.ts`; OpenClaw remains the canonical example.

### 2.2 Tab: Skills

- **Matrix mapping:** "Skills (Global)" / "Skills (Local)".
- **Backed by:** the existing `services::skill::SkillService` and the
  `commands/skill.rs` commands. **No new backend code in the IA
  PR.** The `SkillService::sync_to_app_dir` dispatch in
  `services/skill.rs:2241–2309` is already a per-agent "where do
  skills go" table; the IA reuses it as-is.
- **Reuses:** `src/components/skills/UnifiedSkillsPanel.tsx`
  unchanged. Its existing iteration over `SKILLS_APP_IDS` becomes
  iteration over `Object.keys(AGENT_REGISTRY).filter(id => AGENT_REGISTRY[id].tabs.skills.enabled)`.
- **Retires:** the duplicate `SkillsPage.tsx` (legacy install flow).
  The unified panel already covers install, uninstall, update, repos,
  and zip import. The removal plan lists `SkillsPage.tsx` deletion as
  part of PR-C sub-3.

### 2.3 Tab: Version / Update

- **Matrix mapping:** "Version/Update".
- **Scope support:** Global only (no per-project CLI version).
- **UI:** a single read-only card showing the agent's `--version`
  output, plus a "Check for update" button. For agents that document
  a self-update subcommand (Goose `goose update`, Nanobot
  `pip install -U nanobot-ai`, Qwen npm upgrade) the button calls
  `agent_command(agent_id, Op::RunUpdate, {})` and shows a
  progress + result toast.
- **Backed by:** `Op::GetVersion` (one shell-out to
  `<cli_binary> --version`) and `Op::RunUpdate` (per-agent subcommand
  string from the matrix's "Version/Update" cell, or `SPECULATIVE`
  marker if no canonical command exists).
- **New Rust:** small `agent::version::read_version(binary) -> String`
  helper and a per-agent "update command" table. The
  `SPECULATIVE` markers in `agent-matrix.md` (Qwen, Mastra, dsh,
  Antigravity update transport) stay as `SPECULATIVE` — the UI shows
  "No documented update command" and the bead does **not** invent one.

### 2.4 Tab: MCP

- **Matrix mapping:** "MCP (Global)" / "MCP (Local)".
- **Backed by:** existing `services::mcp::McpService` and
  `commands/mcp.rs`. **No new backend code in the IA PR.** The
  per-agent `McpService::sync_server_to_app_no_config` dispatch in
  `services/mcp.rs:111–151` and the
  `McpService::import_from_all_apps` table at lines 518–548 become
  registry-driven loops (iterate `agent_matrix()` instead of the
  hard-coded `[(&str, …); 6]` array), but the public function
  signatures do not change.
- **Reuses:** `src/components/mcp/UnifiedMcpPanel.tsx` unchanged.
- **Retires:** the legacy per-app `get_mcp_config` /
  `upsert_mcp_server_in_config` / `delete_mcp_server_in_config` /
  `set_mcp_enabled` commands in `commands/mcp.rs:55–161` and their
  frontend counterparts in `src/lib/api/mcp.ts` (the removal plan
  §C.2 already lists the legacy forms for deletion; the IA PR finishes
  the job by deleting the legacy Tauri commands in the same window).

### 2.5 Tab: Prompts

- **Matrix mapping:** "Prompts" is **not** in the matrix vocabulary
  (AGENTS.md §2). It is an existing CCS concept (the prompt library
  per agent) that survives the refactor. The IA design treats it as
  an **internal CCS tab**, not a matrix column, and notes this
  explicitly in `AGENTS.md` as a follow-up.
- **Backed by:** existing `services/prompt.rs` (lookup
  `src-tauri/src/services/prompt.rs`) and `commands/prompt.rs`. The
  per-agent split in `commands/prompt.rs` (Pi gets its own
  `get_pi_prompt_file` / `replace_pi_prompt_file` /
  `list_pi_prompt_templates` / etc.) is the **only** piece of
  per-agent branching that survives the IA, and it is preserved
  untouched.
- **Reuses:** `src/components/prompts/PromptPanel.tsx` unchanged.
- **Scope support:** Global only (the prompt library is user-wide).
  The Pi prompt _files_ (the on-disk `AGENTS.md` snapshot) are
  surfaced through the **Context Files** tab, not here.

### 2.6 Tab: Providers (replaces ProviderList)

- **Matrix mapping:** "Providers" is also **not** in the matrix
  vocabulary; it is the per-agent list of upstream provider presets
  (Claude's "Anthropic Official", Codex's "OpenAI Official",
  OpenCode's "Moonshot" preset, etc.). Same treatment as Prompts:
  internal CCS tab, not a matrix column.
- **Backed by:** the existing `database/dao/providers.rs` (CRUD on
  `providers` table) and the `commands/provider.rs` commands that
  the removal plan keeps (`get_providers`, `add_provider`,
  `update_provider`, `delete_provider`, `switch_provider`,
  `import_default_config`, `read_live_provider_settings`, plus the
  per-agent `ensure_*_official_provider` and `get_*_config_status`
  shims). The proxy-routing commands
  (`switch_proxy_provider`, `set_proxy_takeover_for_app`,
  `get_proxy_*`, `start_proxy_server`, `stop_proxy_*`,
  `get_failover_queue`, `set_auto_failover_enabled`) are deleted
  with PR-C in the removal plan.
- **Reuses:** the existing per-agent provider cards in
  `src/components/providers/ProviderCard.tsx` (used inside
  `ProviderList.tsx`). After PR-C the per-agent form components in
  `src/components/providers/forms/` (the `Gemini*` / `Grok*` /
  `Claude*` / `Codex*` / `Universal*` / `ProviderForm.tsx` /
  `ProviderAdvancedConfig.tsx` etc.) are deleted; the
  `ProviderCard` is the only surviving piece.
- **UI:** the Providers tab renders the agent's list of configured
  providers as `ProviderCard`s. The "active" provider is highlighted
  the same way the current `ProviderList.tsx:446–455` does (per-agent
  via the existing `isCurrent` logic; that logic moves to a small
  helper in `src/hooks/useAgentCurrent.ts` so the IA does not keep
  the per-agent switch).

### 2.7 Tab: Hooks

- **Matrix mapping:** "Hooks (Global)" / "Hooks (Local)".
- **Backed by:** `Op::ListHooks` / `Op::ReadHook` / `Op::WriteHook`
  variants. The implementation reads the agent's hooks file
  (per matrix: `~/.qwen/settings.json` `hooks`, `~/.claude/settings.json`
  `hooks`, `~/.codex/hooks.json`, `~/.mastracode/hooks.json`,
  `~/.gemini/antigravity-cli/plugins/<plugin>/hooks.json`,
  `cordis.yml` `plugins` entries for dsh, …) and renders an editor.
- **New Rust:** `agent::hooks::{list, read, write, delete}` per agent,
  with the per-agent location table generated from
  `docs/agent-matrix.md` (anything marked `SPECULATIVE` is rendered
  read-only with a note). The host layer (§5 below) makes the
  read/write transparent to remote hosts.

### 2.8 Tab: Rules

- **Matrix mapping:** "Rules (Global)" / "Rules (Local)".
- **Scope decision:** the matrix cross-agent rollup
  (`docs/agent-matrix.md` Part C) shows **no agent** ships a
  first-class rules file distinct from context files. The IA
  documents this honestly: the Rules tab is **view-only** in v1 and
  reads whatever the agent's docs call "rules" (typically a
  documented file under the agent's config dir). When a real
  per-agent rules file appears in the matrix (follow-up research
  bead), the same `<ContextFileList>` primitive used by the Context
  Files tab can be reused for Rules — the IA is designed for that
  path.

### 2.9 Tab: Plugins

- **Matrix mapping:** "Plugins (Global)" / "Plugins (Local)".
- **Scope:** Global only in v1. Per-agent exceptions (OpenCode's
  per-project `plugin` array, Antigravity's `plugins/<plugin>/skills/`
  activation) are surfaced as "Also see in: <Context Files or
  Skills>" hints; the actual install / enable / disable lives in the
  Plugins tab.
- **Backed by:** `Op::ListPlugins` / `Op::InstallPlugin` /
  `Op::EnablePlugin` / `Op::DisablePlugin` / `Op::UninstallPlugin`.
  Per-agent plugin stores (matrix column "Plugins (Global)"):
  - Claude Code: `~/.claude/plugins/`
  - OpenCode: `opencode.json` `plugin` array
  - Qwen: `qwen-extension.json` extension packages
  - Mastra: `ModePack` / `OmPack` agent plugins
  - Antigravity: `~/.gemini/antigravity-cli/plugins/`
  - dsh: Cordis packages in `packages/*`
  - Nanobot: `nanobot plugins list` registry
- **New Rust:** `agent::plugins::` module with a per-agent
  thin shim. Anything `SPECULATIVE` in the matrix renders a
  "Listing not implemented" placeholder.

---

## 3. ONE reusable Rust command layer

### 3.1 The new module: `src-tauri/src/agent.rs`

The new file (companion to the descriptor in §1.2) exposes a single
`#[tauri::command]` plus its supporting enums:

```rust
// src-tauri/src/agent.rs  (new — ~80 lines)

use serde::{Deserialize, Serialize};
use serde_json::Value;
use crate::app_config::AppType;
use crate::AppError;

#[derive(Debug, Clone, Deserialize)]
#[serde(tag = "op", rename_all = "snake_case")]
pub enum AgentOp {
    /* Context Files */
    ListContextFiles { scope: Scope },
    ReadContextFile  { scope: Scope, name: String },
    WriteContextFile { scope: Scope, name: String, content: String },
    DeleteContextFile{ scope: Scope, name: String },

    /* Skills — thin pass-through to services::skill */
    ListSkills,
    InstallSkill     { skill_id: String, scope: Scope },
    UninstallSkill   { skill_id: String },
    ToggleSkillApp   { skill_id: String, app: String, enabled: bool },

    /* Version / Update */
    GetVersion,
    RunUpdate,

    /* MCP — pass-through to services::mcp */
    ListMcpServers,
    UpsertMcpServer  { id: String, name: String, server: Value, scopes: Vec<Scope> },
    DeleteMcpServer  { id: String },
    ToggleMcpApp     { id: String, app: String, enabled: bool },
    ImportMcpFromApps,

    /* Hooks */
    ListHooks        { scope: Scope },
    WriteHook        { scope: Scope, event: String, command: String },

    /* Plugins */
    ListPlugins,
    InstallPlugin    { source: String },
    EnablePlugin     { name: String },
    DisablePlugin    { name: String },
    UninstallPlugin  { name: String },

    /* Prompts (pass-through) */
    ListPrompts,
    UpsertPrompt     { id: String, name: String, content: String, enabled: bool },
    DeletePrompt     { id: String },
    EnablePrompt     { id: String },

    /* Providers (CRUD) */
    ListProviders,
    SwitchProvider   { id: String },
    AddProvider      { provider: Value },
    UpdateProvider   { id: String, provider: Value },
    DeleteProvider   { id: String },
}

#[derive(Debug, Clone, Copy, Deserialize)]
#[serde(rename_all = "lowercase")]
pub enum Scope { Global, Local }

impl Scope { pub fn to_agent_scope(self) -> agent::Scope { … } }

#[tauri::command]
pub async fn agent_command(
    app: AppType,
    op: AgentOp,
    state: tauri::State<'_, crate::AppState>,
) -> Result<Value, AppError> {
    agent::dispatch(app, op, &state).await
}
```

`agent::dispatch` is a **single `match` on `op`** that calls into:

| Op branch          | Forwards to                                                        |
| ------------------ | ------------------------------------------------------------------ |
| `ListContextFiles` | `agent::context_files::list(app, scope)`                           |
| `*ContextFile`     | `agent::context_files::read/write/delete(app, scope, name, …)`     |
| `ListSkills`       | `commands::skill::get_installed_skills` (existing)                 |
| `*Skill`           | `services::skill::SkillService::*` (existing)                      |
| `GetVersion`       | `agent::version::read_version(app)`                                |
| `RunUpdate`        | `agent::version::run_update(app)`                                  |
| `ListMcpServers`   | `services::mcp::McpService::get_all_servers` (existing)            |
| `*Mcp*`            | `services::mcp::McpService::*` (existing)                          |
| `*Hook*`           | `agent::hooks::{list, write}(app, scope, …)`                       |
| `*Plugin*`         | `agent::plugins::{list, install, enable, disable, uninstall}(app)` |
| `*Prompt`          | `services::prompt::*` (existing)                                   |
| `ListProviders`    | `commands::provider::get_providers` (existing, after PR-C)         |
| `*Provider`        | `commands::provider::*` (existing, after PR-C)                     |

The point: **zero new per-agent command files.** The dispatch layer
is one file. The per-agent knowledge lives in `agent::<feature>::*`
helpers (also keyed by `AppType`) and the descriptor in §1.2.

### 3.2 What is deleted from `src-tauri/src/commands/`

| File                          | Disposition (after removal plan PR-C)                    | Why                                       |
| ----------------------------- | -------------------------------------------------------- | ----------------------------------------- |
| `commands/openclaw.rs`        | **Delete**                                               | per-agent; folded into `agent::*`         |
| `commands/hermes.rs`          | **Delete**                                               | per-agent; folded into `agent::*`         |
| `commands/pi.rs`              | **Delete**                                               | per-agent; folded into `agent::*`         |
| `commands/copilot.rs`         | **Delete**                                               | Claude routing variant; deleted with PR-C |
| `commands/codex_oauth.rs`     | **Keep**, re-exposed as `agent_command(app=Codex, op=…)` | Codex OAuth is per-agent                  |
| `commands/xai_oauth.rs`       | **Delete** (already in PR-B)                             | grok removed                              |
| `commands/provider.rs`        | **Keep** (after PR-C)                                    | CRUD on providers stays                   |
| `commands/prompt.rs`          | **Keep**                                                 | per-agent Prompts tab                     |
| `commands/mcp.rs`             | **Keep**                                                 | existing unified MCP API                  |
| `commands/skill.rs`           | **Keep**                                                 | existing unified Skill API                |
| `commands/subscription.rs`    | **Keep**                                                 | OAuth quota                               |
| `commands/global_proxy.rs`    | **Delete** (PR-C)                                        | proxy removed                             |
| `commands/failover.rs`        | **Delete** (PR-C)                                        | proxy removed                             |
| `commands/proxy.rs`           | **Delete** (PR-C)                                        | proxy removed                             |
| `commands/session_manager.rs` | **Delete** (PR-D)                                        | session manager removed                   |
| **New:** `commands/agent.rs`  | **Add**                                                  | re-exports `agent_command`                |

> **Note on `commands/openclaw.rs`, `commands/hermes.rs`,
> `commands/pi.rs`:** these are short per-agent command files (the
> `commands::openclaw_agents_defaults` family, the
> `commands::hermes_memory` family, the `commands::pi_*` family).
> They are folded into `agent::dispatch` by reading their `pub fn`
> bodies and migrating them to `agent::<feature>::openclaw::…` etc.
> so the public function signature is `pub(crate) fn xxx(app, payload)`
> and the new command is the only `#[tauri::command]`.

### 3.3 `tauri::generate_handler!` changes (in `src-tauri/src/lib.rs`)

The `agent_command` is the **only** new entry. After PR-C and the
above deletions the handler list shrinks to roughly:

- `agent_command` (new, single entry point for everything per-agent)
- `commands::mcp::*` (kept; legacy compat commands in `commands/mcp.rs:55–161` are deleted in the same window)
- `commands::skill::*` (kept)
- `commands::prompt::*` (kept)
- `commands::provider::*` (kept, after PR-C)
- `commands::subscription::*` (kept)
- Generic app commands that are not per-agent: `commands::deeplink::*`,
  `commands::deeplink_import::*`, `commands::import_export::*`,
  `commands::profile::*`, `commands::env::*`, `commands::webdav_sync::*`,
  `commands::s3_sync::*`, `commands::backup::*` (or whatever the
  remaining file is called after the renaming), `commands::misc::*`,
  `commands::settings::*`, `commands::config::*`, `commands::plugin::*`
  (the Claude plugin status shim, not the Plugins tab; see §2.9).
- App-management commands: `restart_app`, `install_update_and_restart`,
  `check_app_update_available`, `check_for_updates`, `is_portable_mode`,
  `copy_text_to_clipboard`, `get_settings`, `save_settings`.

The total command count drops from ~225 to roughly 80–100. Every
per-agent operation is reached through `agent_command`.

### 3.4 Mapping table — current `*_config.rs` / `commands/*.rs` → new layer

| Current function                                        | New path                                                                        |
| ------------------------------------------------------- | ------------------------------------------------------------------------------- |
| `codex_config::get_codex_config_dir`                    | `agent::paths::config_dir(app=Codex)`                                           |
| `codex_config::read_codex_live_settings`                | `agent_command(op=ListProviders, …).live_settings`                              |
| `codex_config::update_codex_toml_field`                 | `agent_command(op=UpdateProvider, …)`                                           |
| `opencode_config::get_opencode_dir`                     | `agent::paths::config_dir(app=OpenCode)`                                        |
| `opencode_config::set_mcp_server`                       | `agent_command(op=UpsertMcpServer, app=OpenCode)`                               |
| `opencode_config::add_plugin`                           | `agent_command(op=InstallPlugin, app=OpenCode)`                                 |
| `openclaw_config::get_agents_defaults`                  | `agent_command(op=ListPlugins, app=OpenClaw)`                                   |
| `openclaw_config::get_env_config`                       | `agent_command(op=ListHooks, app=OpenClaw, …)`                                  |
| `openclaw_config::get_tools_config`                     | `agent_command(op=ListPlugins, app=OpenClaw)` (tools are OpenClaw's "plugins")  |
| `hermes_config::read_memory`                            | `agent_command(op=ReadContextFile, app=Hermes, scope=Global, name="MEMORY.md")` |
| `hermes_config::write_memory`                           | `agent_command(op=WriteContextFile, …)`                                         |
| `hermes_config::get_mcp_servers_yaml`                   | `agent_command(op=ListMcpServers, app=Hermes)`                                  |
| `hermes_config::update_mcp_servers_yaml`                | `agent_command(op=UpsertMcpServer, app=Hermes)`                                 |
| `pi_config::get_pi_agent_dir`                           | `agent::paths::config_dir(app=Pi)`                                              |
| `pi_config::read_pi_native_providers`                   | `agent_command(op=ListProviders, app=Pi)`                                       |
| `pi_config::insert_pi_provider` / `replace_pi_provider` | `agent_command(op=AddProvider / UpdateProvider, app=Pi)`                        |
| `mcp::sync_single_server_to_<agent>`                    | `agent_command(op=ToggleMcpApp, app=<agent>, …)`                                |
| `mcp::remove_server_from_<agent>`                       | `agent_command(op=DeleteMcpServer, app=<agent>, …)`                             |
| `mcp::<agent>::import_from_<agent>`                     | `agent_command(op=ImportMcpFromApps, app=<agent>)`                              |
| `services::skill::SkillService::install`                | `agent_command(op=InstallSkill, app=<agent>)`                                   |
| `services::skill::SkillService::toggle_app`             | `agent_command(op=ToggleSkillApp, app=<agent>)`                                 |
| `services::skill::SkillService::sync_to_app`            | internal — no public command                                                    |
| `services::mcp::McpService::import_from_<agent>`        | `agent_command(op=ImportMcpFromApps, app=<agent>)`                              |
| `commands::mcp::*`                                      | `agent_command(op=<Mcp variant>, app=<target>)`                                 |
| `commands::skill::*` (legacy)                           | **Delete** (covered by unified + agent_command)                                 |
| `commands::prompt::*`                                   | `agent_command(op=<Prompt variant>, app=<target>)`                              |
| `commands::provider::*` (CRUD only, post PR-C)          | `agent_command(op=<Provider variant>, app=<target>)`                            |
| `commands::provider::*` (proxy-routing)                 | **Delete** (PR-C)                                                               |
| `commands::failover::*`                                 | **Delete** (PR-C)                                                               |
| `commands::proxy::*`                                    | **Delete** (PR-C)                                                               |
| `commands::session_manager::*`                          | **Delete** (PR-D)                                                               |
| `commands::openclaw::*`                                 | folded into `agent::dispatch`                                                   |
| `commands::hermes::*`                                   | folded into `agent::dispatch`                                                   |
| `commands::pi::*`                                       | folded into `agent::dispatch`                                                   |
| `commands::xai_oauth::*`                                | **Delete** (PR-B)                                                               |
| `commands::codex_oauth::*`                              | folded into `agent_command(app=Codex, op=GetVersion                             | ListPrompts | …)` |

---

## 4. ONE reusable frontend tab component

### 4.1 The new component: `<AgentTab>`

```tsx
// src/components/agents/AgentTab.tsx  (new, ~150 lines)

import { AGENT_REGISTRY, type TabKey } from "@/config/agentRegistry";
import { useSelectedAgent } from "@/state/agentSelection";

interface AgentTabProps {
  tabKey: TabKey;
}

export function AgentTab({ tabKey }: AgentTabProps) {
  const selected = useSelectedAgent();
  const descriptor = AGENT_REGISTRY[selected];

  if (!descriptor.tabs[tabKey]?.enabled) {
    return <AgentTabUnsupported tabKey={tabKey} agentId={selected} />;
  }

  switch (tabKey) {
    case "context-files":
      return <ContextFilesTab agent={descriptor} />;
    case "skills":
      return <UnifiedSkillsPanel /* currentApp={descriptor.id} */ />;
    case "version":
      return <VersionTab agent={descriptor} />;
    case "hooks":
      return <HooksTab agent={descriptor} />;
    case "rules":
      return <RulesTab agent={descriptor} />;
    case "mcp":
      return <UnifiedMcpPanel />;
    case "plugins":
      return <PluginsTab agent={descriptor} />;
    case "prompts":
      return <PromptPanel appId={descriptor.id} />;
    case "providers":
      return <ProvidersTab agent={descriptor} />;
  }
}
```

Each branch is a **thin shell** that:

1. Reads the agent descriptor (label, icon, paths, capability flags).
2. Calls `invoke("agent_command", { app, op, payload })` for any new
   operation.
3. Renders the existing shared panel (Skills, MCP, Prompts) **as-is**
   or a new tab body for the tabs that are new (Version, Hooks, Rules,
   Plugins, Context Files for non-OpenClaw agents, Providers).

### 4.2 Mapping table — current components → new layer

| Current component                                     | New role                                                                       | Net change                                                   |
| ----------------------------------------------------- | ------------------------------------------------------------------------------ | ------------------------------------------------------------ |
| `AppSwitcher.tsx`                                     | Top selector (visual)                                                          | Rewired to `AGENT_REGISTRY`; icon map replaced               |
| `Providers/ProviderList.tsx`                          | **Retired** — replaced by `ProvidersTab`                                       | Delete with PR-C sub-3                                       |
| `Providers/ProviderCard.tsx`                          | Used inside `ProvidersTab`                                                     | Unchanged (per-agent logic moves to hook)                    |
| `Providers/ProviderActions.tsx`                       | Used inside `ProvidersTab`                                                     | Unchanged                                                    |
| `Providers/ProviderHealthBadge.tsx`                   | Used inside `ProvidersTab`                                                     | Unchanged                                                    |
| `Providers/ProviderStatusBadge.tsx`                   | Used inside `ProvidersTab`                                                     | Unchanged                                                    |
| `Providers/ProviderEmptyState.tsx`                    | Used inside `ProvidersTab`                                                     | Unchanged                                                    |
| `Providers/AuthSettingsPanel.tsx`                     | **Retired** — OAuth lives in the Providers tab (per-agent sub-panel)           | Delete with PR-C                                             |
| `Providers/AddProviderDialog.tsx`                     | **Retired** — the per-agent `<AddProviderButton>` lives in `ProvidersTab`      | Delete with PR-C                                             |
| `Providers/EditProviderDialog.tsx`                    | **Retired** — replaced by `ProvidersTab`'s edit drawer                         | Delete with PR-C                                             |
| `Providers/forms/ProviderForm.tsx`                    | **Retired**                                                                    | Delete with PR-C                                             |
| `Providers/forms/ProviderAdvancedConfig.tsx`          | **Retired**                                                                    | Delete with PR-C                                             |
| `Providers/forms/ProviderPresetSelector.tsx`          | **Retired**                                                                    | Delete with PR-C                                             |
| `Providers/forms/StructuredOptionsEditor.tsx`         | **Retired**                                                                    | Delete with PR-C                                             |
| `Providers/forms/RequestHeadersEditor.tsx`            | **Retired**                                                                    | Delete with PR-C                                             |
| `Providers/forms/LocalProxyRequestOverridesField.tsx` | **Retired**                                                                    | Delete with PR-C                                             |
| `Providers/forms/CopilotAuthSection.tsx`              | **Retired** (after the AuthCenter migration)                                   | Migrated or deleted with PR-C; see removal §F                |
| `Providers/forms/ClaudeFormFields.tsx`                | **Retired**                                                                    | Delete with PR-C                                             |
| `Providers/forms/CodexFormFields.tsx`                 | **Retired**                                                                    | Delete with PR-C                                             |
| `Providers/forms/CodexCommonConfigModal.tsx`          | **Retired**                                                                    | Delete with PR-C                                             |
| `Providers/forms/CodexConfig*.tsx`                    | **Retired**                                                                    | Delete with PR-C                                             |
| `Providers/forms/CodexOAuthSection.tsx`               | **Retired** — replaced by `agent_command` OAuth sub-op                         | Delete with PR-C                                             |
| `Providers/forms/ApiKeyInput.tsx`                     | **Retired** — input primitive stays in `ProvidersTab`                          | Delete; primitive lifted into `ProvidersTab` body            |
| `Providers/forms/BasicFormFields.tsx`                 | **Retired**                                                                    | Delete with PR-C                                             |
| `Providers/forms/EndpointSpeedTest.tsx`               | **Retired**                                                                    | Delete with PR-C                                             |
| `Providers/forms/CustomUserAgentField.tsx`            | **Retired**                                                                    | Delete with PR-C                                             |
| `Providers/forms/CommonConfigEditor.tsx`              | **Retired**                                                                    | Delete with PR-C                                             |
| `Providers/forms/ProviderAdvancedConfig.tsx`          | **Retired**                                                                    | Delete with PR-C                                             |
| `Providers/forms/Gemini*`                             | **Retired** (already deleted in PR-A)                                          | —                                                            |
| `Providers/forms/Grok*`                               | **Retired** (already deleted in PR-B)                                          | —                                                            |
| `Mcp/UnifiedMcpPanel.tsx`                             | Reused as-is for the MCP tab                                                   | No change to file                                            |
| `Skills/UnifiedSkillsPanel.tsx`                       | Reused as-is for the Skills tab                                                | No change to file                                            |
| `Skills/SkillsPage.tsx`                               | **Retired** — unified panel covers it                                          | Delete with PR-C sub-3 (removal §C.2)                        |
| `Prompts/PromptPanel.tsx`                             | Reused as-is for the Prompts tab                                               | No change to file                                            |
| `Prompts/PromptFormPanel.tsx`                         | Reused inside `PromptPanel`                                                    | No change                                                    |
| `Prompts/PromptListItem.tsx`                          | Reused inside `PromptPanel`                                                    | No change                                                    |
| `Prompts/PromptToggle.tsx`                            | Reused inside `PromptPanel`                                                    | No change                                                    |
| `Prompts/PiNativePromptResources.tsx`                 | Reused inside `PromptPanel`                                                    | No change                                                    |
| `Prompts/PiPromptPanel.tsx`                           | Reused inside `PromptPanel`                                                    | No change                                                    |
| `Workspace/WorkspaceFilesPanel.tsx`                   | **Retired** — generalized to `ContextFilesTab`                                 | Delete (replaced by generic tab)                             |
| `Workspace/WorkspaceFileEditor.tsx`                   | Reused inside `ContextFilesTab`                                                | No change to file                                            |
| `Workspace/DailyMemoryPanel.tsx`                      | Reused inside `ContextFilesTab` (when the matrix marks "Daily Memory" present) | No change                                                    |
| `Settings/AuthCenterPanel.tsx`                        | **Retired** — per-agent OAuth sections move into the Providers tab             | Delete with PR-C; per-agent OAuth lives in the Providers tab |
| `Settings/SettingsPage.tsx` (OAuth-related sections)  | **Retired** — same reason                                                      | Delete                                                       |
| `Agents/AgentsPanel.tsx` (placeholder stub)           | **Replaced** by `<AgentShell>` (tab bar + `<AgentTab>`)                        | Rewrite in IA PR                                             |
| `common/AppToggleGroup.tsx`                           | Reused in every tab's per-agent enable/disable strip                           | No change                                                    |
| `common/AppCountBar.tsx`                              | Reused in MCP and Skills tabs (and in `ProvidersTab`)                          | No change                                                    |

> **Net effect of the frontend consolidation:** of the ~80 files
> under `src/components/providers/`, `src/components/workspace/`,
> and `src/components/agents/` today, the IA keeps **8** unchanged
> (`UnifiedMcpPanel`, `UnifiedSkillsPanel`, `PromptPanel`,
> `ProviderCard`, `ProviderActions`, `ProviderHealthBadge`,
> `ProviderStatusBadge`, `ProviderEmptyState`), retires **40+** in
> PR-C and PR-D, and adds **6** new ones (`AgentShell`,
> `AgentTab`, `ContextFilesTab`, `VersionTab`, `HooksTab`,
> `RulesTab`, `PluginsTab`, `ProvidersTab`).

### 4.3 The new component: `<AgentShell>`

```tsx
// src/components/agents/AgentShell.tsx  (new, ~80 lines)

export function AgentShell() {
  const selected = useSelectedAgent();
  const descriptor = AGENT_REGISTRY[selected];
  const enabledTabs = (
    Object.entries(descriptor.tabs) as [TabKey, TabSupport][]
  )
    .filter(([, t]) => t.enabled)
    .map(([k]) => k);

  return (
    <div className="flex flex-col h-full">
      <AgentSelector /> {/* rewired AppSwitcher */}
      <TabBar tabs={enabledTabs} /> {/* one row, one per enabled tab */}
      <div className="flex-1 min-h-0">
        <AgentTab tabKey={currentTabKey()} />
      </div>
    </div>
  );
}
```

The `AgentShell` lives at the top of the new view; the
`currentTabKey` is local state, **not** persisted (see §1.3).

### 4.4 State hooks

- `useSelectedAgent(): AppId` — reads `selectedAgent` from a small
  context (or Zustand slice — match the existing pattern in
  `src/state/`).
- `useAgentDescriptor(): AgentDescriptor` — convenience hook over
  `AGENT_REGISTRY[useSelectedAgent()]`.
- `useAgentCommand<T>(op: AgentOp): { data, loading, error, run }` —
  wraps `invoke("agent_command", { app, op })` with React Query (or
  whatever the existing data-fetching primitive is; the codebase
  uses React Query in MCP and Skills hooks — follow the same
  pattern).

The existing `useMcp`, `useSkills`, `usePromptActions`,
`useOpenClaw`, `useHermes` hooks are **retained** as the
implementation of `useAgentCommand` for the MCP, Skills, Prompts,
OpenClaw, and Hermes op families. The IA does not unify them at the
hook level; it only unifies at the command level. This keeps the
diff focused.

---

## 5. Host awareness

The IA design is **host-agnostic** by default. The `agent_command`
takes an `app: AppType` and an `op: AgentOp`; the actual file I/O
is delegated to `agent::<feature>::<fn>(app, op, host)` where
`host: &HostAdapter` is provided by the Step 5 design. The
matrix cells that are marked `SPECULATIVE` because of a remote-host
file path become the responsibility of the host adapter, not the
per-agent backend. This doc does not redefine the `HostAdapter`
interface — see `docs/host-switch-design.md` for that contract.
The only IA-side commitment is that **every op takes the host
through `&HostAdapter`** so the IA tabs work transparently against
local, SSH, and Tailscale-SSH hosts without code changes.

---

## 6. Migration path

The migration is sequenced so the IA lands on a clean tree. The
removal plan (Step 3) is the destructive half; the IA design is
the constructive half. The exact PR ordering is:

1. **PR-A** (removal): delete Gemini. `AGENT_REGISTRY` in v1 does
   **not** include `gemini`.
2. **PR-B** (removal): delete GrokBuild. `AGENT_REGISTRY` v1 still
   does not include `grokbuild`.
3. **PR-C** (removal): delete proxy/failover/presets subsystem. By
   the end of PR-C, `AppSwitcher` and `ProviderList` are unused by
   the new view but still exist in the tree (read-only). `ProxyAppId`
   / `PROXY_APP_IDS` / `isProxyAppId` are deleted; `AppType` still
   has all 7 remaining variants.
4. **PR-D** (removal): delete session manager. The session-manager
   view is gone.
5. **PR-IA-1 (this design)**: introduce
   - `src/config/agentRegistry.ts` (hand-written from
     `docs/agent-matrix.md`)
   - `src-tauri/src/agent.rs` + `src-tauri/src/agent/` module
   - `src/components/agents/AgentShell.tsx` + `AgentTab.tsx` +
     the per-tab bodies for the new tabs (Context Files, Version,
     Hooks, Rules, Plugins, Providers)
   - One new `agent_command` Tauri command
   - One new `useSelectedAgent` / `useAgentDescriptor` /
     `useAgentCommand` hook trio
   - Reuses `UnifiedMcpPanel`, `UnifiedSkillsPanel`, `PromptPanel`
     unchanged.
6. **PR-IA-2**: fold the per-agent command files
   (`commands/openclaw.rs`, `commands/hermes.rs`, `commands/pi.rs`,
   `commands/codex_oauth.rs`) into `agent::dispatch`. Delete
   `src-tauri/src/commands/{openclaw,hermes,pi,codex_oauth}.rs`.
   Update `commands/mod.rs`. Delete the legacy
   `commands/mcp.rs:55–161` compat commands. Delete the
   `commands/skill.rs:185–335` legacy install/uninstall commands.
7. **PR-IA-3**: delete `src/components/AppSwitcher.tsx` (its
   rewire is the only consumer), `src/components/providers/*` and
   `src/components/workspace/WorkspaceFilesPanel.tsx`,
   `src/components/agents/AgentsPanel.tsx` (the placeholder).
8. **PR-IA-4**: at this point `AppSwitcher` and `ProviderList` are
   gone; `App.tsx` no longer imports them. The new `AgentShell` is
   the only top-level view. Delete `src/hooks/useDragSort.ts`
   (the provider-routing drag sort).

At the end of PR-IA-4, the IA design is fully landed. The convoy
owner reviews in a single batch.

### 6.1 Backwards compatibility for the `ProviderList` data

Until PR-IA-2, the `commands::provider::*` commands still expose the
CRUD that `ProviderList.tsx` calls. `ProviderCard` continues to
work. The `addProvider` / `updateProvider` / `deleteProvider` /
`switchProvider` payload is unchanged. The IA PRs only **add**
the new `agent_command` and the per-op envelopes; the old commands
are deleted **after** every call site has been migrated to the
new path.

### 6.2 The `AppType` enum survives unchanged

The Rust `AppType` keeps all 7 remaining variants through the
entire IA. The IA only **adds** `agent::AgentDescriptor` and
`agent::AgentOp`; it does not reshape `AppType`. This keeps the
DB schema (which references `AppType`) untouched.

### 6.3 Per-agent presets (Claude, Codex, …)

The per-agent preset files in `src/config/<agent>ProviderPresets.ts`
(Claude, Codex, Hermes, OpenClaw, OpenCode, Pi) are **kept** —
they are the seed data the `AddProvider` flow shows in the
Providers tab. The IA design does not invent a new provider
preset system; it inherits the existing one. `geminiProviderPresets`
and `grokBuildProviderPresets` are deleted by PR-A and PR-B
respectively.

### 6.4 The `universal` provider preset (after PR-A, before PR-C)

`src/config/universalProviderPresets.ts` is removed by PR-C sub-3
(removal plan §C.3). The "Universal providers" panel is removed by
PR-C sub-3; the `ClaudeDesktop` variant of `AppType` is **kept**
because the matrix still lists it (Part B — Claude Code Claude
Desktop entry), and it surfaces in the AGENT selector as a
second Claude entry.

---

## 7. Open questions and risks

### 7.1 Open questions

- **Q1 — Should the `Prompts` and `Providers` tabs be added to the
  matrix vocabulary?** The bead body says "Context Files, Skills,
  Version/Update, Hooks, Rules, MCP, Plugins" — `Prompts` and
  `Providers` are not in that list. This doc proposes them as
  internal CCS tabs (per §2.5 and §2.6). If the convoy owner
  prefers, the IA could ship without the **Prompts** tab and keep
  prompts as a global panel reachable from the AgentShell footer.
  Recommend keeping `Prompts` as a per-agent tab because the
  current `PromptPanel` is already agent-scoped via its `appId`
  prop.
- **Q2 — Should `selectedTab` be persisted per agent?** This doc
  says no for v1 (§1.3). If the convoy owner wants "last opened
  tab" per agent, the implementation is a one-line `localStorage`
  read/write in the `useSelectedAgent` context.
- **Q3 — How should the IA render `SPECULATIVE` matrix cells?** This
  doc says "show a `Coming Soon` placeholder" (§1.2
  `TabSupport.placeholder`). The alternative is to render
  the tab with a banner explaining the data is best-effort and
  asking the user to confirm. Recommend the placeholder for v1.
- **Q4 — Should `commands/codex_oauth.rs` and the `CodexOAuthSection`
  remain after the IA?** This doc folds `codex_oauth` into
  `agent_command` (the OAuth flow is reachable as
  `agent_command(app=Codex, op=…CodexOAuth…)`). The
  `CodexOAuthSection` UI is replaced by the Providers tab's
  per-agent OAuth sub-panel. Confirm with the convoy owner.
- **Q5 — How should the per-agent MCP import on first run
  (`McpService::import_from_all_apps`) be exposed in the IA?** This
  doc exposes it as `agent_command(op=ImportMcpFromApps)`. The
  current "Import from apps" button in `UnifiedMcpPanel` is the
  only consumer; the IA keeps that button in the MCP tab.

### 7.2 Risks

- **R1 — Frontend hook proliferation.** `useMcp`, `useSkills`,
  `usePromptActions`, `useOpenClaw`, `useHermes` are kept. The new
  `useAgentCommand` is a thin wrapper around `invoke("agent_command")`
  and does not replace them. The risk is that future contributors
  add per-agent hooks instead of routing through `useAgentCommand`.
  Mitigation: add an ESLint rule (or a comment in the
  `src/hooks/index.ts`) that points contributors to
  `useAgentCommand` first.
- **R2 — `agent::dispatch` becoming a god function.** The match
  has 25+ arms in v1. Mitigation: keep the inner helpers
  (`agent::context_files::*`, `agent::hooks::*`, etc.) as
  submodules; the dispatch arm is a one-line call into the right
  submodule. The dispatch is a router, not an implementation.
- **R3 — Matrix drift.** The hand-written `AGENT_REGISTRY` and
  `agent::agent_matrix()` can drift from `docs/agent-matrix.md`
  over time. Mitigation: add a follow-up bead that generates the
  TypeScript and Rust descriptors from the markdown table at build
  time (out of scope for this bead).
- **R4 — `SPECULATIVE` cells in the matrix become silent
  failures in the UI.** If a tab is marked `enabled: true` in the
  registry but the per-agent backend is a placeholder, the user
  clicks a tab and sees an empty box. Mitigation: the
  `TabSupport.placeholder` flag (§1.2) renders a "Coming Soon"
  card with a link to `docs/agent-matrix.md` row for the agent.
- **R5 — `ClaudeDesktop` as an `AppType` variant.** The matrix
  treats it as a routing variant of Claude. The IA renders it
  as a second Claude entry. The risk is that the user sees two
  "Claude" icons in the selector and is confused. Mitigation:
  `AgentDescriptor.label = "Claude Desktop"` and a small
  `Claude Desktop` badge (the existing `APP_BADGE_ICON` in
  `AppSwitcher.tsx:15`).
- **R6 — The `CodexOAuth` per-agent OAuth flow is widely used.**
  Removing `commands::codex_oauth` and folding it into
  `agent_command` is a behavior-preserving change, but the
  internal `CodexOAuthState` managed in `lib.rs:1162` (XaiOAuth's
  sibling) is currently a separate `tauri::State`. The IA PR
  needs to confirm that the new `agent_command` path doesn't
  require a separate `tauri::State` for Codex OAuth (it shouldn't
  — the `app: AppType` argument is the dispatch key).
- **R7 — Backward compatibility of the
  `tauri::State<AppState>` argument.** The new `agent_command`
  takes `tauri::State<'_, AppState>` the same way the existing
  commands do. No change to the app state schema. No migration
  needed for the data layer.

---

## 8. Summary

- **One AGENT selector** backed by a single
  `AGENT_REGISTRY` (frontend) / `agent::agent_matrix()` (backend).
- **Nine tabs** per agent, each rendered by the same `<AgentTab>`
  component: Context Files, Skills, Version/Update, Hooks, Rules,
  MCP, Plugins, Prompts, Providers.
- **One reusable Rust command** (`agent_command`) with 25+ `AgentOp`
  variants, dispatching to the existing per-agent backend modules
  (`<agent>_config.rs`, `mcp/<agent>.rs`,
  `services::skill::SkillService`, `services::mcp::McpService`,
  `services::prompt`) through thin per-feature shims.
- **One reusable frontend tab component** (`<AgentTab>`) that
  reuses `UnifiedMcpPanel`, `UnifiedSkillsPanel`, `PromptPanel`
  unchanged and adds new tab bodies for Context Files, Version,
  Hooks, Rules, Plugins, Providers.
- **Migrates** in four IA PRs after PR-A/B/C/D of the removal
  plan. No app data migration needed; the `AppType` enum and the
  `providers` DB table are untouched.
- **Open questions** are listed for convoy-owner review (§7.1);
  **risks** are listed with mitigations (§7.2).

The vocabulary in this doc is aligned with `AGENTS.md` §2 and
`docs/agent-matrix.md`. The two extra tabs (`Prompts` and
`Providers`) are explicitly flagged as **internal CCS tabs**, not
matrix columns, so the public vocabulary remains unchanged.

---

## References

- [AGENTS.md](../AGENTS.md) — vocabulary and hand-off contract.
- [Agent Capability Matrix](./agent-matrix.md) — the capability
  matrix that drives the `AGENT_REGISTRY` and `agent::agent_matrix()`.
- [Removal Plan](./removal-plan.md) — the destructive half
  (PR-A/B/C/D) that this IA design replaces.
- [Host Switch Design](./host-switch-design.md) — supplies the
  `HostAdapter` this doc references.
- Authoritative Rust modules cited inline:
  `src-tauri/src/app_config.rs:378–395` (AppType),
  `src-tauri/src/services/mcp.rs:111–151, 518–548` (MCP dispatch),
  `src-tauri/src/services/skill.rs:569–629, 2241–2309` (Skill dispatch),
  `src-tauri/src/commands/{mcp,skill,prompt,provider}.rs`,
  `src-tauri/src/mcp/mod.rs:1–40` (per-agent MCP module list).
- Authoritative frontend modules cited inline:
  `src/config/appConfig.tsx:19–200` (current scattered constants),
  `src/components/AppSwitcher.tsx:1–226` (the selector to be rewired),
  `src/components/providers/ProviderList.tsx:1–740` (the panel to be
  retired), `src/components/{mcp,skills,prompts,workspace}/*`
  (the panels to be reused), `src/components/agents/AgentsPanel.tsx`
  (the placeholder to be replaced).
