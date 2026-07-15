# Knowledge Patch

`knowledge-patch` is an offline plugin that ships knowledge patches for specific technology versions as native Claude Code and Codex skills. The bundled patches are available immediately after installation; no downloader is involved.

Knowledge patches are derived from the respective projects' official release notes and documentation. This project is not affiliated with or endorsed by those projects.

## Install

Use the Claude Code or Codex instructions in the repository `README.md`, then run `/knowledge-patch:setup` in Claude Code or invoke `$knowledge-patch-setup` in Codex.

## Commands

- `/knowledge-patch:setup` detects relevant bundled patches and writes activation state after confirmation.
- `/knowledge-patch:activate` activates patches by name or alias, removes patches when asked to deactivate them, and clears activation when passed `none`.
- `/knowledge-patch:status` shows the bundled patch catalog, activation state, generated-at age, content hashes, and missing artifacts.

## Skills

- `using-knowledge-patch` is the gateway skill. Load it before writing, reviewing, debugging, or administering a technology that may have a bundled patch.
- `knowledge-patch-setup` supports setup, activation, and deactivation workflows.
- `knowledge-patch-status` supports status checks.
- Technology patch skills are listed in `catalog/patches.json` and the generated gateway index at `skills/using-knowledge-patch/references/patch-index.md`.

## Activation

Activation is priority state, not installation. The preferred project state file is `.knowledge-patch/activation.json`; user-level state can live at `$XDG_STATE_HOME/knowledge-patch/activation.json`, and `$KNOWLEDGE_PATCH_STATE` can point to a specific state file.

All bundled technology patches already live under `skills/`. Activation tells agents which patches the user intentionally wants prioritized. Deactivation removes matching patch names and reasons from that state; `none` clears all active patch names and activation reasons.

## SessionStart Hook

`hooks/session-start` is optional. When Claude Code runs the SessionStart hook from `hooks/hooks.json`, it can add gateway guidance and active patch names to the session context. Manual use through `using-knowledge-patch`, `/knowledge-patch:setup`, and `/knowledge-patch:activate` remains the reliable path.

The `startup|clear|compact` matcher is deliberate. `resume` is excluded because resumed context already carries the earlier injection. The hook prefers `$CLAUDE_PROJECT_DIR` for project state and exits silently if state is malformed or an unexpected error occurs.

## Detection Signals

In `catalog/detection.json`, a plain file glob means that the path's existence is evidence. A signal in `glob::marker` form requires a matching file whose contents contain the literal marker. Content markers keep shared manifests such as `package.json` and `Cargo.toml` from activating unrelated patches.

## Shipped Layout

- `.claude-plugin/marketplace.json` exposes the Claude Code marketplace entry.
- `.agents/plugins/marketplace.json` exposes the Codex marketplace entry.
- `plugins/knowledge-patch/.claude-plugin/plugin.json` is the Claude Code plugin manifest.
- `plugins/knowledge-patch/.codex-plugin/plugin.json` is the Codex plugin manifest and declares `"skills": "./skills/"`.
- `plugins/knowledge-patch/skills/` contains the gateway skill, helper skills, and technology patch skills.
- `plugins/knowledge-patch/catalog/` contains patch, alias, detection, and build catalogs.
- `plugins/knowledge-patch/commands/` contains `setup.md`, `activate.md`, and `status.md`.
- `plugins/knowledge-patch/hooks/` contains the optional SessionStart hook.
