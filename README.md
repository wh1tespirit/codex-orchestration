# codex-orchestration

An agent skill that turns your main AI agent into a **lead** and sends the code work to cheaper **Codex workers**. You give it an approved plan. It researches, implements, verifies, reviews, commits, merges into `dev` (or `main`), and removes the temporary branch and worktree. You get a clean repository and a short report.

The skill follows the open [Agent Skills](https://agentskills.io/) format (`SKILL.md`). It is written for Claude Code, Codex CLI, OpenCode, Hermes Agent, and any other agent that loads `SKILL.md` skills and can run shell commands.

> **Status:** version 0.2.0. The skill text is complete, but it has not yet been run end to end on a real project, and it has not been tested in every agent listed above. Treat the first run as a trial and read the report.

## What it does

```
approved plan
  → baseline (git status, dev branch?)
  → Researcher: reads and searches the code (read-only)
  → Implementer: writes the code, delegates small lookups to its own subagents
  → lead verifies: touched files, diff, project checks
  → fresh reviewer checks the change (fix and re-review until clean)
  → lead accepts against the plan's definition of done
  → lead commits the task files
  → merge --no-ff into dev (or main), remove worktree and branch
  → report
```

| Role | Default model | Default effort | Job |
|---|---|---|---|
| Lead | your current agent | — | Plans assignments, verifies, commits, merges. Does not read or write code. |
| Researcher | `gpt-6-luna` | `xhigh` | Finds files and code paths. Read-only. |
| Implementer | `gpt-6.1-sol` | `high` | Writes the code. Hands lookups to subagents. |
| Reviewer | set by your review skill | — | Checks the change without seeing the plan. |
| Fallback Researcher | Claude `haiku` (Haiku 5.5) | agent default | Takes over research if Codex hits its usage limit. |
| Fallback Implementer | Claude `sonnet` (Sonnet 5.5) | agent default | Takes over coding if Codex hits its usage limit. |

### If Codex runs out of limits

A Codex usage limit does not stop the run. The skill detects the limit error from the relay result and sends the rest of the work to Claude subagents: Haiku reads and searches, Sonnet writes the code. The switch happens once per run, the new worker continues from the current working tree (it does not start over), and the same rules hold: no commits by workers, the lead re-verifies every result, and the reviewer is unchanged. The report says which part Codex did and which part Claude did. If your agent cannot start Claude subagents, or you set `"fallback": false`, the skill stops and reports the state instead.

## Why use it

- **Lower cost.** The most expensive model decides and checks. It does not spend tokens on searching and typing code.
- **Independent checks.** The lead never trusts a worker's own report. It re-runs the project checks and reads the diff. A reviewer with fresh context looks at the change without the plan.
- **A clean repository.** One branch and one worktree per feature, not per task. After the merge, both are removed with the safe `git branch -d`. Files that were already dirty before the run are never staged.
- **Predictable behavior.** The skill never changes a worker's model or effort by itself. Fix rounds repeat until the checks and the review pass, with no round limit.
- **You control the money.** You set the model and effort of each role in a request or in a small JSON file.

## Requirements

- An agent that loads `SKILL.md` skills and can run shell commands.
- `git` and Node.js 18 or newer.
- The [Codex CLI](https://github.com/openai/codex): `npm i -g @openai/codex`, then `codex login`.
- The [`codex-delegate`](https://github.com/amElnagdy/delegate-skills) skill (required). It provides the relay that runs Codex and writes a structured result.
- The [`loop-code-review`](https://github.com/di-sukharev/loop-code-review-skill) skill (recommended). Without it, the skill starts a plain fresh-context reviewer.
- The [`asd-ste100-ru`](https://github.com/wh1tespirit/asd-ste100-ru) skill (optional). It makes Russian reports short and unambiguous.

## Install

The easiest way is the [`skills`](https://skills.sh/) CLI. It installs into the skill folders of many agents at once.

```bash
# 1. Codex CLI (skip if you have it)
npm i -g @openai/codex && codex login

# 2. Required dependency
npx skills add amelnagdy/delegate-skills@codex-delegate -g -y

# 3. Recommended dependency
npx skills add di-sukharev/loop-code-review-skill -g -y

# 4. This skill
npx skills add wh1tespirit/codex-orchestration -g -y

# 5. Optional: Russian reports
npx skills add wh1tespirit/asd-ste100-ru -g -y
```

Manual install (any agent): clone this repository into your agent's skills folder, for example `~/.claude/skills/codex-orchestration`, `~/.agents/skills/codex-orchestration`, or the folder your agent documents. The folder name must stay `codex-orchestration`.

Check: start your agent and ask "which skills do you have?" The list must include `codex-orchestration` and `codex-delegate`.

### Install with a prompt

Copy the text below and paste it into your AI agent. The agent installs everything and checks the result.

````text
Install the "codex-orchestration" agent skill and its dependencies on this machine. Work step by step and do not skip checks.

1. Check the tools. Run `git --version`, `node --version` (must be 18 or newer), and `codex --version`.
   - If `codex` is missing, run `npm i -g @openai/codex`. Then tell me to run `codex login` myself (I must sign in), and wait until I confirm.
2. Install the skills with the skills CLI. Use global mode:
   - `npx skills add amelnagdy/delegate-skills@codex-delegate -g -y`  (required)
   - `npx skills add di-sukharev/loop-code-review-skill -g -y`  (recommended)
   - `npx skills add wh1tespirit/codex-orchestration -g -y`  (the skill itself)
   - Optional, only if I write in Russian: `npx skills add wh1tespirit/asd-ste100-ru -g -y`
   If `npx skills` fails or does not support your agent, clone the repositories from GitHub into your agent's skills folder instead (the folder name must equal the skill name), using https://github.com/amElnagdy/delegate-skills, https://github.com/di-sukharev/loop-code-review-skill, https://github.com/wh1tespirit/codex-orchestration, and https://github.com/wh1tespirit/asd-ste100-ru.
3. Check the models. Run `codex debug models` and compare the list with the defaults in the codex-orchestration SKILL.md (`gpt-6-luna` for the researcher, `gpt-6.1-sol` for the implementer). If a default is not on my list, ask me which model to use for each role, then create `~/.config/codex-orchestration/config.json` using the format in the skill's `examples/codex-orchestration.json`.
4. Check the result. Confirm that the skills `codex-orchestration` and `codex-delegate` are visible to you (restart or reload skills if your agent needs it). Find `scripts/relay.mjs` inside the codex-delegate folder and run `node <that path> --help` to prove that it works.
5. Do not run any task and do not change any project files. Report what you installed, where each skill is, which checks passed, and what I must still do by hand.
````

## Use

Ask your agent, in your own words, to run a plan with codex-orchestration:

```text
Use codex-orchestration for the plan in docs/plans/add-export.md.
```

```text
Run this through codex-orchestration with implementer effort medium: add a --dry-run flag to the CLI.
```

The input is an approved plan (a file path or text). For a small task with a clean tree, a plan of a few lines is enough, and the skill skips the research step and the worktree.

## Settings

Order of priority: the request text, then `.codex-orchestration.json` in the repository root, then `~/.config/codex-orchestration/config.json`, then the defaults.

```json
{
  "researcher":  { "model": "gpt-6-luna",  "effort": "medium" },
  "implementer": { "model": "gpt-6.1-sol", "effort": "medium" },
  "fallback": {
    "researcher":  { "model": "haiku" },
    "implementer": { "model": "sonnet" }
  }
}
```

`fallback` names the Claude models used when Codex hits its usage limit. Use `"fallback": false` to turn it off.

Lower the effort to spend less on a small subscription, or raise it for hard tasks. The skill checks the values with `codex debug models` before it starts. If a model or an effort is not available, it stops and asks you. It never swaps them silently.

## Rules the skill follows

- The lead does not read or edit code.
- Workers never commit. The lead commits, and stages only the task files by name.
- No secrets or local files (`.env`, `.local/`, logs, databases) are added.
- No `git push` without your explicit permission, and never with force.
- Merge conflicts stop the run. The skill does not resolve them by itself.
- Only branches and worktrees created by the run are removed, with `git branch -d`.

## Limits

- Codex CLI is required for the workers. The lead can be any agent, but Codex workers use your OpenAI account and its limits.
- If Codex itself is the lead, background waiting needs the `codex-background` helper from the `codex-delegate` author. This path is untested here.
- The Codex limit fallback needs an agent that can start Claude subagents (for example Claude Code). Other agents stop and report instead. Limit detection reads the relay's error text and has not been tested against every wording of Codex limit messages.
- The default model names depend on your Codex model list. Set your own names if they differ.
- Dependencies are third-party skills. Review them before you install, as you would any code that runs with your agent's permissions.

## License

MIT. See [LICENSE](LICENSE).
