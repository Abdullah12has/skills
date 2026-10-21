# Switchboard

An agent skill that hands bounded coding, repository analysis, and reviews to local Codex, Claude Code, and Cursor Agent CLIs. The orchestrating agent scopes each task, picks the requested account and model, and verifies results before integrating them.

## Install

Needs a host that loads skills and runs local commands (Claude Code, Codex, or similar), at least one authenticated worker CLI, and Git for parallel edits.

Copy this folder to `~/.claude/skills/switchboard` (Claude Code) or `~/.codex/skills/switchboard` (Codex). Install only one copy per host.

## Use

- "Use switchboard to ask Codex to review my uncommitted changes."
- "Delegate this fix to Cursor, then review the diff and run the tests."
- "Dry run: split these edits between Claude and Codex."

For a named account, give its config directory (`CODEX_HOME` or `CLAUDE_CONFIG_DIR`); the skill never guesses one.

The skill uses each CLI's own permission controls and refuses bypass flags. Only Codex's sandbox is OS-enforced; the orchestrator still reviews every change.

## Evals

[evals/evals.json](evals/evals.json) holds simulated scenarios. To run them, give fresh evaluators SKILL.md and the prompts only, keep expectations with the grader, and block real CLI launches.

Last checked live against codex-cli 0.163.0, Claude Code, and Cursor Agent on macOS, October 2026.
