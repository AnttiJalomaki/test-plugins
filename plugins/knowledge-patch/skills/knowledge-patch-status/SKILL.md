---
name: knowledge-patch-status
description: Report bundled patch catalog status, active priority state, generated-at age, and missing artifacts for the knowledge-patch plugin.
---

# Knowledge Patch Status

Use this skill when the user asks what knowledge patches are available, active, stale, or missing.

## Status Checks

1. Read `../../.codex-plugin/plugin.json` or `../../.claude-plugin/plugin.json` for the build version.
2. Read `../../catalog/patches.json` for bundled patches, covered versions, content hashes, generated timestamps, and coverage paths.
3. Read activation state from `$KNOWLEDGE_PATCH_STATE`, `.knowledge-patch/activation.json`, or `$XDG_STATE_HOME/knowledge-patch/activation.json`.
4. Verify that each catalog `path` exists and that each `coverage_path` exists.
5. For every patch, report its `generated_at` timestamp and age relative to now in an understandable unit such as days; flag missing, invalid, or notably old timestamps instead of hiding them.
6. Report active patches first, then matching bundled patches, then any catalog, staleness, or artifact problems.

If no activation state exists, say that all bundled patches remain available as native skills and activation is only a priority hint.

Do not fetch network resources and do not copy skills into another directory while checking status.
