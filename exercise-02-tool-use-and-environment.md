# Exercise 2 — Implement Tool Use & Environment (Domain 2)

**Goal:** build a team of specialized agents with deliberately different capabilities,
connect them to external tools, and prepare an environment they can run in on GitHub's
infrastructure.

**You will create:**

| Step | File | Purpose |
|------|------|---------|
| 1 | `.github/agents/reviewer.agent.md` | Low-autonomy code reviewer |
| 2 | `.github/agents/test-runner.agent.md` | Medium-autonomy test executor |
| 3 | `.github/agents/security-scanner.agent.md` | Security analyst that reports but cannot fix |
| 4 | `.github/agents/orchestrator.agent.md` | Delegating coordinator |
| 5 | `.mcp.json` | External tool servers |
| 6 | `.github/workflows/copilot-setup-steps.yml` | Cloud-agent environment |

**Prerequisite:** [Exercise 1](exercise-01-agent-architecture.md) complete.

**Time:** ~55 minutes

---

## The idea behind this exercise

Exercise 1 produced one set of instructions for all agents. That works until you want an
agent that reviews code *without* being able to change it, and another that must change
code to do its job at all. Those are contradictory requirements, and no single
configuration satisfies both.

Custom agents resolve this. Each is a file describing one role: a persona, a set of
responsibilities, and — the part that matters most — an explicit list of tools. The tool
list is not a hint. It is the boundary of what the agent can physically do.

The four agents you build here form a deliberate progression in capability:

```
reviewer          read, search                    → can look, cannot touch
test-runner       read, search, edit, execute     → can write and run
security-scanner  read, search, execute           → can run and inspect, cannot fix
orchestrator      read, search, agent             → can delegate, cannot act directly
```

Notice that the orchestrator, despite sitting at the top, is *less* capable than the
test-runner in raw terms. Coordination authority and execution authority are separated on
purpose. A coordinator that cannot edit files cannot cause damage through a reasoning
error — the worst it can do is ask the wrong agent to do something, and that agent's own
limits still apply.

---

## Anatomy of an agent file

Every file in `.github/agents/` follows the same shape: YAML frontmatter, then a markdown
body that becomes the agent's system prompt.

````markdown
---
name: example
description: "One sentence describing when to use this agent."
tools:
  - read
  - search
---

You are ... (the body becomes the agent's instructions)
````

**`description` is the required field.** This trips people up, because `name` looks more
important. The description is what a coordinating agent reads when deciding whether to
delegate to this agent — it is the agent's advertisement of its own purpose, so write it
as "when you would use this," not "what this is."

### The available tools

| Tool | Grants |
|------|--------|
| `read` | Reading file contents |
| `search` | Searching across the codebase |
| `edit` | Creating and modifying files |
| `execute` | Running shell commands |
| `agent` | Invoking other agents |
| `web` | Fetching external web content |

---

## Step 1 — The reviewer agent (low autonomy)

**Create this file:** `.github/agents/reviewer.agent.md`

````markdown
---
name: reviewer
description: "Reviews Todo application changes for defects, security risks, missing tests, and violations of repository conventions."
tools:
  - read
  - search
---

You are a read-only code reviewer for the Todo application.

## Review Checklist

1. Identify correctness and security defects.
2. Check input validation and error handling.
3. Confirm todo operations are scoped by authenticated user ID.
4. Check async/await usage in the API.
5. Identify missing tests for changed behavior.
6. Verify frontend API calls use relative `/api` paths.
7. Check for committed secrets or credentials.

## Constraints

- Do not edit files.
- Do not execute commands.
- Report only actionable findings supported by code evidence.
- List the most severe findings first.

## Output Format

For every finding, report:

- **Severity**: Critical, High, Medium, or Low
- **File**: repository-relative path
- **Finding**: concise explanation
- **Recommendation**: specific remediation

If no defects are found, state that clearly and mention remaining test gaps.
````

### Why this shape

**The tool list is the actual guarantee.** The body says "do not edit files," but that
line is a request the model could in principle rationalize around. The absence of `edit`
from the tool list is enforced by the runtime. Write both: the instruction explains the
intent, the tool list makes it true. When an exam question asks how you *guarantee* an
agent cannot modify code, the answer is the tool list, not the prompt.

**The checklist reflects this repository.** Items 3 and 6 are not generic review advice —
they map to specific rules in your `copilot-instructions.md`. A reviewer whose checklist
mirrors your conventions produces findings you actually care about.

**A structured output format makes the result machine-readable.** Later exercises pipe
review output into a consolidation step. Free-form prose cannot be aggregated; severity
plus file plus finding can.

**Verify:**

```bash
test -s .github/agents/reviewer.agent.md && echo "PASS: file non-empty"
grep -q "description:" .github/agents/reviewer.agent.md && echo "PASS: has description"
grep -qE "^  - (edit|execute)" .github/agents/reviewer.agent.md \
  && echo "FAIL: reviewer should not have edit or execute" \
  || echo "PASS: read-only"
```

**Behavioural test:** select `reviewer` in the Chat agent picker and ask it to fix a bug.
It should report the bug and decline to change the file. That refusal is the feature.

---

## Step 2 — The test-runner agent (medium autonomy)

**Create this file:** `.github/agents/test-runner.agent.md`

````markdown
---
name: test-runner
description: "Runs API, frontend, and E2E tests for the Todo application, diagnoses failures, and reports results with recommended fixes."
tools:
  - read
  - search
  - edit
  - execute
---

You are the test execution and analysis agent for the Todo application.

## Responsibilities

1. Run the relevant test suites for the changed code.
2. Diagnose failures using test output and source evidence.
3. Repair broken or missing tests when the fix is clearly in test code.
4. Report results and remaining risks.

## Commands

- API tests: `dotnet test src/api/Tests/TodoApi.Tests.csproj`
- Frontend tests: `cd src/frontend && npm ci && npm test`
- E2E tests: `cd src/frontend && npx playwright test`

## Constraints

- Do not weaken or delete assertions to make tests pass.
- Do not modify production code to satisfy a failing test without stating the cause.
- Report the exact command and its result.

## Output Format

```markdown
# Test Summary

## Results
- API: X passed, Y failed
- Frontend: X passed, Y failed

## Failures
### [Test Name]
- **Error**: message
- **Root Cause**: analysis
- **Fix**: recommendation

## Coverage Gaps
- Untested behavior worth covering
```
````

### Why this agent gets more power

It cannot do its job otherwise. Running a test requires `execute`; repairing a broken test
requires `edit`. This is the correct reason to grant capability — a concrete task
requirement, not a general sense that more is better.

But raising capability raises risk, and the risk here is specific and worth naming. An
agent told "make the tests pass" and given `edit` has two paths available: fix the code,
or delete the assertion. The second is faster and technically satisfies the request. The
**Constraints** section closes that path explicitly.

This is the general pattern for medium autonomy: **for every capability you grant, write
the constraint that prevents its cheapest misuse.**

> ### Watch the nested code fence
>
> This file contains a fenced code block *inside* it (the Output Format). When copying,
> make sure the final ` ``` ` is included — it is easy to lose, and the guide above shows
> the file wrapped in four backticks precisely so the inner three-backtick fences survive.
> An unterminated fence will not break the agent, but it makes the prompt render oddly.

**Verify:**

```bash
test -s .github/agents/test-runner.agent.md && echo "PASS: file non-empty"
grep -qE "^  - execute" .github/agents/test-runner.agent.md && echo "PASS: medium autonomy"
```

**Behavioural test:** select `test-runner` and ask it to run the API tests. It should
execute `dotnet test` and report results — the capability the reviewer lacked.

---

## Step 3 — The security-scanner agent (report-only)

**Create this file:** `.github/agents/security-scanner.agent.md`

````markdown
---
name: security-scanner
description: Performs security analysis that static tooling cannot reach — tenancy scoping, request-binding surface, IaC and workflow configuration, and triage of CodeQL and dependency-review findings.
tools:
  - read
  - search
  - execute
---

You are the security scanning agent.

GitHub Advanced Security already runs in `ci.yml`: CodeQL analyses C# and JavaScript, and
`dependency-review-action` gates the manifest diff. Do not repeat those checks. You have no
advisory database and no dataflow engine, and a non-deterministic second opinion is worse than
none. Your job is what they cannot parse, plus judgement on what they report.

## Scan Every Run

- **Tenancy scoping** — every query against `_context.TodoItems` must filter by the caller's
  `userId`. Flag any data-access path, new or modified, that does not. This is the invariant most
  likely to be broken by a new endpoint, and no static rule expresses it.
- **Request-binding surface** — actions that bind an entity straight from the body
  (`Create(TodoItem item)`) let a client set every property on it, including `Id`, `UserId`, and
  timestamps. Report any bound property the caller should not control, even where the service
  layer happens to overwrite it afterwards.
- **Authorization coverage** — every action reachable under `api/` must be covered by
  `[Authorize]`, or carry an explicit and justified `[AllowAnonymous]`.
- **IaC and application-code agreement** — Bicep and `Program.cs` must not contradict each other.
  Watch for `azureADOnlyAuthentication` enabled while a connection-string path remains,
  `Authentication__RequireSignedTokens` differing across environments, or CORS and ingress
  widened in one place only.
- **Trust boundary** — the API trusts `x-ms-client-principal` only because Easy Auth strips
  inbound copies and the Static Web App linked backend injects a fresh one. Flag any change that
  adds ingress, alters routing, or reads that header somewhere new.
- **Workflow security** — `pull_request_target` combined with a checkout of the PR head,
  unpinned action versions, over-broad `permissions`, and any step that writes a secret into a
  log, step summary, or uploaded artifact.
- **Bicep configuration** — public endpoints, missing HTTPS enforcement, over-permissive network
  rules, weakened authentication settings. No static analyser covers ARM.
- **Finding triage** — for each CodeQL and dependency-review finding, state whether the affected
  path is reachable in this repository, what the blast radius is, and the minimal fix. Say so
  explicitly when a finding is a false positive, and why.

Never duplicate a deterministic scanner. If a check can be expressed as a CodeQL query or an
advisory-database lookup, it belongs in `ci.yml`, not here.

## Security Rules
- HTTPS must be enforced for all services
- No secrets in code, logs, or artifacts
- Managed identity preferred over connection strings
- Minimal permissions in workflow YAML
- No `pull_request_target` with checkout of PR head without review

## Output Format
```markdown
# Security Scan Report

## Summary
- Critical: X | High: Y | Medium: Z | Low: W

## Findings
### [Finding ID]
- **Severity**: Critical/High/Medium/Low
- **Category**: Tenancy/Binding/AuthZ/Config-Drift/Trust-Boundary/Workflow/Infrastructure/Triage
- **Location**: file:line
- **Description**: What was found
- **Remediation**: How to fix
```
````

### Why `execute` but not `edit`

This agent runs `az bicep build` and reads the SARIF that CI produces, so it needs `execute`.
It reports findings rather than fixing them, so it does not get `edit`.

That combination is the point of this step. It is a **third autonomy shape**, sitting
alongside the low reviewer and the medium test-runner, and it is the one people get wrong:
they assume `execute` and `edit` travel together. They do not. Security findings need human
judgement about severity and remediation, and an agent that silently "fixed" a vulnerability
would hide the decision that mattered.

### Why it doesn't look for CVEs or hardcoded secrets

Because `ci.yml` already does, and does it better. CodeQL performs interprocedural taint
analysis and writes SARIF into the Security tab with deduplication and dismissal state.
`dependency-review-action` compares the resolved manifest diff against a versioned advisory
database and *fails the job* at high severity. An agent asked the same questions has neither
a database nor a dataflow engine, returns different answers on re-run, and cannot gate.

This is the design rule worth taking into the exam: **give the agent the checks that require
judgement, and leave the checks that require determinism to the platform.** Tenancy scoping,
binding surface, and IaC-versus-code drift are semantic questions no query language expresses.
A known CVE in a lockfile is not.

The corollary is the triage bullet. GHAS is excellent at finding things and has no opinion
about whether they matter *here* — whether the vulnerable function is even reachable. That
gap is where the agent earns its place in the pipeline.

Note the last security rule. `pull_request_target` runs with elevated permissions in the
base repository's context; combining it with a checkout of the PR head executes untrusted
code with those permissions. It is a well-known GitHub Actions escalation, and worth
recognizing.

**Verify:**

```bash
test -s .github/agents/security-scanner.agent.md && echo "PASS: file non-empty"
grep -qE "^  - execute" .github/agents/security-scanner.agent.md && echo "PASS: can run tools"
grep -qE "^  - edit" .github/agents/security-scanner.agent.md \
  && echo "FAIL: scanner should report, not fix" \
  || echo "PASS: report-only"
```

**Behavioural test:** select `security-scanner` and ask it to review `TodoController.cs`. It
should raise the request-binding surface on `Create` and `Update` — and it should *not* offer
to patch the file.

---

## Step 4 — The orchestrator agent (delegation)

**Create this file:** `.github/agents/orchestrator.agent.md`

````markdown
---
name: orchestrator
description: "Coordinates the reviewer and test-runner agents to produce a single consolidated quality report for a change."
tools:
  - read
  - search
  - agent
---

You are the orchestration agent for the Todo application CI/CD pipeline.

## Workflow

1. Delegate code review to the `reviewer` agent.
2. Delegate test execution to the `test-runner` agent.
3. Consolidate both reports into one summary.

## Coordination Rules

- Invoke the sub-agents in parallel when their work is independent.
- Do not repeat analysis a sub-agent has already performed.
- Do not run tests or edit files yourself — delegate.
- If any sub-agent reports a Critical finding, the overall risk is Critical.
- Cite the originating agent for every finding you carry forward.

## Output Format

```markdown
# Consolidated Quality Report

## Overall Risk: Low | Medium | High | Critical

## Blocking Issues
- [reviewer|test-runner] issue + evidence

## Advisory Issues
- [reviewer|test-runner] issue + evidence

## Positive Observations
- ...
```
````

### The `agent` tool

`agent` is what permits one agent to invoke another. Without it, an "orchestrator" can
only describe what should happen; with it, delegation actually occurs. This is a
frequently tested fact, usually phrased as *"which tool enables agent-to-agent
invocation?"*

### Read the tool list for what is missing

No `edit`. No `execute`. The orchestrator has no hands of its own.

That is the entire design. Concentrating coordination and execution in one agent creates a
component whose reasoning errors translate directly into file changes. Splitting them
means a confused orchestrator can only misroute a request, and the receiving agent's own
tool boundary still holds. Least privilege applies to coordinators too — arguably more,
since they touch every task.

**Severity escalation** ("if any sub-agent reports Critical, overall is Critical") exists
because aggregation naturally dilutes. Given nine clean reports and one critical finding,
a summarizer will tend toward "mostly fine." For a quality gate, the worst finding must
dominate. State that rule explicitly or you will not get it.

**Verify:**

```bash
test -s .github/agents/orchestrator.agent.md && echo "PASS: file non-empty"
grep -qE "^  - agent" .github/agents/orchestrator.agent.md && echo "PASS: can delegate"
grep -qE "^  - (edit|execute)" .github/agents/orchestrator.agent.md \
  && echo "FAIL: orchestrator should delegate, not act" \
  || echo "PASS: delegation-only"
```

**Behavioural test:** select `orchestrator` and ask for a review of recent changes. It
should hand work to the other two agents rather than doing everything itself.

---

## Step 5 — MCP configuration

**Create this file:** `.mcp.json` (repository root, beside `.github/`)

```json
{
  "mcpServers": {
    "github": {
      "type": "local",
      "command": "npx",
      "args": ["-y", "@github/mcp-server"],
      "tools": ["*"]
    },
    "playwright": {
      "type": "local",
      "command": "npx",
      "args": ["-y", "@playwright/mcp-server"],
      "tools": ["browser_navigate", "browser_click", "browser_snapshot"],
      "env": {
        "PLAYWRIGHT_HEADLESS": "true"
      }
    }
  }
}
```

### What MCP is for

Everything so far used built-in tools that operate on your repository. The Model Context
Protocol is the standard interface for attaching capability that lives *outside* it —
issue trackers, browsers, databases, internal APIs. Rather than each agent product
inventing its own plugin format, MCP defines one protocol that any server can implement
and any client can consume.

The two servers here are representative: the GitHub server reaches the platform (issues,
pull requests, checks), and the Playwright server drives a real browser, which is what
makes end-to-end verification possible.

### Three details that are tested

**`mcpServers` is camelCase in JSON.** In YAML frontmatter the same concept is written
`mcp-servers` with a hyphen. Same idea, two spellings, and the file format decides which.

**`"type": "local"` follows from `command` + `args`.** These servers are launched as child
processes on the machine. The decision rule:

| Configuration contains | Type |
|------------------------|------|
| `command` + `args` | `local` or `stdio` |
| `url` | `http` or `sse` |
| A URL *inside* `args`, with a `command` | still `local` — the command is a bridge |

That last row is the one exam questions are built from. A URL appearing somewhere in the
config does not make the server remote; what matters is whether a process is being spawned
locally.

**`tools` narrows the surface.** `"*"` grants everything the server exposes. The Playwright
entry instead lists three tools by name. Enumerating is the least-privilege choice: if the
server later adds a destructive capability, an explicit list does not silently inherit it.
This is the same principle as the agent tool lists, applied one layer out.

**Verify:**

```bash
test -s .mcp.json && python3 -m json.tool .mcp.json > /dev/null && echo "PASS: valid JSON"
grep -q '"mcpServers"' .mcp.json && echo "PASS: camelCase key"
grep -q '"type": "local"' .mcp.json && echo "PASS: local type"
```

> The servers are fetched with `npx`. If your network or registry blocks that, they will
> fail to start locally. The lab is assessing the configuration file, so a start failure
> does not block progress.

---

## Step 6 — Cloud-agent environment setup

**Create this file:** `.github/workflows/copilot-setup-steps.yml`

```yaml
name: Copilot Setup Steps

on:
  workflow_dispatch:

jobs:
  # Job name must be exactly this — Copilot discovers the job by name.
  copilot-setup-steps:
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@v6

      - uses: actions/setup-dotnet@v6
        with:
          dotnet-version: '8.0.x'

      - uses: actions/setup-node@v7
        with:
          node-version: '22'
          cache: 'npm'
          cache-dependency-path: src/frontend/package-lock.json

      - name: Restore API dependencies
        run: dotnet restore src/api/TodoApi.csproj

      - name: Install frontend dependencies
        run: cd src/frontend && npm ci

      - name: Install Playwright browsers
        run: cd src/frontend && npx playwright install --with-deps chromium
```

### Why this file exists

Your agents have worked locally so far, where the .NET SDK, Node, and your dependencies
are already installed. When Copilot runs as a cloud agent it gets a clean container with
none of that. An agent asked to run the tests would fail at `dotnet: command not found` —
not because its instructions were wrong, but because its environment was empty.

This workflow provisions that container. It is the environment half of "Tool Use &
Environment": tools describe what an agent may do, environment determines whether it can.

### Two hard requirements

**The job key must be exactly `copilot-setup-steps`.** Not the workflow's `name:` — the key
under `jobs:`. Copilot looks up that exact key. Rename it and setup is skipped silently;
the agent starts unprepared and fails on its first build command, with nothing in the logs
pointing at this file. A `name: copilot-setup-steps` field inside a differently-keyed job
does **not** satisfy it.

**The file must be on the default branch.** It is read from the default branch regardless
of which branch the agent is working on. Leaving it on a feature branch means it never
takes effect.

Note also `permissions: contents: read` — setup only needs to fetch the repository, so it
is granted nothing more. The same least-privilege reasoning as the tool lists, now applied
to workflow tokens.

**Verify:**

```bash
test -s .github/workflows/copilot-setup-steps.yml || echo "FAIL: file is empty — save it"
grep -q "^  copilot-setup-steps:" .github/workflows/copilot-setup-steps.yml \
  && echo "PASS: job key correct"
```

To validate the YAML parses *and* has the right structure — macOS ships Ruby, which
includes a YAML parser, whereas system Python often lacks `PyYAML`:

```bash
ruby -ryaml -e '
d = YAML.safe_load(File.read(".github/workflows/copilot-setup-steps.yml"), aliases: true) \
  or abort("FAIL: file is empty")
jobs = d["jobs"].keys
abort("FAIL: jobs are #{jobs}") unless jobs.include?("copilot-setup-steps")
puts "PASS: valid YAML with correct job key"'
```

> **Why `or abort`.** An empty file is *valid YAML* — it parses to `nil`. A naive syntax
> check therefore passes on a file containing nothing at all, which is the single most
> misleading result you can get here. Assert the structure, not just that parsing
> succeeded.

---

## Final verification

Run all of it from the repository root:

```bash
for f in \
  .github/copilot-instructions.md \
  .github/agents/reviewer.agent.md \
  .github/agents/test-runner.agent.md \
  .github/agents/security-scanner.agent.md \
  .github/agents/orchestrator.agent.md \
  .mcp.json \
  .github/workflows/copilot-setup-steps.yml
do
  [ -s "$f" ] && echo "OK    $f" || echo "EMPTY $f"
done
```

Every line must read `OK`. An `EMPTY` result almost always means an unsaved editor buffer
rather than a missing file — the file appears correct on screen while containing zero
bytes on disk.

---

## Exam notes

### File paths to memorize

| File | Purpose |
|------|---------|
| `.github/copilot-instructions.md` | Repository-wide instructions |
| `.github/instructions/*.instructions.md` | Path-specific instructions |
| `AGENTS.md` | Agent-oriented instructions |
| `.github/agents/*.agent.md` | Custom agent profiles |
| `.github/prompts/*.prompt.md` | Reusable prompts |
| `.github/workflows/copilot-setup-steps.yml` | Cloud-agent setup |
| `.mcp.json` | MCP configuration |

### Agent file facts

- `description` is the **required** frontmatter field, not `name`.
- Tools available: `read`, `search`, `edit`, `execute`, `agent`, `web`.
- `agent` is the tool that enables agent-to-agent invocation.
- The tool list — not the prompt text — is what actually constrains an agent.

### MCP type decision rule

- Has `command` + `args`? → `local` or `stdio`
- Has `url`? → `http` or `sse`
- URL inside `args` alongside a command? → still `local`; the command is a bridge

### Naming

- JSON: `mcpServers` (camelCase)
- YAML frontmatter: `mcp-servers` (hyphenated)

### CLI flags for running agents in CI

| Flag / variable | Effect |
|-----------------|--------|
| `--no-ask-user` | Prevents the agent blocking on a prompt and hanging the job |
| `--agent=NAME` | Selects a specific custom agent |
| `COPILOT_GITHUB_TOKEN` | Authentication |

`--no-ask-user` is the one to remember. An agent that pauses for confirmation in a
non-interactive runner will hang until the job times out.

---

## Common pitfalls

**Files that are empty on disk.** The most common failure in this exercise. Verify with
`test -s`, never `test -f`.

**Copying a nested code fence incompletely.** The test-runner, security-scanner, and
orchestrator files each contain a fenced block; the closing fence is easy to drop.

**Renaming the setup job.** It must be `copilot-setup-steps` under `jobs:`. There is no
warning when this is wrong.

**Granting tools speculatively.** Adding `edit` to the reviewer "just in case" removes the
guarantee that made it a reviewer.

**Assuming `execute` implies `edit`.** They are independent. The security-scanner runs
commands and still cannot change a single byte.

**Assuming the prompt enforces the constraint.** "Do not edit files" in the body is intent;
omitting `edit` from the tool list is enforcement. Write both, and know which one is doing
the work.

---

## What you built

A four-agent team with graduated, deliberately chosen capability; a configuration
attaching external tools through MCP; and a provisioned environment for running on
GitHub's infrastructure.

The through-line across every step is least privilege applied at four layers: the agent's
tool list, the MCP server's tool filter, the workflow's token permissions, and the
separation of coordination from execution. Each layer independently limits what a failure
at that layer can reach.

**Next:** Exercise 3 — Memory and State (Domain 3).
