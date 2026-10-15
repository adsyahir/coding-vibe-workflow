---
name: vibe-coding-workflow
description: Use when starting a feature, fixing a reported bug, picking up a branch, or asking "what's next" on in-flight work. Also use when the user names a stage — plan, implement, test the flow, review, commit, push, ticket — or asks which model or agent to use for a stage.
---

# Vibe Coding Workflow

Seven stages, each with an owner and an exit condition. The pipeline exists so no stage silently
absorbs another's job — implementation does not review itself, and review does not discover what
manual testing should have.

```
1 PLAN       strong model     → written plan
2 IMPLEMENT  cheaper subagents→ code + tests green
3 E2E        the human        → real flow verified
4 REVIEW     fresh eyes       → findings resolved
5 COMMIT     group-committer  → grouped commits, approved
6 PUSH       you + human      → branch pushed, PR open
7 TICKET     tracker skill    → ticket logged (ask first)
```

**Why E2E sits before review.** E2E is what changes the diff — schema, config, build steps and
cross-package wiring fail there, and each failure is a code edit. Reviewing first means reviewing a
diff that is about to move, then reviewing again. Review reads the *final* diff, with the E2E bug log
as its input. The exception is a one- or two-file bugfix where E2E is a single click: there the diff
is small enough that review first costs nothing.

**You may not advance a stage on your own authority.** The human moves the pipeline. Your job is to
say which stage the work is in, and what its exit condition still needs.

---

## Step 0 — load the project profile

**This skill knows the process. It does not know the project.** Before the first stage, read the
project's own facts, in this order, and stop at the first that exists:

1. `.claude/vibe-profile.md`
2. `CLAUDE.md` / `AGENTS.md` in the repo root
3. Nothing — then **derive** what you can (below) and **ask** for the rest

What you need, and how to derive it when it is not written down:

| Fact | Derive it with | Ask if |
|---|---|---|
| Commit message style | `git log --oneline -15` | the log is empty or inconsistent |
| Protected branches | `git branch -r`, repo settings, the profile | more than one long-lived branch exists |
| Test command | `package.json` scripts · `composer.json` scripts · `Makefile` · `pyproject.toml` · `go.mod` · `Cargo.toml` · CI config | nothing is discoverable |
| Lint / format command | same files | — |
| How schema or config changes are applied | migration dir, `docker-compose`, infra files | the feature touches persistence |
| Issue tracker | the profile, existing branch names, PR template | stage 7 is reached and none is declared |
| Where feature notes live | `.claude/features/` by default | — |

Never assume a framework. A repo is not Laravel because the last one was. Look.

If a fact matters for the current stage and you cannot derive it, **ask one question** — do not guess
and do not stall the other work waiting for the answer.

---

## Bugfix path (stage 0)

A reported bug does not get a unit breakdown. It gets a **diagnosis first**, then joins the pipeline
at stage 2.

A screenshot of an error message is a symptom, not a diagnosis. Before changing any code, reproduce
it against the real thing and read the actual status and body. For an HTTP API:

```bash
curl -sk -X <METHOD> '<url>' -H 'Accept: application/json' \
  -H 'Authorization: <auth>' -d '<the real payload>' -w '\nHTTP:%{http_code}\n'
```

For a CLI, run the command with the real arguments and keep the exit code. For a UI, open the
network tab and read the response body, not the toast. Then check the server or application log at
that timestamp.

A generic message ("Something went wrong. Please try again.") usually hides a 500 or a blocked
request — the error handler swallowed everything but a display string. Find which before theorising.

**Exit:** you can state the failing line and why it fails. Only then write the fix.

---

## Stage 1 — Plan

Use **superpowers:brainstorming** first if the shape is unsettled, then **superpowers:writing-plans**.
For a plan you will execute in-session, the `Plan` agent is available.

**Exit:** a written plan naming the files to touch and the risks. No code yet.

Record the plan in `.claude/features/<branch>.md` — that file is the branch's memory and every later
stage reads it. One branch, one file.

## Stage 2 — Implement (delegated to subagents)

**Default: superpowers:subagent-driven-development.** You are the controller, not the typist. One
fresh implementer subagent per task from the plan, a task review after each, a broad review at the
end. A subagent never inherits this session's history — you construct exactly the context it needs,
which is what keeps it focused and your own context free for coordination.

Pick the execution skill by the shape of the work:

| Situation | Skill |
|---|---|
| Plan with mostly independent tasks (the normal case) | **superpowers:subagent-driven-development** |
| Human asked for inline, or tasks are tightly coupled | **superpowers:executing-plans** |
| 2+ genuinely independent tasks, no shared state | **superpowers:dispatching-parallel-agents** |

**Models:** the controller runs on a strong model — planning, task briefs, adjudicating review
findings. Implementer subagents run on a cheaper one (`Agent` with `model:`). Escalate a stuck task
on fix round 4+, as the skill prescribes. The controller writes code itself only for trivial edits
where briefing a subagent costs more than the edit.

Inside each subagent: **superpowers:test-driven-development** for anything with behaviour. If the
ecosystem has a cleanup or simplifier skill available, run it once the code works — check what is
actually installed rather than naming one from memory.

### Two defaults this pipeline overrides

`subagent-driven-development` is written for an autonomous run. Say so when you deviate rather than
silently following either default:

- **Per-task commits → batched.** The skill has implementers commit per task. Here they do not commit
  at all: work accumulates and stage 5 runs one `group-committer` pass. Put this in every implementer
  brief, because the skill's own prompt tells them to commit.
- **"Never pause between tasks" → the human's gates still hold.** Continuous execution applies
  *within* stage 2, not across the pipeline. You still stop at the stage boundary. The skill's four
  stop conditions (irreversible op, security-sensitive action, side effect outside the worktree,
  unworkable plan) apply on top, not instead.

Its ledger discipline does apply: track progress in a file, not only in todos. A controller that
loses its place after compaction re-dispatches completed tasks, which is the most expensive failure
this skill exists to prevent.

**Exit:** the suite is green *and you ran it*. Not "should pass" —
**superpowers:verification-before-completion** means you paste the output.

Append to the feature file as you go: decisions, deviations, anything deliberately skipped.
Future-you has no memory of this session.

## Stage 3 — End-to-end (the human drives)

**REQUIRED SUB-SKILL: Use guided-e2e-testing.**

This is the stage that cannot be delegated to a model: a green suite proves the units work, not that
the feature does. Schema changes, seed data, config, environment and cross-package wiring all fail
here first.

One step at a time, each citing `file:line`. Every bug found gets logged — that log is the input to
stages 4 and 5.

**Before step 1, verify the environment matches what the code assumes.** Drift is the most expensive
failure here because it surfaces as a generic error, not a stack trace. Check the things the feature
depends on actually exist, in whatever way the stack allows — a schema introspection query, a config
dump, a health endpoint, `--dry-run`.

Two drift patterns worth checking in any stack:

- **A migration or setup step that the default command does not pick up** — a non-default directory,
  a separate service, a manual step in a runbook. Confirm it ran; do not assume the default command
  covered it.
- **A secondary store that drifts independently of the primary** — an audit database, a read replica,
  a search index, a cache schema. A change guarded by an existence check is silently skipped there,
  and every write then throws at runtime.

**Exit:** the human has walked the real flow and says it works.

## Stage 4 — Code review

**superpowers:requesting-code-review**, or a `code-reviewer` agent, or the host's review command.

Receiving findings: **superpowers:receiving-code-review**. Verify each one before implementing it —
reviewers are wrong sometimes, and agreeing with a wrong finding costs more than disputing it.

**Scope lock.** When the human has said "fix X only, don't touch other code", that constraint binds
this stage too. State it in the review prompt so the reviewer does not report pre-existing problems
in the surrounding file, and park anything it finds anyway in the feature file under a `Parked`
heading — do not fix it. A review finding is not permission to widen the diff; only the human is.

**Exit:** every finding fixed, disputed with reasoning, or explicitly deferred by the human.

## Stage 5 — Group commit

Use the **group-committer** agent. It groups by repo and by logical boundary, shows a plan, and waits
for explicit approval. It never commits unprompted.

Read `git log --oneline -15` for the convention before writing any message. Mirror what is there —
prefix, capitalisation, tense, whether bodies are used. Do not import a convention from another
project.

**Exit:** commits exist, the human approved the grouping, nothing is on a protected branch.

## Stage 6 — Push

Push the **feature branch** and open a PR. **superpowers:finishing-a-development-branch** covers the
integration decision.

**Protected branches are PR-only.** Whatever the profile lists — and `main`/`master` always — is
never written to directly: not by merge, not by fast-forward, not "just this once because it's one
line". "Push to <protected>" from the human means *open the PR against it*; if they truly want a
direct push, make them say so after you have named it as a direct push.

**A diverged target needs its own branch.** Before proposing a PR into a second target, measure it:

```bash
git rev-list --left-right --count origin/<target>...origin/<source>
```

If the target is behind the source's base, merging the feature branch drags every unmerged commit
across with it. Cut a branch **from the target**, cherry-pick onto it, PR that. Check the touched
files are identical across both lines first (`git diff origin/A origin/B -- <paths>`); empty output
means the cherry-pick is conflict-free.

Undoing a bad push to a protected branch needs a force-push, which is denied by default and has to go
back to the human. That asymmetry is the whole reason for the rule.

**Exit:** branch pushed, PR open against every target, the feature file's Resolution section filled
in with the real SHAs and PR numbers.

## Stage 7 — Ticket (ask first, always)

After the PRs are open, log the work in whatever tracker the profile declares — Jira, Linear, GitHub
Issues, Aztra, none.

**This stage never runs on your own initiative.** Ask the human whether to create the ticket and wait
for an explicit yes — even when they have approved every earlier stage, even when the work is
obviously ticket-worthy. Approval of stage 6 is not approval of stage 7.

When they say yes, batch the remaining choices (project, assignee, parent, status, time) into **one**
prompt rather than a series. Fill the body from the feature file: problem, root cause, the PR links,
and the deploy steps a reviewer would otherwise have to rediscover.

**Exit:** ticket created and its URL reported, or the human declined.

---

## Which stage am I in?

| Signal | Stage |
|---|---|
| No plan file, or the plan is a conversation | 1 |
| Plan exists, tests red or unwritten | 2 |
| Tests green, never exercised for real | 3 |
| Flow verified, nobody has read the diff | 4 |
| Review clean, `git status` dirty | 5 |
| Committed, not pushed | 6 |
| Pushed, PRs open, no ticket | 7 — ask, do not create |

## Rules that hold across every stage

- **Never commit on a protected branch.** Branch first. And never push to one — PR only (stage 6).
- **Scope is the human's to widen, not yours.** "Fix X only" survives contact with a review finding,
  a tempting refactor, and an obvious adjacent bug. Park them; do not fix them.
- **Changes stay inside the feature's own module** unless the human approves crossing out. A shared
  table, another team's service, a common utility — that is a decision, not a detail.
- **The feature file is updated as you go**, not reconstructed at the end.
- **Say which stage you are in** when the human asks "what's next" — then give that stage's next
  single action.

## Common mistakes

- **Assuming the stack.** This skill is used across projects. Derive the test command, the commit
  style and the protected branches from *this* repo, every time.
- **Skipping stage 3** because the suite is green. The suite does not run against the real
  environment, exercise setup steps, or touch the UI.
- **Reviewing your own implementation in the same breath.** Separate stage, fresh eyes, ideally a
  different model.
- **Writing the code yourself in stage 2** because briefing a subagent "feels slower". It is slower
  once, and cheaper every turn after — your context stays free for coordination.
- **Letting an implementer subagent commit.** The skill's prompt tells it to; this pipeline batches
  at stage 5. Override it in the brief.
- **Committing mid-review** so the diff "looks tidy". Review the working tree.
- **Advancing a stage because it feels done.** Exit conditions are observable; feelings are not.
- **Reading "push to <protected>" as a direct push.** It means open the PR.
- **Treating a review finding as a mandate.** Outside the agreed scope it is a note for the feature
  file, not an edit.
- **Changing code before reproducing the bug.** The displayed error text is rarely the real error.
