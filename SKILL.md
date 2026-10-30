---
name: codex-orchestration
description: >-
  Runs an approved plan from research to a merged, clean repository. The agent that runs this skill
  is the lead and does not read or write code. Codex workers do the code work: a lighter model
  reads and searches, a stronger model writes the code. A fresh reviewer checks the result, and the
  lead commits and merges into dev (or main) and removes the temporary branch and worktree.
  If Codex hits its usage limit during the run, the work continues with Claude subagents
  (Haiku for research, Sonnet for code).
  Use when the user asks for codex-orchestration, or asks to run a plan through Codex workers.
license: MIT
compatibility: >-
  Needs an agent that can run shell commands and load SKILL.md skills (Claude Code, Codex CLI,
  OpenCode, Hermes Agent, and others), plus git, Node 18+, and the Codex CLI (`codex`),
  installed and authenticated. Requires the codex-delegate skill.
metadata:
  version: 0.2.0
---

## Goal

The lead's tokens usually cost the most. Codex workers read and write the code. The lead owns the
plan, verifies the results, and lands the work. If no rule fits a case, keep the code work with the
workers and the decisions with the lead. Count retries and the lead's own turns in the total cost.

## Requirements

Check once, before the first run. If one item is missing, stop and tell the user how to install it.

1. `codex --version` works and `codex login` is done.
2. The `codex-delegate` skill is installed. Find its folder with
   `find ~ -name relay.mjs -path '*codex-delegate*'`; the folder that holds `scripts/relay.mjs` is
   `<delegate-dir>`.
3. Recommended: the `loop-code-review` skill. Without it, use the fallback in step 6.
4. Optional: the `asd-ste100-ru` skill, for Russian reports.

## Input

An approved plan: a file path or the plan text. Writing the idea, the spec, and the plan is the job of
other skills and is not part of this skill.
For a small task (normal risk, up to 3 files, one goal, clear checks), the lead writes a plan of
3-5 lines and goes on.

## Settings

Defaults. The user can change any value; the first source that has a value wins:
1. the request text ("implementer effort medium");
2. `.codex-orchestration.json` in the repository root, then `~/.config/codex-orchestration/config.json`;
3. the defaults below.

| Role | Model | Effort |
|---|---|---|
| Researcher (read-only) | `gpt-6-luna` | `xhigh` |
| Implementer | `gpt-6.1-sol` | `high` |
| Implementer's subagents | `gpt-6-luna` | the Researcher's effort |
| Fallback Researcher (Codex limit) | Claude `haiku` (Haiku 5.5) | the agent's default |
| Fallback Implementer (Codex limit) | Claude `sonnet` (Sonnet 5.5) | the agent's default |

File format (see `examples/codex-orchestration.json`):
```json
{ "researcher":  { "model": "gpt-6-luna",  "effort": "medium" },
  "implementer": { "model": "gpt-6.1-sol", "effort": "medium" },
  "fallback": { "researcher": { "model": "haiku" }, "implementer": { "model": "sonnet" } } }
```
Set `"fallback": false` to turn the fallback off (see "Codex limit fallback").

The default models match the Codex model list at the time of writing. If a model is not on the
user's list, the user sets the roles in the file or in the request.

Before the first run, check that each model and effort exists: `codex debug models`.
If one does not exist, stop and tell the user. **Never change the model or the effort of the
Researcher and the Implementer by yourself, not on a failure and not to save cost.** The only
exception is the Codex limit fallback below, and it changes the provider, not the Codex settings.
The reviewer follows its own skill's settings; do not limit them.

## Codex limit fallback

When Codex stops because of a usage limit, the run does not stop. The remaining work goes to Claude
subagents.

**Detect.** A relay run ends with `status: failed` (or `completed` with no work done), and
`stderrTail`, `finalMessage`, or the events log say that a usage, rate, or credit limit is reached
(for example "hit your usage limit", "rate limit", "quota"). Other failures (`timeout`, `aborted`,
`codex_unavailable`, an auth error, a crash) are not a limit: handle them as usual.

**Switch.**

| Role | Codex | Fallback |
|---|---|---|
| Researcher | `gpt-6-luna` | Claude `haiku` (Haiku 5.5, read-only) |
| Implementer | `gpt-6.1-sol` | Claude `sonnet` (Sonnet 5.5) |

- Switch once. After the first limit error, send all later worker tasks of this run to the Claude
  workers: research, remaining subtasks, and fix rounds. Do not try Codex again in this run. A new
  run starts with Codex again.
- Start each fallback worker as a fresh subagent in the same repository or worktree, with the
  subagent tool of your agent. In Claude Code: the Agent tool, `subagent_type: general-purpose`,
  `model: haiku` or `model: sonnet`. Use the model aliases; they resolve to `claude-haiku-5-5` and
  `claude-sonnet-5-5`. Set no effort unless the user named one.
- A Codex session cannot be resumed by Claude. Hand over the work in the new brief: the plan, the
  brief rules from the `codex-delegate` skill, and what is already done (`touchedFiles`, the diff
  stat, the last `finalMessage`, the subtask that stopped). Tell the worker to continue from the
  current tree and not to restart or revert existing changes.
- The same limits apply as for Codex workers: do not commit, do not push, do not touch the baseline
  files. The lead still does not edit code, and still runs step 5 on every result.
- The reviewer is not affected. If the reviewer was a Codex run, use a Claude reviewer with a fresh
  context.
- Say it in one line when you switch: "Codex limit reached, continuing with Claude Haiku and Sonnet
  subagents." Add the reset time if the message gives one.
- **Stop and report** (state of `git status`, what is done, what is not) if: `"fallback": false`; your
  agent cannot start Claude subagents; or the Claude worker cannot start or is also limited. Never do
  the code work yourself.

## Rules

- You are the lead: the agent that runs this skill. Do not explore or edit code. To verify a claim,
  read only the code that it cites. For more, ask a worker.
- Dispatch workers with `node <delegate-dir>/scripts/relay.mjs`. Follow the brief rules of the
  `codex-delegate` skill. Run every dispatch in the background with the means of your agent, and
  continue when it ends. Add `--timeout 2h` to implementation runs. Pass `--model`, `--effort`,
  `--subagent-model`, `--subagent-effort` explicitly on every run and on every resume.
- A worker sees only its brief. The brief has the task context: the repository path, scope,
  constraints, the plan, the exact check commands from the project rules (`AGENTS.md`, `CLAUDE.md`,
  `Taskfile`, `Makefile`), and "do not commit". Write to workers in English and to the user in the
  user's language.
- Follow project rules. Never add secrets or local files (`.env`, `config.json`, `.local/`, logs,
  databases).
- Do not push without the user's permission. Do not use force. Do not touch branches and worktrees
  that this run did not create.

## Steps

1. **Baseline.** Record `git status --short --untracked-files=all` and `git branch --list dev`.
   Files that are dirty now are not task files: never stage them.
2. **Workspace.** Small task and a clean tree: work in the current repository, no worktree.
   Otherwise create one branch `feat/<slug>` and one worktree `../<repo>-<slug>` for the whole
   feature (not one per task). Run all workers there with `--cd`.
3. **Research.** Skip for a small task. Send a Researcher with `--read-only`. Ask for the code
   paths, data model, reusable APIs, affected callers, checks, and unknowns that can change the
   plan. If the evidence changes the plan, update the plan.
4. **Implement.** Send the plan to the Implementer. Tell it to delegate searching and reading to its
   subagents. Send one subtask at a time; accept each against the plan. Return incomplete work in
   the same session (`--session <threadId>` from `result.json`).
5. **Verify.** Do not trust the worker's report. Check `touchedFiles` against the plan, read the
   diff stat, and run the project's check commands yourself.
6. **Review.** Run `loop-code-review` on all task changes. Without it, start a reviewer with a fresh
   context (a subagent of your agent, or a new read-only Codex run) and give it the diff, the
   project rules, and the check commands, but not the plan. Send accepted findings to the
   Implementer in the same session, then verify and review again. Repeat until the checks pass and
   the review has no accepted findings. There is no round limit. Do not change the Implementer's
   model or effort between rounds.
7. **Accept.** Compare the result with the plan's definition of done. If the scope or risk grew,
   stop and ask the user.
8. **Commit.** Stage only the task files by name. One commit per subtask or one for the task, in
   English, with a short area prefix. No dirty task files remain. Baseline files stay as they were.
9. **Merge.** Merge into `dev` if it exists, otherwise into `main`: `git merge --no-ff feat/<slug>`.
   On a conflict, stop and report; do not resolve it by yourself. If you worked in the current
   branch of a small task, merge only if that branch is not the target.
10. **Clean up.** After a successful merge, run `git worktree remove <path>` and
    `git branch -d feat/<slug>`. Use `-d`, never `-D`. Check that `git worktree list` and
    `git branch` show no leftovers from this run. If the user rejects the work, ask before you
    delete anything.

## Finish

Report: the result, the checks that ran, the commits, the merge target, what was cleaned up, the
baseline files that were left untouched, unresolved issues, and the cost (workers, rounds, models,
efforts). If the Codex limit fallback ran, say which subtasks Codex did and which Claude did. State that a push was not done unless the user allowed it.
If the user's language is Russian and the `asd-ste100-ru` skill is installed, write the report with
it. Otherwise write short, plain sentences in the user's language.
