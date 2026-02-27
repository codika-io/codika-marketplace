# Codika Plugins (Public)

Public Claude Code plugin providing skills for the `codika-helper` CLI (`@codika-io/helper-sdk`).

## What This Repo Contains

This is a pure-documentation plugin — no code, no dependencies, no secrets. Each skill is a `SKILL.md` file that teaches Claude Code agents how to use the CLI.

## Skills

- `setup-codika` — Install the CLI globally and run `codika-helper login`
- `create-project` — Create a project via `codika-helper project create`
- `deploy-use-case` — Validate then deploy via `codika-helper verify` + `codika-helper deploy`
- `verify-use-case` — Validate workflows via `codika-helper verify use-case`

## Conventions

- Skills must not contain secrets, API keys, or internal URLs
- Skills should reference CLI commands and flags, not internal implementation details
- Skills should guide agents to use `setup-codika` when authentication fails
- Keep SKILL.md files self-contained — each should work independently
