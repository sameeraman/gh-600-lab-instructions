# Exercise 8 — Exam Practice & Cheat Sheet

**Goal:** test recall under exam conditions and consolidate everything into a reference you
can revise from.

**Prerequisite:** [Exercise 7](exercise-07-full-pipeline-integration.md) complete.

**Time:** ~30 minutes

---

## How these questions work

GH-600 questions tend to give you an artifact — a YAML fragment, a JSON config, a log
excerpt — and ask you to identify what is correct, what is missing, or what a value should
be. They reward recognizing configuration by shape rather than recalling prose.

Answer all ten before checking. A wrong answer you reasoned through is more useful than a
right one you guessed.

---

## Questions

### Q1

What is the REQUIRED field in custom agent YAML frontmatter?

A) `name` · B) `description` · C) `tools` · D) `model`

<details>
<summary>Answer</summary>

**B) `description`.** `name` is optional. The description is how a coordinating agent
decides whether to delegate to this agent, which is why it is the mandatory one.

</details>

### Q2

Given this MCP config, what is the correct `type`?

```json
{
  "mcpServers": {
    "sentry": {
      "type": "???",
      "url": "https://mcp.sentry.io/sse"
    }
  }
}
```

A) `local` · B) `stdio` · C) `http` · D) `sse`

<details>
<summary>Answer</summary>

**C) `http`** (or `sse` if `http` is unavailable — SSE is the legacy form).

The decision rule is about `url` versus `command`, not about what the URL contains. This
one has a `url` and no `command`, so it is remote. The `/sse` in the path is a distractor.

</details>

### Q3

What happens when a `preToolUse` hook returns `{"permissionDecision": "ask"}` in a cloud
agent?

A) Agent is prompted to confirm · B) Tool executes normally · C) Tool is denied · D) Hook fails open

<details>
<summary>Answer</summary>

**C) Tool is denied.** A cloud agent is non-interactive — there is nobody to ask — so `ask`
fails closed. Note that D is the dangerous-sounding wrong answer: security controls should
never fail open.

</details>

### Q4

Which file configures the cloud agent environment?

A) `.github/workflows/ci.yml` · B) `.github/workflows/copilot-setup-steps.yml` ·
C) `.github/copilot-instructions.md` · D) `.github/agents/setup.agent.md`

<details>
<summary>Answer</summary>

**B) `.github/workflows/copilot-setup-steps.yml`** — and the job key inside it must be
exactly `copilot-setup-steps`, on the default branch.

</details>

### Q5

An agent needs to invoke another custom agent. Which tool is required?

A) `execute` · B) `search` · C) `agent` · D) `web`

<details>
<summary>Answer</summary>

**C) `agent`.** `execute` runs shell commands, which is a different capability entirely.

</details>

### Q6

What prevents stale workflow runs when multiple agents push to the same PR branch?

A) `fail-fast: true` · B) `concurrency` with `cancel-in-progress: true` ·
C) `needs: [previous-job]` · D) `environment: production`

<details>
<summary>Answer</summary>

**B) `concurrency` with `cancel-in-progress: true`.**

`fail-fast` controls matrix legs within one run. `needs` orders jobs. Neither cancels an
older run.

</details>

### Q7

Where would you find who deleted a workflow artifact?

A) PR timeline · B) Session logs · C) Organization audit log · D) Workflow logs

<details>
<summary>Answer</summary>

**C) Organization audit log**, the `artifact.destroy` event, which carries the actor.
Workflow logs record what happened inside a run, not administrative actions taken later.

</details>

### Q8

What is the difference between `/delegate` and `/fleet`?

A) `/delegate` splits work, `/fleet` sends to cloud · B) `/delegate` sends to cloud agent,
`/fleet` splits into parallel subagents · C) They are the same · D) `/fleet` is for CI only

<details>
<summary>Answer</summary>

**B).** `/delegate` hands a task to a cloud agent for background work. `/fleet` splits work
into parallel local subagents.

Mnemonic: delegate sends work *away*; a fleet is many things working *together*.

</details>

### Q9

Copilot pushes workflow changes to a PR but the workflows do not run. What should you do?

A) Re-push · B) Fix the syntax · C) Click "Approve and run workflows" · D) Recreate the PR

<details>
<summary>Answer</summary>

**C) Click "Approve and run workflows."**

This is an accountability gate, not a bug. Without it, an agent could modify the workflow
that validates its own changes and have the modified version run immediately — granting
itself capability. A human must approve first.

</details>

### Q10

A hook log shows `toolName: 'grep'`. What agent capability does this map to?

A) `read` · B) `search` · C) `execute` · D) `web`

<details>
<summary>Answer</summary>

**B) `search`.** Both `grep` and `glob` map to `search`. The related trap is `view`, which
maps to `read` and means reading a file — not browsing the web.

</details>

---

## Score guide

| Score | Interpretation |
|-------|---------------|
| 9–10 | Ready |
| 7–8 | Solid; revisit the domains you missed |
| 5–6 | Re-read the exam notes in each guide |
| < 5 | Work through the exercises again, building the files yourself |

---

## Cheat sheet

### Critical file paths

```
.github/copilot-instructions.md            Repository instructions
.github/instructions/*.instructions.md     Path-specific (requires applyTo)
AGENTS.md                                  Agent instructions
.github/agents/*.agent.md                  Agent profiles (requires description)
.github/prompts/*.prompt.md                Reusable prompts
.github/skills/<name>/SKILL.md             Agent skills
.github/hooks/*.json                       Guardrail hooks
.github/workflows/copilot-setup-steps.yml  Cloud setup (job name must match)
.mcp.json  /  .github/mcp.json             MCP config
~/.copilot/session-state/<id>/events.jsonl Session state
```

### Tool permission levels

```
Low:    read, search
Medium: read, search, edit, execute
High:   read, search, edit, execute, agent (+ MCP)
```

### MCP type decision

```
Has command + args? → local / stdio
Has url?            → http / sse
URL inside args?    → still local (bridge pattern)
```

Naming: `mcpServers` in JSON, `mcp-servers` in YAML frontmatter.

### Hook decisions (`preToolUse`)

```
"allow" → tool executes
"deny"  → tool blocked (include a reason)
"ask"   → interactive: prompts user
          cloud agent: treated as DENY
```

### Hook toolName → agent capability

```
view   → read          edit   → edit
grep   → search        create → edit
glob   → search        bash   → execute
                       task   → agent
```

### CI/CD patterns

```yaml
# Matrix agents
strategy:
  fail-fast: false
  matrix:
    agent: [reviewer, security-scanner, auditor]

# Concurrency (cancel stale)
concurrency:
  group: ${{ github.workflow }}-${{ github.head_ref || github.run_id }}
  cancel-in-progress: true

# Every run must complete
concurrency:
  group: production-agent-work
  queue: max          # never with cancel-in-progress

# Handoff
needs: [review, audit]
# + upload-artifact / download-artifact
```

### CLI flags

```
--agent=NAME                      Select an agent profile
--no-ask-user                     Prevent CI hang
--allow-tool='read,search'        Restrict tools inline
--allow-tool='shell(curl:*)'      Scope shell to one command
--resume / --continue             Resume a session
COPILOT_GITHUB_TOKEN              Authentication
```

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

### Control categories

```
Preventive  Tools, hooks, branch protection, rulesets
Detective   Session logs, CodeQL, secret scanning, audit logs
Corrective  Revert PR, stop session, rotate secrets
```

### Enforcement ladder

```
Instructions      guide       not enforceable
Tool lists        bound       enforced by runtime
Hooks             intercept   can deny
Branch protection enforce     cannot be bypassed
```

### Approval gates

```
Branch protection             → merging
Environment required reviewer → deploying
"Approve and run workflows"   → agent-modified workflows
```

### Artifact configuration

```yaml
- uses: actions/upload-artifact@v7
  if: always()                      # upload even on failure
  with:
    name: report
    path: report.md
    if-no-files-found: error        # fail loudly on missing output
    retention-days: 7               # artifacts expire; PR comments do not
```

---

## The ideas underneath

If you remember nothing else:

**Least privilege at every layer.** Agent tool lists, MCP tool filters, workflow token
permissions, and scoped shell allowances. Each independently limits the blast radius of a
failure at that layer.

**Guidance is not enforcement.** Instructions tell an agent what you want. Tool lists, hooks,
and branch protection determine what is possible. When the consequence is serious, do not
rely on the first.

**Agents have no memory.** They have context. Anything that must persist goes into a durable
artifact — an instruction file, a PR comment, a workflow artifact.

**Evidence over impression.** A plan is not proof. A confident summary is not proof. Tests,
scans, diffs, and logs are proof.

**Keep agent work inside GitHub's accountability chain.** Issue → branch → PR → checks →
review → merge. An agent operating inside that chain can be trusted with far more autonomy
than one operating beside it.

---

## Resources

- [GH-600 skills outline](https://learn.microsoft.com/en-us/credentials/certifications/exams/gh-600/)
- [GitHub Copilot documentation](https://docs.github.com/en/copilot)
- [GitHub Actions documentation](https://docs.github.com/en/actions)

Good luck.
