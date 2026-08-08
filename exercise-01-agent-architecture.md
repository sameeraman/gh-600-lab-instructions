# Exercise 1 — Prepare Agent Architecture (Domain 1)

**Goal:** give every agent that touches this repository a shared, written understanding
of how the codebase works and what it is not allowed to do.

**You will create:** `.github/copilot-instructions.md`

**Time:** ~15 minutes

---

## Why this exercise comes first

An agent starts each task with no memory of your project. It does not know that your
controllers are supposed to stay thin, that tests run through a specific `.csproj`, or
that user IDs must never be read from a request body. Left to infer, it will guess —
reasonably, but inconsistently, and differently on every run.

Repository instructions solve this by moving that knowledge out of your head and into a
file the agent reads automatically. This is *context engineering*: the quality of an
agent's output is bounded by the quality of the context it receives, so context becomes
something you design deliberately rather than something that happens to you.

There is a second, less obvious benefit. A written instruction file is reviewable. When
an agent misbehaves, you can point at the line that misled it and fix the line. Behaviour
that lives only in prompts typed into a chat window cannot be audited, versioned, or
code-reviewed.

---

## The architecture principle behind Domain 1

Agents work best when their work is **legible** — when a human can look at the output and
judge it quickly. GitHub already provides a chain of accountability built for exactly
this:

```
issue → branch → pull request → status checks → review → merge
```

The design rule for the whole lab is: **put agent work inside that chain, never beside
it.** An agent that opens a pull request is safe to run aggressively, because the PR is a
checkpoint a human controls. An agent that pushes directly to `main` has escaped the
chain, and no amount of prompt engineering compensates for that.

### Autonomy levels

Every agent you build gets a deliberately chosen level of capability:

| Level | Tools | Suitable for |
|-------|-------|--------------|
| **Low** | `read`, `search` | Summarizing, reviewing, analyzing |
| **Medium** | `read`, `search`, `edit`, `execute` | Writing code, running tests |
| **High** | `agent`, `web`, MCP servers | Coordinating other agents, reaching external systems |

Grant the lowest level that can complete the task. A reviewer that cannot edit cannot
"helpfully" fix the bug it was asked to report — which is what you want, because you
asked for a report.

---

## Step 1 — Create the repository instructions file

**Create this file:** `.github/copilot-instructions.md`

```markdown
# Repository Instructions

## Architecture

- `src/api` is an ASP.NET Core 8 Web API using Entity Framework Core.
- Production data uses Azure SQL; local development and tests use EF Core InMemory.
- `src/frontend` is a React application built with Vite.
- `infra` contains Azure Bicep infrastructure.
- `.github/workflows` contains CI/CD workflows.

## Conventions

- Use async/await for database and service operations.
- Keep controllers thin; business logic belongs in services.
- Scope every todo operation by the authenticated user ID.
- Keep frontend API calls relative to `/api`.
- Make focused changes and preserve existing public APIs where possible.

## Testing

- API: `dotnet test src/api/Tests/TodoApi.Tests.csproj`
- Frontend: `cd src/frontend && npm ci && npm test`
- Frontend build: `cd src/frontend && npm run build`
- Bicep: `az bicep build --file infra/main.bicep`

Run relevant tests after every implementation change.

## Security

- Never commit credentials, tokens, or connection-string secrets.
- Use managed identity for Azure service authentication.
- Never trust a user ID supplied in a request body.
- Do not weaken authentication, authorization, or branch protections.
- Infrastructure and production deployment changes require human review.
```

### What each section is doing

**Architecture** orients the agent before it starts searching. Without it, an agent asked
to "add a field to the todo model" may not realize there are two database providers in
play and that a migration is required for one but not the other.

**Conventions** encode decisions that are invisible in any single file. "Keep controllers
thin" is not discoverable from reading one controller — it is a pattern across many, and
stating it prevents the agent from eroding it one reasonable-looking change at a time.

**Testing** is the highest-value section in the file. Agents are far more useful when they
can close their own feedback loop: make a change, run the tests, see the failure, correct
it. That only works if they know the exact command. Note the API command names a specific
`.csproj` — a bare `dotnet test` in this repository picks up an unrelated project and
produces confusing failures. Precision here directly buys you reliability.

**Security** states the constraints that must hold regardless of what a task asks for.
"Never trust a user ID supplied in a request body" is a real defect class: an agent
implementing a create endpoint will naturally bind the whole request object to the entity,
which would let a caller write rows on another user's behalf. The instruction pre-empts it.

### Why the tone is imperative

Every line is a directive, not a description. "Use async/await" rather than "the codebase
generally uses async/await." Descriptive prose invites the model to weigh it against other
considerations; imperative prose reads as a rule. Keep lines short and independently
checkable — a reviewer agent can cite a single bullet as evidence for a finding, which it
cannot do with a paragraph.

---

## Step 2 — Verify

Run from the repository root (`starter/`):

```bash
test -s .github/copilot-instructions.md && echo "PASS: file exists and is non-empty"
grep -q "## Security" .github/copilot-instructions.md && echo "PASS: security section present"
```

> **Use `test -s`, not `test -f`.** `-f` passes on an empty file. If your editor created
> the file but you have not saved it, `-f` reports success while the agent sees nothing.
> This is a genuinely common way to lose an hour.

### Behavioural test

The real check is whether the instructions change agent behaviour. Open Copilot Chat and ask:

> How do I run the API tests in this repository?

A correct answer names `dotnet test src/api/Tests/TodoApi.Tests.csproj`. If you get a
generic `dotnet test`, the instructions are not being picked up — confirm the file is at
`.github/copilot-instructions.md` exactly, and that it is saved.

---

## Exam notes

### Instruction file locations

| File | Scope |
|------|-------|
| `.github/copilot-instructions.md` | Entire repository |
| `.github/instructions/*.instructions.md` | Specific paths, via an `applyTo` glob |
| `AGENTS.md` | Agent-oriented instructions, read by multiple agent tools |

### Choosing an autonomy level

| Task | Autonomy | Reason |
|------|----------|--------|
| Summarize code | Low (`read`, `search`) | Nothing needs to change |
| Add tests | Medium (`read`, `search`, `edit`, `execute`) | Must write files and run them |
| Modify deployment | High, with added controls | Production impact |

### What makes a task suitable for an agent

1. Clear inputs and outputs
2. Scoped to a repository, branch, or pull request
3. Reviewable through a diff, artifact, or log
4. Validatable by tests or scans
5. Achievable with least-privilege tools

### The distinction most often tested

**An agent-generated plan does not prove the implementation is safe.** A convincing plan
is not evidence. Safety comes from artifacts a human can review — a pull request diff, a
test result, a scan report. If a question offers "the agent produced a detailed plan" as
justification for skipping review, it is the wrong answer.

---

## Common pitfalls

**The file exists but is empty.** Creating a file in an editor does not write it to disk
until you save. Always verify with `test -s` or `wc -c`.

**Instructions written as suggestions.** "Try to keep controllers thin" is weaker than
"Keep controllers thin." Write rules.

**Instructions that restate the obvious.** "This is a C# project" adds nothing the agent
cannot see. Spend the space on what is *not* visible from any single file.

**Instructions that go stale.** A test command that no longer works is worse than no
command, because the agent will trust it and report a confusing failure. Update this file
when the build changes.

---

## What you built

A single file that every agent in this repository now reads before doing anything. It is
the foundation for Exercise 2, where you create agents with specific roles — each one
inherits these instructions automatically, so none of them need to restate the
architecture, the test commands, or the security rules.

**Next:** [Exercise 2 — Tool Use & Environment](exercise-02-tool-use-and-environment.md)
