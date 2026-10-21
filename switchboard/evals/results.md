# Eval results

2026-10-18, after adding Codex and shortening SKILL.md: **9/9 scenarios, 31/31 expectations passed.**

Three subagents each read only SKILL.md and answered three scenario prompts as separate requests, without seeing expectations. The orchestrator graded their responses against [evals.json](evals.json). Nothing was launched and no files changed, so this checks decisions, not live auth, model access, or CLI compatibility.

| Case | Pass | Note |
| --- | --- | --- |
| 1. Two isolated editors, explicit models | 4/4 | Separate worktrees, uncommitted work excluded, setup shown but not run. |
| 2. Unknown account alias | 2/2 | Asked for the work config dir; no guess or fallback. |
| 3. Expired OAuth | 3/3 | Request failure wins over auth status; login command for the same config. |
| 4. Cursor needs `--force` | 3/3 | `--mode ask` patch proposal, applied and tested by the orchestrator. |
| 5. Dirty tree, no escalation | 3/3 | Reviewed diff from the original checkout; tool limits ≠ OS isolation. |
| 6. Misleading success metadata | 3/3 | Denials inspected, two models not collapsed, worker test claim unverified. |
| 7. API key behind proxy | 3/3 | Environment inherited as is; no `env -i`, no `--model`. |
| 8. Bounded Claude edit | 6/6 | `Read,Glob,Grep,Edit`, `acceptEdits`, `--permission-prompts none`, no shell; orchestrator diffs and runs pytest. |
| 9. Codex review of uncommitted work | 4/4 | `CODEX_HOME=/tmp/codex-work`, `-m gpt-5.5`, `exec review --uncommitted` read-only in the original checkout. |

## Live run

2026-10-19, macOS, codex-cli 0.163.0, Claude Code and Cursor Agent on their default accounts. Disposable Git fixture with a failing `slugify` test. Each worker got its own copy; I diffed each edit and ran `python3 -m unittest` myself.

| Check | Result |
| --- | --- |
| Codex outside Git | Refused without `--skip-git-repo-check`, exit 1. |
| Codex `-s read-only` | `touch` blocked by the OS ("Operation not permitted"). |
| Codex `exec review --uncommitted -c sandbox_mode=read-only` | Found the planted bug; rollout shows a read-only sandbox. |
| Codex `-s workspace-write` edit | One-line fix in `slug.py`; tests pass. |
| Codex bad `-m` | Exit 1 with `error` and `turn.failed` events. |
| Codex JSONL | `thread.started.thread_id`, `turn.completed`, and `item.*` events; no model name (the rollout has it). |
| Codex `exec resume <id>` | Failed from outside the checkout (it has no `-C`). From inside, it kept read-only and workspace-write. |
| Claude read-only and edit | `is_error: false`, one `modelUsage` key, edit limited to `slug.py`; tests pass. |
| Cursor in an untrusted folder | Trust screen instead of JSON, exit 1. |
| Cursor `--trust --mode ask` | JSON with `is_error`, `result`, `session_id`; no model field. |
| Cursor edit without `--mode ask` or `--force` | Edited and ran tests under the user's allowlist; took about 10 minutes. |

SKILL.md was corrected for the resume, stdin, rollout-model, and Cursor-edit findings.

To rerun: give fresh evaluators SKILL.md and the prompts only, keep expectations with the grader, and block real CLI launches.
