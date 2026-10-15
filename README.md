# vibe-workflow

A staged feature workflow for Claude Code, with a human gate at every stage boundary.

```
1 PLAN → 2 IMPLEMENT → 3 E2E → 4 REVIEW → 5 COMMIT → 6 PUSH → 7 TICKET
```

**Nothing in here is tied to a language or framework.** The skills hold the *process*; each project
supplies its own facts through `.claude/vibe-profile.md`. A repo is not Laravel because the last one
was — the workflow derives the test command, commit style and protected branches from the repo it is
actually in, and asks when it cannot.

## What's inside

| Path | What it does |
|---|---|
| `skills/vibe-coding-workflow/` | The seven stages, their owners and exit conditions |
| `skills/guided-e2e-testing/` | One step at a time, verified against the system between steps |
| `agents/group-committer.md` | Groups dirty trees into commits; shows a plan, waits for "proceed" |
| `templates/vibe-profile.md` | Per-project facts the skills read instead of guessing |

## Install

```
/plugin marketplace add <this-repo-url>
/plugin install vibe-workflow
```

Install at **user** scope so it applies to every project on the machine. On a second computer, run
the same two commands — the plugin travels as a git repo, which is the only thing that reliably
crosses machines. Nothing in `~/.claude` or `~/.claude-work` syncs by itself.

## Per project

```bash
mkdir -p .claude
cp <plugin>/templates/vibe-profile.md .claude/vibe-profile.md
```

Fill in what the project actually does and delete the rest. A missing fact gets derived or asked
for; a **stale** one gets followed silently, so leave a line out rather than leaving it wrong.

Feature notes live in `.claude/features/<branch>.md` — one branch, one file, written as you go.

## Dependencies

The workflow skill delegates to `superpowers:*` skills (brainstorming, writing-plans,
subagent-driven-development, test-driven-development, requesting-code-review,
receiving-code-review, systematic-debugging, verification-before-completion,
finishing-a-development-branch). Install superpowers too, or the stage instructions will name skills
that are not there.
