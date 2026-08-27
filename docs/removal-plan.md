# Removal Plan — Step 3 of the Alchemy Refactor

> **Status:** planning only. Do **not** start any code work in this PR. The mayor
> must explicitly start the staged convoy before any of the deletion steps below
> are executed.
>
> **Scope:** remove (A) the Gemini CLI agent, (B) the Grok Build agent, (C) the
> provider-routing / presets / proxy subsystem, and (D) the session-manager
> subsystem. Replace with the AGENT-selector IA designed in Step 4.
>
> **Codebase ground truth:** all file paths and line numbers below are taken
> from the current working tree (`convoy/cc-switch-alchemy-refactor-plan-steps-2-/12603eab/head`).
> Anything `// SPECULATIVE` has not been verified line-by-line during planning and
> must be re-confirmed when the implementation PR opens.

---

## 0. Strategy and shared utilities to preserve

### 0.1 Strategy

Targets A and B (single-agent removals) are independent of C (proxy/presets) and
D (session manager). Target C is the largest and most invasive change. Target D
touches the DB schema and therefore needs its own migration.

Recommended PR slicing (each PR keeps `cargo build` + `pnpm typecheck` + `pnpm test:unit` green):

1. **PR-A** — remove Gemini agent.
2. **PR-B** — remove Grok Build agent.
3. **PR-C** — remove provider-routing / proxy / failover / presets.
4. **PR-D** — remove session-manager UI + backend (keeps `session_usage*` only
   for kept agents codex/opencode/pi, plus the `proxy_request_logs.session_id`
   column used by the kept Usage dashboard).

PR-A and PR-B are reversible in isolation. PR-C and PR-D depend on PR-A+B having
landed (because they unregister Gemini and GrokBuild from the `pub use mcp::*`
table, the failover gate, and `PROXY_STARTUP_APP_TYPES`).

### 0.2 Shared utilities that must be preserved

These are referenced by kept features and must NOT be deleted even though they
appear in files we are partially removing:

- `src/utils/grokBuildConfig.ts` and its test — used by `grokBuildProviderPresets.test.ts`
  for shared Codex URL/model extraction helpers (`extractCodexBaseUrl`,
  `extractCodexModelName`). After PR-B the test goes away; **audit whether the
  `utils/grokBuildConfig.ts` file is still imported elsewhere before deleting it**
  (it is currently grokBuild-only, so it is safe to delete with PR-B, but flag
  it in the PR description).
- `src/components/providers/forms/CopilotAuthSection.tsx` and
  `useCopilotAuth.ts` — Copilot is a kept Claude variant for routing, not
  targeted for removal.
- `src/components/providers/forms/CodexOAuthSection.tsx`,
  `src/components/providers/forms/CodexFormFields.tsx`,
  `src/components/providers/forms/CodexConfig*.tsx`,
  `src/components/providers/forms/CodexCommonConfigModal.tsx` — all kept
  (Codex agent stays).
- `src/components/providers/forms/ClaudeFormFields.tsx`,
  `ClaudeDesktopProviderForm.tsx`, `BasicFormFields.tsx`,
  `ApiKeyInput.tsx`, `EndpointSpeedTest.tsx`, `LocalProxyRequestOverridesField.tsx`,
  `ProviderAdvancedConfig.tsx`, `ProviderPresetSelector.tsx`,
  `RequestHeadersEditor.tsx`, `StructuredOptionsEditor.tsx`,
  `CustomUserAgentField.tsx`, `CommonConfigEditor.tsx` — these are used by both
  the removed and the kept agents. **Do not delete them in PR-A or PR-B; they
  will be retired in PR-C** when the multi-provider provider-routing UI is
  replaced by the data-driven IA from Step 4.
- `src/components/providers/ProviderCard.tsx`,
  `ProviderList.tsx`, `AddProviderDialog.tsx`, `EditProviderDialog.tsx`,
  `ProviderActions.tsx`, `ProviderHealthBadge.tsx`,
  `ProviderStatusBadge.tsx`, `FailoverPriorityBadge.tsx`,
  `ProviderEmptyState.tsx`, `AuthSettingsPanel.tsx` — same: only retire in
  PR-C.
- `src/components/universal/UniversalProviderPanel.tsx`,
  `UniversalProviderCard.tsx`, `UniversalProviderFormModal.tsx` — universal
  providers are a Claude+Codex+Gemini cross-app feature. After PR-A, the
  universal panel must be reduced to Claude+Codex only; full removal in PR-C.
- `src/components/agents/AgentsPanel.tsx` — a 22-line stub. Keep untouched in
  PR-A and PR-B; this is the placeholder the Step-4 IA will fill in.
- `src/components/mcp/UnifiedMcpPanel.tsx` and
  `src/components/skills/UnifiedSkillsPanel.tsx` — keep working. They depend on
  `MCP_APP_IDS` / `SKILLS_APP_IDS` (frontend-only, no Rust dep) and on the
  per-agent backend MCP modules under `src-tauri/src/mcp/<agent>.rs`. After
  PR-A/B these two constants lose `gemini` and `grokbuild` and the backend
  re-exports in `src-tauri/src/mcp/mod.rs` lose the corresponding entries; the
  panels must be re-verified to still render for the kept agents.
- `src/components/prompts/PromptPanel.tsx`, `PromptFormPanel.tsx`,
  `PromptListItem.tsx`, `PromptToggle.tsx`, `PiNativePromptResources.tsx`,
  `PiPromptPanel.tsx` — independent of removed targets.
- `src/components/settings/AuthCenterPanel.tsx` — references
  `XaiOAuthSection` (PR-B) and `CodexOAuthSection` (kept) and
  `CopilotAuthSection` (kept). Edit to remove the XaiOAuthSection import in PR-B.
- `src/components/usage/*` (Usage dashboard) — depends on `proxy_request_logs`
  table; the table is kept (it powers the kept Usage dashboard). The
  `session_id` column on that table is harmless to drop _only_ if the Usage
  dashboard never references it. Audit before removal in PR-D.

### 0.3 Gating checks

Every PR must keep the following green:

- `cd src-tauri && cargo build`
- `cd src-tauri && cargo test`
- `pnpm install && pnpm typecheck`
- `pnpm test:unit`

If any test imports a removed module (most likely in `tests/components/` or
`tests/lib/`), delete or update the test in the same PR.

---

## A) Gemini CLI agent

### A.1 Files to delete (Rust)

```
src-tauri/src/gemini_config.rs
src-tauri/src/gemini_mcp.rs
src-tauri/src/mcp/gemini.rs
src-tauri/src/session_manager/providers/gemini.rs
src-tauri/src/services/session_usage_gemini.rs
```

(Removing `session_manager/providers/gemini.rs` and
`services/session_usage_gemini.rs` is fine here; both will be wholly retired in
PR-D. We delete them in PR-A to avoid leaving dangling dead code that the
provider-routing and session-manager tests still import indirectly.)

### A.2 Files to delete (Frontend)

```
src/config/geminiProviderPresets.ts
src/components/providers/forms/GeminiCommonConfigModal.tsx
src/components/providers/forms/GeminiConfigEditor.tsx
src/components/providers/forms/GeminiConfigSections.tsx
src/components/providers/forms/GeminiFormFields.tsx
src/components/providers/forms/hooks/useGeminiConfigState.ts
src/components/providers/forms/hooks/useGeminiCommonConfig.ts
src/icons/extracted/gemini.svg
```

### A.3 Files to edit (Rust)

- `src-tauri/src/lib.rs`
  - L15 `mod gemini_config;` — **delete**
  - L16 `mod gemini_mcp;` — **delete**
  - L53–59 `pub use mcp::{...}` — drop `import_from_gemini`,
    `remove_server_from_gemini`, `sync_enabled_to_gemini`,
    `sync_single_server_to_gemini`.
  - L710 comment `// Claude / Codex / Gemini` — reword to
    `// Claude / Codex`.
  - L945–951 the `McpService::import_from_gemini(...)` block + surrounding
    log lines (`"✓ Imported {count} MCP server(s) from Gemini"`,
    `"No Gemini MCP servers found"`, `"Failed to import Gemini MCP"`) —
    **delete** the whole `if app == AppType::Gemini` arm inside the import
    loop in the auto-import routine.
  - L985 `AppType::Gemini` arm in the prompts auto-import loop — **delete**.
  - L1228–1237 the `scrub_leaked_gemini_common_config` call + log + the
    function's own definition (in `services/provider/mod.rs:5984`) — delete
    the call here; the function definition is removed when its only caller
    disappears.
  - L1938 `const PROXY_STARTUP_APP_TYPES` — drop `"gemini"` (and `"grokbuild"`
    if PR-B has landed first; otherwise leave).
  - L2050 `AppType::Gemini` arm in `legacy_common_config` loop — **delete**.
  - L2304–2305 the gemini URL in a redaction test — change the test string
    to a generic URL or a codex URL; **do not delete the test**.

- `src-tauri/src/mcp/mod.rs`
  - L10 doc comment line `- gemini - Gemini MCP 同步和导入` — **delete**.
  - L16 `mod gemini;` — **delete**.
  - L30–33 `pub use gemini::{...}` — **delete**.

- `src-tauri/src/services/mod.rs`
  - L22 `pub mod session_usage_gemini;` — **delete**.

- `src-tauri/src/services/mcp.rs`
  - L124 `sync_single_server_to_gemini(...)` arm in the dispatch table — **delete**.
  - L173 `AppType::Gemini => mcp::remove_server_from_gemini(id)?` arm in the
    `remove_server_from_*` match — **delete**.
  - L373–380 `pub fn import_from_gemini(state)` and its body — **delete**.
  - L525 `("gemini", Self::import_from_gemini(state))` entry in the
    `import_all_agents` table — **delete**.

- `src-tauri/src/services/provider/mod.rs`
  - L5984 `pub async fn scrub_leaked_gemini_common_config(state)` — **delete**
    (after the lib.rs call site is gone).
  - L1137, L1178, L1202, L1260, L1296, L1333, L1377, L1410, L1443, L1454 —
    these are test references to `scrub_leaked_gemini_common_config`. Update
    or delete the corresponding test functions.

- `src-tauri/src/app_config.rs`
  - All `AppType::Gemini` enum variants, parser cases, dispatchers, and
    methods. Concretely the lines called out in the exploration report
    (L30, L31, L45, L46, L65, L68, L115, L116, L130, L131, L150, L153,
    L389, L390, L403, L404, L427, L437, L438, L457, L458, L501, L502,
    L516, L517, L727, L728, L766, L767, L840, L841, L879, L886, L887,
    L1159, L1172, L1183, L1209) — search the file for `Gemini` and remove
    each occurrence. After removal the enum's `Display`, `FromStr`, the
    `provider_config_dir_name()`, `config_file_name()`,
    `default_config()`, `parse_*_config()`, `supports_local_proxy()`,
    `current_config_path()`, `migrate_*()` and `get_official_*()` helpers
    should be re-checked for dead branches.

- `src-tauri/src/database/dao/providers_seed.rs`
  - L9, L10, L30, L65, L66, L69 — `gemini-official` seed entry — **delete**.

- `src-tauri/src/commands/provider.rs`
  - L306 `ensure_grokbuild_official_provider` is grok-specific (not gemini),
    leave it to PR-B.

- `src-tauri/src/commands/failover.rs`
  - L11 `require_failover_app()` currently passes Gemini and GrokBuild via
    `app.supports_local_proxy()`. After PR-A the gate is fine; after PR-B
    (or in a combined PR-A+B+C) `supports_local_proxy()` returns false for
    all current AppType values, so the gate becomes dead. Defer the gate
    removal to PR-C.

- `src-tauri/src/session_manager/providers/mod.rs` and `mod.rs`
  - L3 / L7 — `gemini` import and registration. **Delete** in PR-A (since
    the file `session_manager/providers/gemini.rs` itself is deleted in
    PR-A). The session_manager mod.rs `match` arms for `"gemini"` at
    L64/L114/L175/L208 (and anywhere else the agent id is matched) must
    also drop the `gemini` arm.

### A.4 Files to edit (Frontend)

- `src/config/appConfig.tsx`
  - L7 import of `GeminiIcon` — **delete**.
  - L23 `APP_IDS` — drop `"gemini"`.
  - L35 `DEFAULT_VISIBLE_APPS` — drop `gemini: true`.
  - L47 `SKILLS_APP_IDS` — drop `"gemini"`.
  - L54–57 `type ProxyAppId` — drop `"gemini"` (PR-C removes the type
    entirely, but we can drop it now).
  - L56 the `"claude" | "codex" | "gemini" | "grokbuild"` literal — drop
    `"gemini"` (PR-C removes the whole type).
  - L63 `PROXY_APP_IDS` — drop `"gemini"` (PR-C removes the whole const).
  - L92 `MCP_APP_IDS` — drop `"gemini"`.
  - L127–134 `APP_ICON_MAP.gemini` entry — **delete**.

- `src/App.tsx`
  - L240 `sharedFeatureApp !== "gemini" &&` — **delete** the `gemini` clause.
  - L317 `sharedFeatureApp === "gemini" ||` — **delete** the `gemini` clause.
  - Search for any other `"gemini"` literals (likely in fallback label
    maps) and remove.

- `src/components/BrandIcons.tsx`
  - L9 `import GeminiSvg from "@/icons/extracted/gemini.svg?url";` — **delete**.
  - L38–49 `export function GeminiIcon(...)` — **delete**.

- `src/components/AppSwitcher.tsx`
  - L34 `gemini: "gemini"` in `APP_ICON_NAME` — **delete**.
  - L46 `gemini: "Gemini"` in `APP_DISPLAY_NAME` — **delete**.

- `src/components/settings/SettingsPage.tsx` (audit only)
  - Settings renders per-agent configuration panels that may branch on
    `appId === "gemini"`. The exploration report did not flag a literal
    `"gemini"` in Settings, but a full `grep -n '"gemini"' src/components/settings/`
    is required before merging PR-A.

- `src/i18n/locales/{en,ja,zh,zh-TW}.json`
  - All keys enumerated in the exploration report §1.8 (geminiConfig,
    geminiConfigDir, geminiDesc, addGeminiProvider, apiFormatGeminiNative,
    apiHintGeminiNative, fullUrlHintGeminiNative, browsePlaceholderGemini,
    geminiTitle, the `gemini:` value at L909/L1717/L2013/L2629, and the
    `geminiConfig: { ... }` object at L1582–1600). Remove the keys from
    **all four** locale files in lockstep.

### A.5 Order of operations to keep `cargo build` green

1. Edit `src-tauri/src/services/mcp.rs` to drop Gemini rows in the
   `import_all_agents` table and the `AppType::Gemini` arms in the
   `remove_server_from_*` match. This stops `services::mcp` from calling
   into the soon-to-be-deleted `mcp::gemini` module.
2. Edit `src-tauri/src/mcp/mod.rs` to remove the `mod gemini;` and the
   `pub use gemini::{...}` block.
3. Delete `src-tauri/src/mcp/gemini.rs`. `cargo build` still passes because
   no other kept module references it.
4. Edit `src-tauri/src/lib.rs` to drop the `mod gemini_*;` lines, the
   `pub use mcp::{...gemini...}` rows, the `import_from_gemini` call in the
   auto-import loop, the `AppType::Gemini` arms in the prompts and
   `legacy_common_config` loops, and the `scrub_leaked_gemini_common_config`
   call. Also remove `"gemini"` from `PROXY_STARTUP_APP_TYPES`.
5. Edit `src-tauri/src/services/provider/mod.rs` to drop the
   `scrub_leaked_gemini_common_config` definition and its test references.
6. Edit `src-tauri/src/app_config.rs` to drop every `AppType::Gemini` arm.
7. Edit `src-tauri/src/database/dao/providers_seed.rs` to drop the
   `gemini-official` seed.
8. Delete `src-tauri/src/gemini_config.rs`, `src-tauri/src/gemini_mcp.rs`.
9. Delete `src-tauri/src/session_manager/providers/gemini.rs` and update
   `src-tauri/src/session_manager/{mod.rs,providers/mod.rs}` to drop the
   `"gemini"` import + match arm.
10. Delete `src-tauri/src/services/session_usage_gemini.rs` and drop
    `pub mod session_usage_gemini;` from `src-tauri/src/services/mod.rs`.
11. `cargo build && cargo test` — both should pass.

### A.6 Order of operations to keep `pnpm typecheck` + `pnpm test:unit` green

1. Edit `src/config/appConfig.tsx` to drop `"gemini"` from `APP_IDS`,
   `SKILLS_APP_IDS`, `MCP_APP_IDS`, and `APP_ICON_MAP`, and to drop the
   `GeminiIcon` import. This narrows all downstream types immediately.
2. Edit `src/components/mcp/UnifiedMcpPanel.tsx` and
   `src/components/skills/UnifiedSkillsPanel.tsx` — the for-loops over
   `MCP_APP_IDS` / `SKILLS_APP_IDS` will simply not iterate over `gemini`
   anymore. No code change needed beyond step 1, but verify with
   `pnpm typecheck` that no panel renders a stale "gemini" tab.
3. Edit `src/components/providers/forms/ProviderForm.tsx` (and any
   `*.tsx` that imports a deleted Gemini form). Most-likely the form
   imports `GeminiFormFields` via a per-app switch — drop the `gemini`
   case.
4. Delete `src/components/providers/forms/Gemini*` and the
   `useGeminiConfigState.ts` / `useGeminiCommonConfig.ts` hooks under
   `forms/hooks/`. Audit `src/components/providers/forms/hooks/index.ts`
   (or equivalent) for re-exports.
5. Edit `src/App.tsx` to drop the `"gemini"` literal in
   `sharedFeatureApp` branches and any other location.
6. Edit `src/components/AppSwitcher.tsx`, `src/components/BrandIcons.tsx`
   to drop the Gemini icon mapping / component.
7. Delete `src/config/geminiProviderPresets.ts`.
8. Delete `src/icons/extracted/gemini.svg`. Audit
   `src/icons/extracted/index.ts` and `metadata.ts` for the inline SVG and
   the metadata entry — remove both.
9. Run `pnpm typecheck` and `pnpm test:unit`. Some `tests/components/*`
   tests reference the deleted forms by name. Audit `tests/components/`
   (the report only flagged `ProviderForm.codexCatalog.test.tsx` and the
   form-level tests, not a `Gemini*` test). If a Gemini-specific test
   exists, delete it.
10. Edit i18n locale files to remove the Gemini keys (4 files in lockstep).

### A.7 Risk

- `app_config.rs` is a heavily-arm-matched enum. Forgetting a single arm
  produces a non-exhaustive-match error. Mitigation: `cargo build` will
  surface every missing arm in one go. **Do not split the `app_config.rs`
  edit into a separate commit** — keep it in PR-A so the compiler can
  report all missing arms at once.
- `services/mcp.rs` has an `import_all_agents` match table; missing an
  arm there is a silent no-op. Mitigation: grep for `"gemini"` after the
  edit to confirm zero hits.

---

## B) Grok Build agent (`grokbuild`)

### B.1 Files to delete (Rust)

```
src-tauri/src/grok_config.rs
src-tauri/src/mcp/grokbuild.rs
src-tauri/src/session_manager/providers/grokbuild.rs
src-tauri/src/services/session_usage_grokbuild.rs
src-tauri/src/services/subscription_grok.rs
src-tauri/src/commands/xai_oauth.rs
```

(Removing `session_manager/providers/grokbuild.rs` and
`session_usage_grokbuild.rs` is fine here; they are wholly retired in PR-D.
Deleting in PR-B avoids leaving dangling dead code.)

### B.2 Files to delete (Frontend)

```
src/config/grokBuildProviderPresets.ts
src/config/grokBuildProviderPresets.test.ts
src/components/providers/forms/GrokBuildProviderForm.tsx
src/components/providers/forms/XaiOAuthSection.tsx
src/components/providers/forms/hooks/useXaiOauth.ts
src/components/XaiOauthQuotaFooter.tsx
src/utils/grokBuildConfig.ts
src/utils/grokBuildConfig.test.ts
```

(If after step B.3 `grokBuildConfig.ts` is still imported by another test
or component, **stop and re-scope PR-B** — that file is shared, not
grokBuild-only.)

### B.3 Files to edit (Rust)

- `src-tauri/src/lib.rs`
  - L17 `mod grok_config;` — **delete**.
  - L52 `pub use grok_config::get_grok_config_path;` — **delete**.
  - L54, L55, L58 drop `import_from_grokbuild`,
    `remove_server_from_grokbuild`, `sync_single_server_to_grokbuild` from
    the `pub use mcp::{...}` block.
  - L955–959 the `McpService::import_from_grokbuild(...)` block + log
    lines (`"✓ Imported {count} MCP server(s) from Grok Build"`,
    `"No Grok Build MCP servers found"`,
    `"Failed to import Grok Build MCP"`) — **delete**.
  - L986 `AppType::GrokBuild` arm in prompts import loop — **delete**.
  - L1160–1169 the `XaiOAuthManager::new(...)` / `app.manage(XaiOAuthState(...))`
    initialization block, the `use crate::proxy::providers::xai_oauth_auth::XaiOAuthManager`
    and `use commands::XaiOAuthState` imports, and the
    `"✓ XaiOAuthManager initialized"` log — **delete**.
  - L1938 `PROXY_STARTUP_APP_TYPES` — drop `"grokbuild"`.
  - L2387–2401 `startup_restore_includes_enabled_grokbuild_route` test
    — **delete** (it asserts that grokbuild shows up in the proxy
    startup list).

- `src-tauri/src/mcp/mod.rs`
  - L17 `mod grokbuild;` — **delete**.
  - L34–36 `pub use grokbuild::{...}` — **delete**.

- `src-tauri/src/services/mod.rs`
  - L23 `pub mod session_usage_grokbuild;` — **delete**.
  - L31 `pub mod subscription_grok;` — **delete**.

- `src-tauri/src/services/mcp.rs`
  - L127 `sync_single_server_to_grokbuild` arm in dispatch — **delete**.
  - L174 `AppType::GrokBuild => mcp::remove_server_from_grokbuild(id)?`
    arm in `remove_server_from_*` match — **delete**.
  - L411–414 `pub fn import_from_grokbuild(state)` + body — **delete**.
  - L526 `("grokbuild", Self::import_from_grokbuild(state))` entry in
    `import_all_agents` table — **delete**.

- `src-tauri/src/commands/mod.rs`
  - L32 `mod xai_oauth;` — **delete**.
  - L68 `pub use xai_oauth::*;` — **delete**.

- `src-tauri/src/commands/provider.rs`
  - L306 `ensure_grokbuild_official_provider` — **delete**.

- `src-tauri/src/database/dao/providers_seed.rs`
  - L16, L75–76, L109–116 — `grokbuild-official` seed entry — **delete**.

- `src-tauri/src/app_config.rs`
  - All `AppType::GrokBuild` arms (the exploration report flagged
    `src-tauri/src/app_config.rs` along with the same line ranges as for
    Gemini). Remove every occurrence. **Do not combine with the Gemini
    PR** unless you re-run the entire `cargo build` cycle; the type
    system will list every missing arm in one compilation.

- `src-tauri/src/session_manager/{mod.rs,providers/mod.rs}`
  - L4 / L7 — `grokbuild` import and registration — **delete**.
  - Drop the `"grokbuild"` match arm from `mod.rs` (L66, L115, L176,
    L209).

- `src-tauri/src/services/session_usage.rs`
  - Inside `sync_all_unlocked`, the per-agent dispatch table (not shown
    in the report) most likely contains `sync_grokbuild_usage(...)` and
    `sync_gemini_usage(...)` arms — **delete** both. This is a likely
    follow-up to the report's gap; the implementer should grep
    `services/session_usage.rs` for `grok` and `gemini` to confirm.

- `src-tauri/src/commands/subscription.rs` is **generic** (takes `tool: String`);
  no change needed.

### B.4 Files to edit (Frontend)

- `src/config/appConfig.tsx`
  - L24 `APP_IDS` — drop `"grokbuild"`.
  - L36 `DEFAULT_VISIBLE_APPS` — drop `grokbuild: true`.
  - L48 `SKILLS_APP_IDS` — drop `"grokbuild"`.
  - L56 `type ProxyAppId` — drop `"grokbuild"`.
  - L64 `PROXY_APP_IDS` — drop `"grokbuild"`.
  - L93 `MCP_APP_IDS` — drop `"grokbuild"`.
  - L135–149 `APP_ICON_MAP.grokbuild` entry — **delete**.

- `src/App.tsx`
  - L237, L314 — drop the `grokbuild` clauses in the `sharedFeatureApp`
    branches.
  - L1573, L1574 — drop the `grokbuild` label / iconName.

- `src/components/AppSwitcher.tsx`
  - L35 `grokbuild: "grok"` in `APP_ICON_NAME` — **delete**.
  - L47 `grokbuild: "Grok Build"` in `APP_DISPLAY_NAME` — **delete**.

- `src/components/SubscriptionQuotaFooter.tsx`
  - L40 comment about Grok credit 额度 — **delete** (or reword).
  - L427 comment — **delete**.
  - L428 `appIdForExpiredHint={appId === "grokbuild" ? "grok" : appId}` —
    simplify to `appIdForExpiredHint={appId}`.

- `src/components/settings/AuthCenterPanel.tsx`
  - L9 `import { XaiOAuthSection } from "@/components/providers/forms/XaiOAuthSection";`
    — **delete**.
  - Remove the `<XaiOAuthSection />` JSX usage. Audit the rest of the
    file for any other `XaiOAuthSection` references.

- `src/i18n/locales/{en,ja,zh,zh-TW}.json`
  - Keys enumerated in the report §2.6 — `grokBuildRestartRequired`,
    `xaiOauthDescription`, `grokConfigDir`, `grokConfigDirDescription`,
    `browsePlaceholderGrok`, `grokbuild` value at L910/L1719/L2014,
    `grokBuild: { ... }` block at L1059, `xaiOauth: { ... }` block at
    L1435, plus any commonModelsDescription phrasing. Remove from all
    four locale files in lockstep.

### B.5 Order of operations to keep `cargo build` green

Same shape as A.5, applied to Grok:

1. `services/mcp.rs`: drop the grokbuild rows.
2. `mcp/mod.rs`: drop `mod grokbuild;` and the `pub use` block.
3. Delete `src-tauri/src/mcp/grokbuild.rs`.
4. `lib.rs`: drop `mod grok_config;`, the `pub use mcp::{...grokbuild...}`
   rows, the `import_from_grokbuild` call, the prompts arm, the
   XaiOAuthManager init, the `"grokbuild"` from `PROXY_STARTUP_APP_TYPES`,
   and the grokbuild startup test.
5. `services/mod.rs`: drop `pub mod session_usage_grokbuild;` and
   `pub mod subscription_grok;`.
6. `commands/mod.rs`: drop `mod xai_oauth;` and `pub use xai_oauth::*;`.
7. `commands/provider.rs`: drop `ensure_grokbuild_official_provider`.
8. `database/dao/providers_seed.rs`: drop `grokbuild-official` seed.
9. `app_config.rs`: drop every `AppType::GrokBuild` arm.
10. Delete `src-tauri/src/grok_config.rs`,
    `src-tauri/src/commands/xai_oauth.rs`,
    `src-tauri/src/services/subscription_grok.rs`.
11. Delete `src-tauri/src/session_manager/providers/grokbuild.rs` and
    update the session_manager mod.rs/providers/mod.rs to drop grokbuild
    import + match arms.
12. Delete `src-tauri/src/services/session_usage_grokbuild.rs`.
13. `cargo build && cargo test`.

### B.6 Order of operations to keep `pnpm typecheck` + `pnpm test:unit` green

1. `appConfig.tsx`: drop `"grokbuild"` from all constants and the
   `APP_ICON_MAP` entry.
2. `AuthCenterPanel.tsx`: remove `XaiOAuthSection` import + usage.
3. `SubscriptionQuotaFooter.tsx`: simplify the `grokbuild` branch.
4. `App.tsx`: drop the grokbuild literals in `sharedFeatureApp` and the
   label map.
5. `AppSwitcher.tsx`: drop the grokbuild rows in `APP_ICON_NAME` and
   `APP_DISPLAY_NAME`.
6. Delete the files listed in B.2. Audit `src/components/providers/forms/ProviderForm.tsx`
   for the `grokbuild` case in its per-agent form switch.
7. i18n: remove the keys from all four locales.
8. `pnpm typecheck && pnpm test:unit`.

### B.7 Risk

- `XaiOAuthManager` is referenced from the `commands/xai_oauth.rs` module
  (L1162) **and** from `services/subscription_grok.rs` (the `query_grok_quota`
  function). After deletion both `XaiOAuthState` (managed in `lib.rs:1168`)
  and the `query_grok_quota` consumer must have no callers; confirm by
  grepping `XaiOAuth` and `query_grok` across the workspace before
  deleting.
- `grokBuildConfig.ts` may be imported by other test files (e.g. shared
  Codex URL/model helpers). Audit before deleting in step B.2.

---

## C) Provider-routing / presets subsystem (the multi-provider switcher core)

This is the largest target. It replaces the proxy/failover/presets machinery
with a simple "active provider" state per agent (Step 4 design).

### C.1 Files to delete (Rust)

```
src-tauri/src/proxy/                                  # entire 40-file module + subdirs
src-tauri/src/commands/provider.rs                    # the proxy-routing part; agent CRUD migrated to a new module in Step 4
src-tauri/src/commands/failover.rs                    # entire file
src-tauri/src/commands/proxy.rs                       # entire file
src-tauri/src/commands/global_proxy.rs                # (if it only exists for global proxy settings; confirm with grep)
src-tauri/src/services/provider/                       # entire subdir if it only backs the routing layer
```

> **SPECULATIVE**: `services/provider/` was reported to contain
> `scrub_leaked_gemini_common_config` only. Verify the subdir is not used
> by any kept command before deleting the whole subdir. If it is, keep
> the subdir and only delete the gemini/grok functions.

### C.2 Files to delete (Frontend)

```
src/components/AppSwitcher.tsx                        # replaced by Step-4 agent selector
src/components/proxy/                                 # entire 8-file dir
src/components/providers/                             # entire 10-file dir + forms/ subdir
src/hooks/useDragSort.ts                              # was tied to provider-routing drag sort
```

### C.3 Preset files — decide obsolete vs reused

The exploration report enumerated 16 preset files. After PR-A and PR-B,
the agents that remain are `claude`, `codex`, `opencode`, `openclaw`,
`hermes`, `pi`. Preset policy in the agent-selector IA:

| File                                                     | Decision           | Reason                                                                                                                                                                        |
| -------------------------------------------------------- | ------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `src/config/claudeProviderPresets.ts` (+ test)           | **KEEP**           | Claude is a kept agent; presets represent real upstream offerings.                                                                                                            |
| `src/config/claudeDesktopProviderPresets.ts`             | **KEEP**           | Claude Desktop is a Claude routing variant.                                                                                                                                   |
| `src/config/codexProviderPresets.ts` (+ test)            | **KEEP**           | Codex is a kept agent.                                                                                                                                                        |
| `src/config/geminiProviderPresets.ts`                    | **DELETE** (PR-A)  | Gemini removed.                                                                                                                                                               |
| `src/config/grokBuildProviderPresets.ts` (+ test)        | **DELETE** (PR-B)  | Grok Build removed.                                                                                                                                                           |
| `src/config/hermesProviderPresets.ts`                    | **KEEP**           | Hermes is a kept agent.                                                                                                                                                       |
| `src/config/openclawProviderPresets.ts`                  | **KEEP**           | OpenClaw is a kept agent.                                                                                                                                                     |
| `src/config/opencodeProviderPresets.ts`                  | **KEEP**           | OpenCode is a kept agent.                                                                                                                                                     |
| `src/config/piProviderPresets.ts`                        | **KEEP**           | Pi is a kept agent.                                                                                                                                                           |
| `src/config/universalProviderPresets.ts`                 | **RETIRE in PR-C** | Cross-app preset referenced Gemini; after PR-A it must drop the Gemini entries. The whole concept of "universal providers" goes away with the proxy/failover removal in PR-C. |
| `src/config/userAgentPresets.ts`                         | **KEEP**           | Used by the kept CommonConfigEditor for custom user-agent fields.                                                                                                             |
| `src/config/codingPlanProviders.ts`                      | **KEEP**           | Not a routing preset; used by `src/components/providers/forms/CopilotAuthSection.tsx` (kept).                                                                                 |
| `src/config/mcpPresets.ts`                               | **KEEP**           | Used by the kept UnifiedMcpPanel.                                                                                                                                             |
| `src/config/constants.ts`                                | **AUDIT**          | May contain routing-only constants. Grep before deletion.                                                                                                                     |
| `src/config/codexTemplates.ts`                           | **KEEP**           | Codex templates, used by `codexProviderPresets`.                                                                                                                              |
| `src/config/piThinkingProfiles.ts` / `piModelCatalog.ts` | **KEEP**           | Pi-specific.                                                                                                                                                                  |

The other "test-only" preset files at the bottom of the report
(`codexReasoningLevelPresets.ts`, `codexChatProviderPresets.ts`,
`mimoTokenPlanPresets.ts`, `qianfanTokenPlanPresets.ts`,
`doubaoSeedPresets.ts`, `longcatProviderPresets.ts`,
`xaiOauthProviderPresets.ts`, `therouterProviderPresets.ts`,
`therouterOpenCodeOpenClawPresets.ts`) are
**// SPECULATIVE** — confirm with `ls src/config/` before deletion. Any
file matching `*OAuth*` or `*xai*` is deleted in PR-B; any other
agent-routing preset (e.g. `therouterProviderPresets.ts`) is deleted in
PR-C.

### C.4 Files to edit (Frontend)

- `src/config/appConfig.tsx`
  - L54–60 `type ProxyAppId`, `export const PROXY_APP_IDS`,
    `export function isProxyAppId` — **delete** the type, the const, and
    the type guard.
  - Search for `PROXY_APP_IDS` and `isProxyAppId` everywhere in the
    frontend (see §3.10 of the exploration report) and remove all
    references. After PR-C, the `ProviderList.tsx`, `ProviderCard.tsx`,
    `ProxyTabContent.tsx`, `ProxyPanel.tsx`, etc. are all deleted in
    step C.2, so the references go with them.
  - `APP_IDS` after PR-A+B is `["claude","codex","opencode","openclaw","hermes","pi"]`.

- `src/App.tsx`
  - L65 `import { AppSwitcher } ...` — **delete**.
  - L67 `import { ProviderList } ...` — **delete**.
  - L74–77 `import { ProxyToggle ... }`, `ClaudeDesktopRouteToggle`,
    `FailoverToggle`, `RoutingActivationBrand` — **delete**.
  - L113, L280–292, L1372–1385 — all the `isProxyAppId` / `proxyAppId`
    branches — **delete** (after the JSX they wrap is removed).
  - L1400–1404 the `<AppSwitcher>` JSX — **delete**.
  - L1325, L1378, L1382, L1384–1385 the `<RoutingActivationBrand>`,
    `<ClaudeDesktopRouteToggle>`, `<ProxyToggle>`, `<FailoverToggle>` JSX
    — **delete**.

### C.5 Files to edit (Rust)

- `src-tauri/src/lib.rs`
  - L32 `mod proxy;` — **delete**.
  - L1938 `const PROXY_STARTUP_APP_TYPES` — **delete** (its only consumer
    is the `for app_type in PROXY_STARTUP_APP_TYPES` loop at L1942, which
    iterates `app_state.proxy_service` to start the proxy server).
  - L1942–1990 (approx) the startup-restore loop — **delete**.
  - L1160–1169 (XaiOAuthManager init already removed in PR-B).
  - L2387–2401 (grokbuild startup test already removed in PR-B).
  - After PR-C: every `use crate::proxy::...` reference in `lib.rs` and
    `commands/*.rs` must be gone; grep before merging.

- `src-tauri/src/services/mod.rs`
  - L16 `pub mod proxy;` — **delete**.

- `src-tauri/src/services/proxy.rs` (or `services/provider.rs`) — **delete**.

- `src-tauri/src/commands/mod.rs`
  - L11 `mod failover;` — **delete**.
  - L25 `mod proxy;` — **delete**.
  - L48, L62 `pub use failover::*;` and `pub use proxy::*;` — **delete**.

- `src-tauri/src/database/dao/mod.rs`
  - L5 `pub mod failover;` — **delete**.
  - L11 `pub mod proxy;` — **delete** only after confirming `proxy_request_logs`
    (and any other table used by the kept Usage dashboard) is not read by
    this DAO. If the proxy DAO is the read path for the Usage dashboard,
    **keep the DAO and delete only the failover DAO** in PR-C. Audit
    with `grep -rn "dao::proxy" src-tauri/src/`.

### C.6 Per-agent provider forms that must be removed

These are referenced by `ProviderForm.tsx` and become dead after the
provider-routing layer is removed. They are removed in PR-C:

```
src/components/providers/forms/CopilotAuthSection.tsx          # see note below
src/components/providers/forms/XaiOAuthSection.tsx            # already gone in PR-B
src/components/providers/forms/ProviderForm.tsx
src/components/providers/forms/ProviderAdvancedConfig.tsx
src/components/providers/forms/ProviderPresetSelector.tsx
src/components/providers/forms/StructuredOptionsEditor.tsx
src/components/providers/forms/RequestHeadersEditor.tsx
src/components/providers/forms/LocalProxyRequestOverridesField.tsx
```

> **Note on `CopilotAuthSection.tsx`**: the report only mentioned it in
> the forms list. It is referenced by `AuthCenterPanel.tsx` (kept) for
> Claude routing. The Step-4 IA replaces the Settings → Auth center with
> per-agent OAuth tabs. Confirm `CopilotAuthSection.tsx` is no longer
> needed after Step 4, and migrate the AuthCenterPanel accordingly
> before deleting it. **Do not delete it in PR-C** unless Step 4 has
> already landed and the new OAuth tab covers Copilot.

### C.7 Order of operations to keep `cargo build` + `pnpm typecheck` green

This is the largest PR. Recommend landing it in **three sub-PRs**, each
green on its own:

#### C-sub-1: stop using the proxy at runtime (no UI changes)

1. Remove `PROXY_STARTUP_APP_TYPES` and its loop in `lib.rs`. The proxy
   server is no longer auto-started.
2. Stop the proxy in `lib.rs::run()`'s shutdown path (if any).
3. Verify `cargo build && cargo test` still pass — the proxy module is
   still in the tree but unused.

#### C-sub-2: stop calling proxy/failover commands from the frontend

1. Remove the `invoke()` calls for `start_proxy_server`, `stop_proxy_*`,
   `get_proxy_*`, `get_failover_queue`, `set_auto_failover_enabled`,
   `switch_proxy_provider`, `set_proxy_takeover_for_app`, etc. from
   `src/App.tsx`, `src/components/proxy/`, `src/components/settings/ProxyTabContent.tsx`,
   `src/hooks/useDragSort.ts`. The `tsc` compiler will then complain
   about unused imports — clean them up.
2. `pnpm typecheck && pnpm test:unit` should pass.
3. (No Rust changes; proxy module still on disk but unreferenced from
   the frontend.)

#### C-sub-3: delete the modules

1. `src-tauri/src/proxy/` (40 files).
2. `src-tauri/src/services/proxy.rs`, `services/provider/` (after audit).
3. `src-tauri/src/commands/{provider.rs, failover.rs, proxy.rs, global_proxy.rs}`.
4. `src-tauri/src/database/dao/failover.rs`; `dao/proxy.rs` only if not
   used by the Usage dashboard.
5. `src/components/AppSwitcher.tsx`, `src/components/proxy/`,
   `src/components/providers/`, `src/hooks/useDragSort.ts`.
6. Edit `src/config/appConfig.tsx` to delete `ProxyAppId`, `PROXY_APP_IDS`,
   `isProxyAppId`.
7. Edit i18n: drop the routing-specific keys (ProxyPanel._, FailoverToggle._,
   AutoFailover._, ProxyTabContent._ — list will be enumerated by a
   follow-up `grep` against `src/i18n/` when the implementation PR opens).
8. `cargo build && cargo test && pnpm typecheck && pnpm test:unit`.

### C.8 Shared utilities preserved

(See §0.2.) The agent-keyed data-driven provider form is **not** built in
this PR. Until Step 4 lands, the kept agents still need a way to edit
their per-agent config. The proposal:

- After PR-C, the only place an end user can configure a kept agent's
  provider is the new `<AgentTab>` from Step 4 (Context Files, MCP,
  Skills, etc.). Step-4 design covers this.
- The legacy `ProviderList` is removed in C-sub-3; until then, PR-C
  leaves a no-op stub. **Alternative**: keep `ProviderList` (read-only)
  for kept agents only, and remove it in the Step-4 implementation PR.

### C.9 Risk

- This PR is the highest-risk of the four. The proxy module is
  intertwined with `commands/provider.rs`, `services/provider/`, the
  `dao/proxy.rs` and the Usage dashboard. Mitigation: ship C-sub-1
  (kill the auto-start) as the first PR; the module is then dead code
  that can be safely removed in C-sub-3.
- The `universal` panel (`src/components/universal/UniversalProviderPanel.tsx`
  - card + form modal) is the user-visible bridge between
    `src/config/universalProviderPresets.ts` and the proxy/failover
    machinery. After C-sub-1 the bridge is dead. Deleting the
    universalProviderPresets and the `universal/` components can be
    lumped into C-sub-3.

---

## D) Session-manager subsystem (UI + backend + DB + tests)

### D.1 Files to delete (Rust)

```
src-tauri/src/session_manager/                          # entire module (mod.rs, providers/*, terminal/*)
src-tauri/src/commands/session_manager.rs
src-tauri/src/services/session_usage_gemini.rs          # already gone in PR-A
src-tauri/src/services/session_usage_grokbuild.rs       # already gone in PR-B
```

### D.2 Files to keep but trim (Rust)

- `src-tauri/src/services/session_usage.rs` — the entry point stays
  because `run_session_sync` in `lib.rs` still calls it. After PR-D
  the function body becomes a no-op (only codex/opencode/pi remain
  and their `sync_*_usage` functions survive). Decide:
  - **Option D-1** (recommended): keep `session_usage.rs` and
    `session_usage_codex.rs` / `session_usage_opencode.rs` /
    `session_usage_pi.rs` for now, because they feed the Usage
    dashboard. The `sync_all_unlocked` body simply drops the
    gemini/grokbuild arms and keeps the rest.
  - **Option D-2** (later, in Step 4): the Usage dashboard moves to
    a "per-agent activity" tab. At that point `session_usage*` is
    renamed to `<agent>_activity.rs` or retired entirely.

### D.3 Database changes

- `src-tauri/src/database/schema.rs`
  - **Drop table `session_log_sync`** (created at L301 in the initial
    schema and again at L1236 in the v7→v8 migration). Both CREATE
    statements must be removed **or** a new vN→vN+1 migration must
    `DROP TABLE IF EXISTS session_log_sync`. **Recommend the new
    migration** — never edit the old one.
  - **Drop table `session_usage_dedup`** (created at L314 in the
    initial schema and at L1554 in v16→v17). Same rule: keep the old
    CREATE in the history, add a new vN→vN+1 migration that drops it.
  - **Drop column `proxy_request_logs.session_id`** (L207, L715). The
    column is only used by the session-manager's TOC to group messages
    by session. The Usage dashboard does not reference it. **Audit
    with `grep -rn "session_id" src/` first** — if anything in the kept
    Usage dashboard reads it, keep the column and just drop the index
    at L223.
  - **Drop the session-related indices**: `idx_request_logs_session`
    (L223) and `idx_session_usage_dedup_semantic` (L325, L1561–1562).
  - **Retire the v15→v16 migration body**: this migration's _only_
    effect today is to call
    `services::session_usage_codex::reset_codex_usage_on_conn`. That
    function is used by **no** other code path. The migration should
    become a no-op (`fn migrate_v15_to_v16(_conn) { Ok(()) }`) so the
    existing dispatch loop at L441–541 still works without removal.
    The original `reset_codex_usage_on_conn` call site is the only
    consumer; once it's gone, the function can be deleted from
    `session_usage_codex.rs`.
  - **Retire v16→v17**: this migration creates
    `session_usage_dedup` and the matching index. After the table is
    dropped by a later migration, the original v16→v17 block can be
    left in place (it creates a table that the next migration drops)
    OR replaced with a no-op. Recommend leaving it in place for
    audit-trail reasons.
  - **Add a new vN→vN+1 migration** (`migrate_v17_to_v18`) that:
    1. `DROP INDEX IF EXISTS idx_session_usage_dedup_semantic;`
    2. `DROP INDEX IF EXISTS idx_request_logs_session;`
    3. `DROP TABLE IF EXISTS session_usage_dedup;`
    4. `DROP TABLE IF EXISTS session_log_sync;`
       (Step order: indices before tables.)
  - **Tests in schema.rs** (L3355 `migrate_v15_to_v16_resets_only_codex_session_usage`,
    L3396 `migrate_v16_to_v17_creates_session_usage_dedup_ledger`,
    L3370/3381/3387/3405 — all the `INSERT INTO session_log_sync` and
    `INSERT INTO session_usage_dedup` snippets inside tests):
    delete these tests.
  - **Backup file `src-tauri/src/database/backup.rs`** L92–104 —
    the `SESSION_TABLES` set includes `session_log_sync` and
    `session_usage_dedup`. Remove both entries from the set. Also
    update the backup-restore tests at L2121–2338 / L2590+ that
    `INSERT INTO session_log_sync` — these are full-backup
    round-trip tests; either drop them or replace the seed data
    with a non-session table.

### D.4 Files to delete (Frontend)

```
src/components/sessions/SessionManagerPage.tsx
src/components/sessions/SessionItem.tsx
src/components/sessions/SessionMessageItem.tsx
src/components/sessions/SessionToc.tsx
src/components/sessions/utils.ts
```

### D.5 Files to delete (Tests)

```
tests/components/SessionManagerPage.test.tsx
tests/components/sessionUtils.test.ts
```

### D.6 Doc to delete

```
session-manager.md                                       # repo root PRD
```

### D.7 Files to edit (Frontend)

- `src/App.tsx`
  - L98 `import { SessionManagerPage } from ...` — **delete**.
  - L1077 the `<SessionManagerPage ... />` render — **delete**.
  - L1314 `currentView === "sessions" && t("sessionManager.title")` —
    **delete** (and the surrounding "sessions" view branch).
  - L1667 / L1709 `title={t("sessionManager.title")}` — **delete** or
    rewire to the new Step-4 view.
  - The `currentView === "sessions"` (and any other routing branch
    that points to the session manager) — **delete** from the view
    switch.

### D.8 Files to edit (Rust)

- `src-tauri/src/lib.rs`
  - L34 `mod session_manager;` — **delete**.
  - L1270 `async fn run_session_sync(db, backfill)` — **delete**
    (after `services::session_usage::sync_all_unlocked` is inlined
    or made a no-op; see D.2).
  - L1295 / L1305 the `run_session_sync(...)` startup + periodic
    invocations — **delete** (or replace with the kept
    `services::session_usage::sync_all_unlocked(db)` direct call if
    the Usage dashboard still needs session-usage rollups).
  - L1464 `commands::get_pi_session_discovery` invoke_handler entry
    — **delete** if no longer referenced from the frontend.
  - L1598 `commands::sync_session_usage` invoke_handler entry
    — **delete** if no longer referenced.
  - L1607–1610 `list_sessions`, `get_session_messages`,
    `delete_session`, `delete_sessions` invoke_handler entries
    — **delete**.

- `src-tauri/src/commands/mod.rs`
  - L26 `mod session_manager;` — **delete**.
  - L63 `pub use session_manager::*;` — **delete**.

- `src-tauri/src/services/mod.rs`
  - L20 `pub mod session_usage;` — **keep** (still feeds the Usage
    dashboard per D.2).
  - L21, L24, L25 — `session_usage_codex`, `session_usage_opencode`,
    `session_usage_pi` — **keep**.

### D.9 i18n cleanup

Remove the `sessionManager: { ... }` block (L1072–1084) and any
`session*` keys from `src/i18n/locales/{en,ja,zh,zh-TW}.json` in
lockstep.

### D.10 Order of operations to keep `cargo build` + `pnpm typecheck` green

The migration is the trickiest part. Order:

1. **Frontend first.** Delete `src/components/sessions/*` and the two
   tests. Remove the imports + JSX from `src/App.tsx`. Remove i18n
   keys. `pnpm typecheck && pnpm test:unit` should pass.
2. **Rust commands.** Delete `src-tauri/src/commands/session_manager.rs`
   and update `commands/mod.rs`. `cargo build` will fail on `lib.rs`
   invoke_handler entries — remove them.
3. **Rust session_manager module.** Delete `src-tauri/src/session_manager/`
   and drop `mod session_manager;` from `lib.rs`. `cargo build` will
   fail on the call to `session_manager::xxx` inside `lib.rs` — update
   to no-op (or remove the call site entirely).
4. **DB schema.** Add `migrate_v17_to_v18` that drops the four session
   objects. **Do not** edit the old migrations. Bump the dispatcher
   loop at L441–541 to call the new migration.
5. **Backup file.** Remove `session_log_sync` / `session_usage_dedup`
   from `SESSION_TABLES`. Update the affected backup-restore tests.
6. **Tests.** Delete the two migration tests in `schema.rs` and the
   components tests (already deleted in step 1).
7. `cargo build && cargo test && pnpm typecheck && pnpm test:unit`.

### D.11 Risk

- **DB downgrade**: downgrading from a build with this PR to a build
  without it is not supported. Document this in the PR description
  and CHANGELOG.
- **Backup round-trip tests**: the tests at L2121+ in `backup.rs`
  insert into `session_log_sync`. If we drop the table in step D.3,
  the test seeds break. Mitigation: rewrite the test seeds to use a
  kept table (e.g. `proxy_request_logs`) or delete the round-trip
  tests if no other table is a good fit.
- **`session_usage_codex::reset_codex_usage_on_conn`** has callers
  outside the v15→v16 migration. Audit with
  `grep -rn "reset_codex_usage_on_conn" src-tauri/src/` before
  removing the function in step D.3.

---

## E) Cross-cutting checklist (apply in each PR)

Before opening a PR, re-confirm:

1. `grep -rn '"gemini"' src/` — must return 0 hits (after PR-A).
2. `grep -rn '"grokbuild"' src/` — must return 0 hits (after PR-B).
3. `grep -rn 'PROXY_APP_IDS\|isProxyAppId\|ProxyAppId' src/` — must
   return 0 hits (after PR-C).
4. `grep -rn 'from.*sessions/' src/` — must return 0 hits (after PR-D).
5. `grep -rn 'session_log_sync\|session_usage_dedup' src-tauri/src/`
   — must return 0 hits outside the new vN→vN+1 migration that drops
   them (after PR-D).
6. `grep -rn 'XaiOAuth\|grok_config\|gro kbuild\|gemini_config' src/`
   — must return 0 hits (after PR-A and PR-B).
7. All four locale files in `src/i18n/locales/` are in lockstep — a
   `diff -q src/i18n/locales/en.json src/i18n/locales/ja.json` should
   not flag structural differences (only translations).
8. `cargo build && cargo test && pnpm typecheck && pnpm test:unit`
   all pass.

## F) Open questions for the implementation PR

- (D-2) Should `services::session_usage*` survive past Step 3, or be
  retired in Step 4? Recommendation: keep, gated on per-agent
  activity.
- (C-6) `CopilotAuthSection.tsx` and the wider Copilot OAuth flow —
  is it really routed through the proxy, or is it independent? If
  independent, **keep it** and add a note in AGENTS.md.
- (A.4) The `metadata.ts` and `extracted/index.ts` icon tables — the
  exploration report flagged entries for gemini in `metadata.ts`
  (L30, L45, L76, L93, L186, L334–336, L366, L619). Are any of those
  shared (e.g. for "claude-desktop" or generic keywords)? Audit
  before deleting.
- (C-3) The "test-only" preset files at the bottom of §3.6 — confirm
  their names with `ls src/config/` before deciding to delete.

---

## G) Summary table (file-level delete count)

| Target             | Rust files                                                       | Frontend files                                                    | Tests                         | Docs                     |
| ------------------ | ---------------------------------------------------------------- | ----------------------------------------------------------------- | ----------------------------- | ------------------------ |
| A. Gemini          | 5 (incl. 1 in `mcp/`, 1 in `session_manager/`)                   | 8 (incl. 1 svg, 2 hooks)                                          | 0 standalone (audit `tests/`) | 0                        |
| B. Grok Build      | 6 (incl. 1 in `mcp/`, 1 in `session_manager/`, 1 in `commands/`) | 8 (incl. 2 utils)                                                 | 1                             | 0                        |
| C. Proxy/Presets   | 40+ (entire `proxy/`) + 3 commands + 1 service + up to 2 daos    | 1 (`AppSwitcher.tsx`) + 8 (`proxy/`) + 30 (`providers/`) + 1 hook | 0 standalone                  | 0                        |
| D. Session manager | 12 (entire `session_manager/`) + 1 command                       | 5 (`sessions/`)                                                   | 2                             | 1 (`session-manager.md`) |

**Total**: ~63 Rust files, ~50 frontend files, 3 test files, 1 doc.

This is the implementation plan. No file has been modified, deleted, or
created in this PR. Execution of the four PRs (A, B, C, D) above is
gated on the mayor starting the staged convoy.
