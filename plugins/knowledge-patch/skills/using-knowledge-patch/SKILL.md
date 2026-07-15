---
name: using-knowledge-patch
description: Gateway for loading matching technology knowledge-patch skills before writing, reviewing, debugging, or VPS/admin work.
---

# Using Knowledge Patch

Use this gateway when a task involves a technology that may have a bundled knowledge patch. The plugin ships tech patches as native skills and generated catalogs; it does not download patches or copy them into another skill directory.

## Routing Rules

1. Before writing, reviewing, debugging, planning, or administering a patched technology, load the matching `knowledge-patch:<tech>-knowledge-patch` skill.
2. Determine the project's pinned technology version from its manifest first; consult its lockfile or configuration only when the manifest does not pin it. Apply only notes introduced at or below that version. If the project version is newer than the catalog's `covered_version`, state that the patch may be stale and trust project reality—its manifest, code, tests, and current behavior—over the patch.
3. If the user manually activated patches, treat that as confirmed intent and load those patches before related work.
4. If multiple patches apply, load the narrowest relevant set. Do not load every patch just because the catalog exists.
5. Within the applicable version floor, knowledge patches override stale model memory when they conflict. Follow the loaded patch and mention the current behavior when it matters.
6. If no exact patch exists, continue normally and say that no bundled patch matched when that affects confidence.

## Finding Patches

Do not keep or inline a full patch list in this gateway. Use:

- `../../catalog/patches.json` for the generated machine-readable catalog.
- `references/patch-index.md` for the generated human-readable index.
- `../../catalog/aliases.json` for manual names, aliases, and fuzzy activation.
- `../../catalog/detection.json` for setup/status detection hints.

## Activation State

Activation is priority state, not installation. A typical state file is:

```json
{
  "schema_version": 1,
  "active_patches": [
    "postgresql-knowledge-patch",
    "docker-knowledge-patch"
  ],
  "activation_reasons": {
    "postgresql-knowledge-patch": "manual: user plans to write SQL",
    "docker-knowledge-patch": "detected: docker-compose.yml"
  }
}
```

Check `$KNOWLEDGE_PATCH_STATE`, then `.knowledge-patch/activation.json` in the current project, then `$XDG_STATE_HOME/knowledge-patch/activation.json` when the user asks for setup, activation, deactivation, or status.

## VPS And Admin Work

For server, VPS, deployment, or machine-administration tasks, consider whether OS, init system, container, proxy, database, TLS, and observability patches apply. Look for concrete evidence such as `/etc/os-release`, systemd, Docker or Podman, compose files, Nginx/Caddy/Traefik config, PostgreSQL config, TLS assets, SSH context, CI/container hints, and project deployment files.

Ask before service-impacting admin changes. Do not assume a normal Linux shell is production. Manual activation beats automatic detection.

## Hook Behavior

The session-start hook is only a hint layer. It may inject this gateway's core rule and active patch names, but the plugin must work fully when hooks never fire.
