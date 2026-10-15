---
name: guided-e2e-testing
description: Use when the user will drive a real flow through the real application and wants to be guided — "walk me through testing this", "let's test the flow", "guide me". Emits one step at a time and verifies each against the system before continuing.
---

# Guided End-to-End Testing

The user drives a real flow through the real app. You emit **one step, wait, verify, emit the next**.

**Core principle:** a step the user has not confirmed is a step you do not know happened. You are a
debugger with a human as the execution engine — not a document generator.

## The Iron Rule

**ONE step per message. Then stop and wait.**

Not two. Not "step 3 and while you're there, 4". Not a numbered list with a note saying "do these in
order". One.

The user cannot hold a wall of steps in their head while clicking through an interface. Every step
they do out of order produces a failure you will then misdiagnose.

### Red flags — you are about to violate this

- "Let me lay out the full sequence first so they see the shape"
- "These two are trivial, I'll combine them"
- "A checklist is more useful than drip-feeding"
- "They can just follow the list themselves"
- "I'll give all the setup steps, then go one at a time for the real test"

**All of these mean: emit ONE step. Stop typing.**

| Rationalization | Reality |
|---|---|
| "Context helps them plan" | They asked to be guided, not briefed. Overview = they skip ahead and break ordering. |
| "Combining saves round-trips" | A merged step that fails is two unknowns. You lose more time diagnosing. |
| "It's just the prerequisites" | Prerequisites are where ordering constraints live. Those need confirming most. |
| "I already listed them earlier" | Then reference step N by name. Do not re-paste the list. |

## Before step 1: build the order, don't publish it

Work out the full dependency chain privately. Then emit only the first step.

Find the real constraints in code, not from memory. They look different in every stack, but the
categories repeat:

- **Time or state windows** that gate the flow — an action allowed only before a cutoff, only in one
  status, only once
- **Required fields with no default**, which fail only at the write
- **Reference or seed data** the environment needs that a fresh install lacks
- **Configuration** a given environment does not set — a key, a feature flag, a service URL
- **External dependencies** that must be up, or stubbed

A constraint that forces a step *earlier* in the order is the single highest-value thing to find
before you start. Getting it wrong strands the user mid-flow with data they cannot undo.

## Step format — every slot REQUIRED

```
# Step N — <imperative, one outcome>

<The action. A URL, a form field, a command. Concrete values, not placeholders.>

**Files:** `path/to/file:123` — <what that line does for this step>

**Expect:** <the observable result, so they know if it worked>

<One line on any trap in this step, only if real.>

Tell me when it's done.
```

**Files is not optional.** It is the difference between a test script and a debugging session: when
step 6 misbehaves the user already knows which file to open, and so do you.

### Citing lines accurately

Line numbers drift. **Look them up now; never recall them.**

```bash
grep -n 'functionName\|field_name' path/to/file
```

Cite the line that *implements the behaviour this step exercises* — the guard, the query, the write.
Not the class declaration. If a step spans files, cite each with its own role.

## Between steps: verify, don't assume

When the user says "done", confirm it against the system before moving on. Use whatever the stack
gives you — a database query, an API read, a log line, a file on disk, a queue depth.

A confirmed "done" plus an unchanged system means the step silently failed. Catching that at step 2
costs a minute; catching it at step 9 costs the whole session.

If verification contradicts the user, say so plainly and show the output. Do not proceed.

## When a step fails

Switch to **superpowers:systematic-debugging**. Do not guess, do not patch forward, and do not skip
to the next step. Fix the cause, re-run that same step, then continue.

Log each real bug as you go — file, line, cause, fix. That list is the input to code review and to
the commit message, and you will not reconstruct it later.

## Quick reference

| Situation | Do |
|---|---|
| User asks "what do I need to test" | Build the order privately, emit step 1 only |
| User says "done" | Verify against the system, then emit the next step |
| Step fails | systematic-debugging, fix, re-run the same step |
| Prerequisite is missing | Insert it as the next step; say why it moved |
| User asks for the whole list | Give the phase names only, then continue one at a time |
| Flow needs data that does not exist | Say what creates it and who creates it |

## Common mistakes

- **Publishing the plan.** Build the order, emit one step. The plan is yours, not theirs.
- **Citing stale lines.** Always `grep -n` before writing a citation.
- **Skipping verification** because the user sounded confident.
- **Fixing forward.** A failing step means stop, not work around.
- **Silent environment edits.** If you change their data or config to unblock a step, say exactly
  what you changed and offer to revert it.
