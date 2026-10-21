---
name: switchboard
description: Delegate bounded coding, repository analysis, or reviews to local Codex, Claude Code, or Cursor Agent CLIs. Use when the user asks to hand work to one of these CLIs, choose a worker account or model, split work between workers, or get an independent review. Not for ordinary coding requests or desktop app support.
---

# Switchboard

You orchestrate headless CLI workers and own integration and verification. A worker's "done" is evidence to inspect, not proof.

## Prepare

1. Extract directory, task, worker, account, model, allowed edits, and success criteria. Default to the current workspace and the CLI's configured account and model. Dry run: show commands only; no launches, auth checks, model listings, or network calls.
2. Find the executable (`codex`, `claude`, `cursor-agent`; confirm a bare `agent` is Cursor) and read its `--help`. Installed help decides which flags exist; never invent one. Use real paths, not aliases or wrappers that may add bypass flags. Don't install or update software unless asked.
3. Before edits, read repo instructions (`AGENTS.md`, `CLAUDE.md`) and `git status`. Pass relevant rules to the worker explicitly, including commit identity and attribution. Commits, pushes, deploys, credential changes, and unrelated edits stay with you unless the user authorized delegating them.
4. Workers don't inherit your sandbox. If credentials or network are blocked, diagnose and request the minimum host approval only where the host allows it. Never prescribe escalation unconditionally.

## Accounts and models

An account is a config directory: `CODEX_HOME` for Codex, `CLAUDE_CONFIG_DIR` for Claude. Use only a directory the user supplied or their config establishes. Names like `work` map to nothing universally: ask, and never create a directory or fall back to another account. No account requested: leave the environment alone.

For an explicitly selected subscription account, stop inherited API keys or provider variables (`ANTHROPIC_*`, `OPENAI_API_KEY`) from rerouting requests. Don't do this for API-key, Bedrock, Vertex, or proxied setups; `env -i` also strips proxy, certificate, and cloud variables. Never print or copy credentials.

Pass a requested model exactly (`-m` for Codex, `--model` for Claude and Cursor); otherwise omit the flag. An accepted flag doesn't prove access. Report an unavailable model; never substitute.

`codex login status` and `claude auth status` can say logged in while credentials are expired; the first real request decides. On an auth failure, stop that worker and give the user the login command for the same config (`CODEX_HOME=<dir> codex login`, `CLAUDE_CONFIG_DIR=<dir> claude auth login`). Don't log in for them or retry on another account.

## Bound the assignment

Give each worker the absolute directory, objective, relevant files, allowed edit paths, out-of-scope actions, and the verification command (or say none exists). Ask for changed paths, checks actually run, permission denials, and open issues.

Prompts, modes, and tool lists are not OS isolation. Codex's `-s` sandbox is OS-enforced; Claude and Cursor tool limits are not, and shell access can write files even during analysis. Repo and user config (hooks, MCP servers, worktree setup scripts) can run code at startup; inspect it in unfamiliar repos.

Read-only workers may share a folder. Concurrent editors each need their own checkout (host worktree tools, `git worktree add`, or a CLI worktree flag); record its path and base ref. A new worktree has only committed files: for uncommitted work, review the original folder read-only or hand over a reviewed diff. Never commit, stash, reset, or copy secrets to prepare a worker. Serialize edits outside Git and checks that share ports, databases, or caches.

## Launch

Run in the authorized checkout. Pass each prompt as one argument (argument array, single-quoted literal, or a private prompt file); never splice task text into shell code. Examples assume POSIX sh and flags confirmed in installed help.

Codex: `-s read-only` for analysis, `-s workspace-write` for edits (network off by default). Outside Git it needs `--skip-git-repo-check`. Close stdin (`</dev/null`); piped input is appended to the prompt.

```sh
codex exec -s read-only -C "$PWD" --json -o "$OUT/codex.txt" 'Inspect README and the manifest. Report purpose and test commands. Do not edit files.' </dev/null
codex exec review --uncommitted -c sandbox_mode=read-only --json -o "$OUT/review.txt" </dev/null  # reviews the current checkout
```

Claude: `--tools` limits built-ins, `--allowedTools` pre-approves them. For edits add `Edit` to both, `--permission-mode acceptEdits`, and `--permission-prompts none` so prompts are denied instead of hanging. Allow only the shell commands the check needs, or run the check yourself. `--strict-mcp-config` drops unlisted MCP servers, not hooks.

```sh
claude -p 'Inspect README and the manifest. Report purpose and test commands. Do not edit files or run commands.' --tools 'Read,Glob,Grep' --allowedTools 'Read,Glob,Grep' --strict-mcp-config --no-session-persistence --output-format json
```

Cursor: drop `--mode ask` to allow edits; writes and shell then follow the user's Cursor allowlist. An untrusted folder prints a trust screen instead of JSON; add `--trust` only for a repo the user trusts.

```sh
cursor-agent -p --mode ask --output-format json --workspace "$PWD" 'Inspect README and the manifest. Report purpose and test commands. Do not edit files or run commands.'
```

Never add `--dangerously-bypass-approvals-and-sandbox`, `-s danger-full-access`, `--dangerously-bypass-hook-trust`, `--dangerously-skip-permissions`, `--permission-mode bypassPermissions`, `--force`, `--yolo`, or `--approve-mcps`. Codex `--approve-for-me` and Cursor `--auto-review` are classifiers, not guarantees. If editing would need a bypass, get an analysis or patch proposal and apply it yourself.

Run independent workers in parallel only when their resources don't conflict. Track each one's process, session or thread ID, checkout, and output file.

## Collect and verify

A launch is not a result. Wait for exit. Don't restart a slow worker (that duplicates edits); if it stalls, read its output and stop it before retrying.

Check the exit code, error fields, permission denials, and the actual result. An auth or trust error can replace the JSON; that is a failure, not an empty success.

- Codex `--json` emits JSONL: look for `error` and `turn.failed`; `thread.started` carries `thread_id`; the final message is in the `-o` file. The model that ran is in the session rollout under `$CODEX_HOME/sessions` (`turn_context.model`).
- Claude and Cursor: `is_error`, `result`, `session_id`. Claude also has `permission_denials` and `modelUsage`; several `modelUsage` keys mean several models, not one resolved model.

Resume only by explicit ID with the same worker, account, checkout, and permissions (`--resume <id>`; `codex exec resume <id>` has no `-C`, so run it in the checkout, and it keeps the original sandbox), never `--last` or most-recent. Codex `--ephemeral` and Claude `--no-session-persistence` runs can't be resumed.

For edits, read the actual diff including untracked files, confirm scope, and run the checks yourself. Never overwrite user changes; remove temporary worktrees only after saving needed work.

Report per worker: CLI, account label (no emails or secrets), requested vs observed model (unverified unless metadata shows it), outcome, changed files, checks you ran vs checks the worker claims, and limitations. Say so when no check exists or a run was blocked.
