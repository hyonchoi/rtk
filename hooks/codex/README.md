# Codex CLI Hooks

> Part of [`hooks/`](../README.md) — see also [`src/hooks/`](../../src/hooks/README.md) for installation code

## Specifics

- Native `PreToolUse` hook via `rtk hook codex`
- Rewrites use Codex's `permissionDecision: "allow"` + `updatedInput` shape
- `rtk-awareness.md` is injected into `AGENTS.md` with an `@RTK.md` reference
- Global installs go to `$CODEX_HOME` when set, otherwise `~/.codex/`
- Project installs write the hook to `.codex/hooks.json` and awareness files to `AGENTS.md` / `RTK.md`
