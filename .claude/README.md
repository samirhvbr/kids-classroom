# Claude Code configuration — KIDS/MARTHINA

Stack: **Laravel**.

## Files
- `settings.json` — the active profile (permissions and effort; it does not choose the model).
- `settings.local.json` — local override (gitignored), takes precedence over `settings.json`.

## Model
- **The model is the user's choice, made with `/model`, per session**, and a subagent
  inherits the session's model.
- **Nothing in this repository chooses it** (repodocs ADR-027): `settings.json` carries no
  `model`, `fallbackModel` or `availableModels`, and its `env` carries no `ANTHROPIC_MODEL`,
  `ANTHROPIC_DEFAULT_*_MODEL` or `CLAUDE_CODE_SUBAGENT_MODEL`.
- There are no stand-by profiles to `cp` over `settings.json` — `/model` does that job.

## Effort
- Effort `max` through the `CLAUDE_CODE_EFFORT_LEVEL` env var (the `effortLevel` field
  itself only accepts low/medium/high/xhigh).

## Permissions
- `defaultMode: plan`; security denies (rm -rf, force push, reset --hard, clean -fd, curl|sh).
- **git push allowed** (in `allow`).
