# Exercise 4 — Evaluation, Error Analysis & Tuning (Domain 4)

**Goal:** learn to diagnose why an agent failed and fix the right layer, rather than
guessing.

**You will create:** nothing — this exercise is diagnosis and judgement.

**Prerequisite:** [Exercise 3](exercise-03-memory-state-execution.md) complete.

**Time:** ~25 minutes

---

## Evaluation must be evidence-based

When someone asks "is this agent working well?", the tempting answer is an impression — the
output looked thorough, the reasoning seemed sound. That is not evaluation.

Evaluation means evidence: test results, scan findings, status checks, uploaded artifacts,
session logs. Each is independently checkable by someone who was not there. The distinction
carries forward from Exercise 1 — an agent's *plan* is not evidence that the implementation
is safe, and an agent's confident summary is not evidence that its work is correct.

Practically, this means an agentic pipeline must emit artifacts. A review that exists only
in a job log that nobody opens has not been evaluated. This is why every agent you built in
Exercise 2 has a defined output format, and why Exercise 5's pipeline uploads every report.

---

## The tuning order

When an agent misbehaves, work through the layers in this order. **Memorize it — the exam
tests both the order and the reasoning behind it.**

```
1. Prompt / task clarity
2. Instructions
3. Tool scope
4. Setup / environment
5. Repository state
6. Memory / session state
7. Model choice          ← LAST
```

The order is not arbitrary. It runs from cheapest and most likely, to most expensive and
least likely. Rewriting a prompt takes a minute and fixes a large share of failures.
Switching models is expensive, hard to evaluate, and rarely the actual problem.

**Model choice is last because it is almost never the cause.** The instinct when an agent
performs badly is to reach for a bigger model. But if the agent could not find your test
command, no model solves that — the information was not in its context. Changing models
without diagnosing first replaces a known problem with an unknown one.

A useful way to hold the order: *did I ask clearly (1), did I write it down (2), can it act
(3), can it run (4), is the repo sane (5), is context stale (6), and only then — is the
model wrong (7)?*

---

## Task — Diagnose five failures

For each, identify the root cause and the correct fix category.

### Failure 1

```
npm ci
ERR! code E401
ERR! Unable to authenticate
```

A) Tune instructions · B) Tune tools · C) Tune setup/environment · D) Tune workflow

<details>
<summary>Answer</summary>

**C — Tune setup/environment.** `E401` is an authentication failure against the package
registry. The agent's reasoning is irrelevant; its environment lacks a credential or the
registry configuration is wrong. No prompt change fixes this.

This is layer 4 in the tuning order, and it is exactly what
`copilot-setup-steps.yml` from Exercise 2 exists to prevent.

</details>

### Failure 2

```
Agent edits files in legacy/ directory despite being told not to
```

A) Tune instructions · B) Tune tools · C) Tune setup/environment · D) Add hook

<details>
<summary>Answer</summary>

**A (first) and D (enforcement).** Both, and understanding why is the point.

Instructions **guide** behaviour — they tell the agent what you want, and most of the time
that is enough. Hooks **enforce** it — they intercept the action and can deny it outright.
An instruction that keeps being violated has demonstrated that guidance alone is
insufficient for this case.

Start with the instruction, because it is cheap and explains intent. Add the hook when the
consequence justifies enforcement. You build that hook in Exercise 6.

</details>

### Failure 3

```
tool=execute command='git push origin main'
```

A) Tune instructions · B) Deny tool with hook + branch protection · C) Tune setup · D) Tune memory

<details>
<summary>Answer</summary>

**B — Deny with a hook, and enforce with branch protection.**

The critical idea: **"tell the agent to be careful" is not a control.** A control is
something that holds when the agent tries anyway. Here you want two independent layers —
a `preToolUse` hook that denies the command before it runs, and branch protection that
rejects the push even if the hook is bypassed.

Defence in depth applies to agents exactly as it does to people. The hook is preventive at
the agent boundary; branch protection is preventive at the platform boundary, and it cannot
be talked around.

</details>

### Failure 4

```
Warning: No files were found with the provided path: review.md
```

A) Set `if-no-files-found: error` · B) Check agent wrote the file before upload · C) Both A and B · D) Tune model

<details>
<summary>Answer</summary>

**C — Both.**

Two distinct problems are in play. The immediate one is that the agent did not produce
`review.md` — investigate why. The deeper one is that this failure surfaced as a *warning*,
so the workflow went green while producing nothing. A pipeline that reports success when its
output is missing is worse than one that fails, because it manufactures false confidence.

Setting `if-no-files-found: error` converts silent absence into loud failure. Apply it to
every artifact your pipeline actually depends on.

</details>

### Failure 5

```
Agent keeps asking questions, CI pipeline hangs
```

A) Add `--no-ask-user` · B) Tune instructions · C) Tune tools · D) Change model

<details>
<summary>Answer</summary>

**A — Add `--no-ask-user`.**

Asking for clarification is correct behaviour interactively and fatal in CI, where nobody
is present to answer. The agent waits until the job times out.

`--no-ask-user` forces the agent to proceed on its best judgement or fail cleanly. Use it on
every Copilot CLI invocation in a workflow — no exceptions.

</details>

---

## Artifacts as evaluation evidence

```yaml
- name: Upload Agent Report
  uses: actions/upload-artifact@v7
  with:
    name: reviewer-report
    path: reviewer-report.md
    retention-days: 7
    if-no-files-found: error
```

Three things worth noting:

**`if-no-files-found: error`** — discussed above. The default is `warn`, which hides exactly
the failure you most need to see.

**`retention-days`** — artifacts expire. If something must survive for audit, an artifact
alone is not sufficient; put the decision in a PR comment, which persists with the
repository.

**`if: always()`** — for test results, upload even when a previous step failed. A failing
test run is the case where you most want the output, and without this the step is skipped.

### Who deleted an artifact?

Artifact deletion is recorded in the **organization audit log** as an `artifact.destroy`
event with an actor field. Not the PR timeline, not the workflow log. This is a recurring
exam question, and the reasoning is that audit logs record platform-level actions across
the organization, while workflow logs only record what happened inside one run.

---

## Exam notes

### Tuning order

```
1. Prompt/task clarity
2. Instructions
3. Tool scope
4. Setup/environment
5. Repository state
6. Memory/session state
7. Model choice (LAST)
```

### Diagnostic shortcuts

| Symptom | Layer |
|---------|-------|
| Auth error, missing command, missing dependency | Setup/environment |
| Agent ignores a documented rule | Instructions, then hook |
| Agent performs a forbidden action | Tools/hooks — not instructions alone |
| CI job hangs | Missing `--no-ask-user` |
| Missing artifact reported as a warning | `if-no-files-found: error` |

### Guidance versus enforcement

- Instructions **guide** — not enforceable.
- Hooks **intercept** — can deny.
- Branch protection **enforces** — cannot be bypassed.

High-risk actions get all three.

### Audit facts

- `artifact.destroy` in the organization audit log identifies who deleted an artifact.
- Session logs and the PR timeline show what an agent did.
- Artifacts expire; PR comments do not.

---

## Common pitfalls

**Reaching for a different model first.** It is step 7. Steps 1–6 are cheaper and far more
often correct.

**Treating instructions as a security control.** They are guidance. If it must not happen,
enforce it.

**Leaving `if-no-files-found` at its default.** Silent success on missing output.

**Omitting `--no-ask-user` in CI.** Guaranteed hang whenever the agent wants to clarify.

**Judging agent quality by how good the output reads.** Evaluate on evidence — tests,
scans, checks — not on prose quality.

---

## What you learned

A repeatable diagnostic procedure and the vocabulary the exam uses for it. The distinction
between guidance and enforcement carries directly into Exercise 6, where you build the
enforcement layer.

**Next:** [Exercise 5 — Multi-Agent Orchestration](exercise-05-multi-agent-orchestration.md)
