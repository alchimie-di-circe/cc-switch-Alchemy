# AGENTS.md — Repository Conventions for Alchemy Refactor

This file is the **root alignment document** for the multi-step refactor that replaces cc-switch's provider-switching model with an **agent-first** information architecture. Polecats producing planning artifacts and code MUST follow these conventions so that the work converges.

## 1. Scope of the refactor

The staged convoy `cc-switch-Alchemy Refactor Plan (Steps 2-5)` targets four sequential work products:

1. **Step 2 — Agent Capability Matrix** (`docs/agent-matrix.md`): a single source of truth documenting each external agent (CLI binary, global/project config locations, native support for Context Files, Skills, Version/Update, Hooks, Rules, MCP, Plugins). This file is the **input contract** for every later step.
2. **Step 3 — Removal Plan** (`docs/removal-plan.md`): the destructive plan that retires Gemini, Grok Build, the provider-routing/presets subsystem, and the session-manager subsystem without breaking the build.
3. **Step 4 — IA Design** (`docs/ia-design.md`): the new AGENT selector, the shared per-agent tabs, the single reusable Rust command layer, and the single reusable frontend tab component.
4. **Step 5 — Host Switch Design** (`docs/host-switch-design.md`): a Settings-level host switcher (local / ssh / tailscale-ssh) implemented behind a `HostAdapter` so the agent tabs work transparently against remote VPS hosts.

All planning documents MUST be consistent with the vocabulary below and reference `docs/agent-matrix.md` as the authoritative column definition.

## 2. Authoritative vocabulary (column + tab names)

These exact terms are the **public contract** for capability matrices, IA labels, and Tauri command argument names. Do not invent synonyms.

| Term | Meaning in this refactor |
| --- | --- |
| **Context Files** | Persistent instruction files loaded into the model prompt (e.g. `AGENTS.md`, `QWEN.md`, `.goosehints`, `CLAUDE.md`). |
| **Skills** | Folder-based, model-loadable instruction packs typically anchored by a `SKILL.md` manifest. |
| **Version/Update** | The mechanism the agent uses to check its own version and self-update (CLI flag, release channel, package manager). |
| **Hooks** | External commands the agent invokes at well-defined lifecycle events (e.g. `PreToolUse`, `PostToolUse`, `SessionStart`). |
| **Rules** | Per-agent declarative rules files, distinct from context files (typically checked into the project, enforced by the agent's lint/validation step). |
| **MCP** | The Model Context Protocol. Native support means the agent reads `mcpServers` / equivalent config and connects to external MCP servers. |
| **Plugins** | Bundled extension packages that ship skills, MCP servers, hooks, and/or subagents as a single installable artifact. |

Every column in `docs/agent-matrix.md` and every tab label in `docs/ia-design.md` MUST use one of these exact labels. **Global/Local** scope is always noted explicitly per agent per tab.

## 3. Research and citation rules

Planning beads require authoritative sources, not inference.

- **Authoritative** = official project README, official docs site, source code in the agent's own GitHub repo, or a release note from the maintainer.
- **SPECULATIVE** = anything you cannot confirm from one of the above. Mark the cell with the literal string `SPECULATIVE` and state the assumption in a note. **Do not invent config schemas.**
- For the already-supported agents (Claude Code, Codex, OpenCode, OpenClaw, Hermes, Pi) the authoritative source is the existing `src-tauri/src/*_config.rs` module in this repository.

## 4. Repository conventions

- **Docs root**: every planning document lives under `docs/` and uses kebab-case filenames (e.g. `agent-matrix.md`, `removal-plan.md`).
- **No application code changes** in the planning beads. Code refactors only land after their plan bead closes.
- **Commits**: small, focused, signed-off. Use Conventional Commits style for the subject line.
- **Branches**: stay on the worktree branch assigned to your bead. Do not switch branches.
- **Cross-references**: planning docs must reference each other by relative path (e.g. `See [Removal Plan](./removal-plan.md).`).
- **Naming for new agent modules**: when an agent backend is added, the Rust module MUST be `<agent_id>_config.rs` in `src-tauri/src/`, and the path getter MUST be `get_<agent_id>_dir()` returning the global config directory. Frontend agent ids use kebab-case (e.g. `qwen-code`, `goose`, `deepseek-harness`).
- **MCP module**: per-agent MCP layers go under `src-tauri/src/mcp/<agent_id>.rs` and re-export through `src-tauri/src/mcp/mod.rs`.
- **No backslashes** in doc paths; always use POSIX-style forward slashes.

## 5. Hand-off contract

A planning bead is **only done** when:

1. Its doc is committed to `docs/` on the bead's branch.
2. The doc cross-references the upstream docs it depends on (e.g. the IA design must reference the matrix and the removal plan).
3. The branch is pushed.
4. A PR is opened against `convoy/cc-switch-alchemy-refactor-plan-steps-2-/12603eab/head` and `gt_done` is called with the PR URL.

If you discover that an earlier bead is wrong or incomplete, **do not silently fix it**. Open an escalation bead or coordinate via `gt_mail_send` so the convoy owner can re-dispatch.
