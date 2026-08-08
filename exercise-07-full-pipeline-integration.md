# Exercise 7 — Full Pipeline Integration

**Goal:** complete the agent team, add path-specific instructions and reusable prompts, and
put a human approval gate in front of production.

**You will create:**

| File | Purpose |
|------|---------|
| `.github/agents/auditor.agent.md` | Compliance auditor |
| `AGENTS.md` | Agent-oriented project documentation |
| `.github/instructions/api.instructions.md` | Rules scoped to `src/api/**` |
| `.github/instructions/workflows.instructions.md` | Rules scoped to workflows |
| `.github/prompts/security-review.prompt.md` | Reusable security review prompt |
| `.github/prompts/test-analysis.prompt.md` | Reusable test analysis prompt |
| `.github/workflows/ci.yml` | Extended with E2E evidence, auditor review, consolidation, and gated deployment |

**Prerequisite:** [Exercise 6](exercise-06-guardrails-and-accountability.md) complete.

**Time:** ~50 minutes

---

## What is still missing

Exercise 5 extended the starter's `ci.yml` so it reviews code and scans it for security
problems. Nothing in it yet asks whether the *change process* was followed — whether the gates
are intact, whether approval is still required, whether a safety control was quietly removed.
That is a different question from "is this code correct?", and it needs its own agent.

This exercise adds that agent, fills in the remaining layers of the context system —
repository-level documentation, path-scoped instructions, reusable prompts — and closes the
loop by putting a human in front of production.

---

## Step 1 — The auditor agent

**Create this file:** `.github/agents/auditor.agent.md`

````markdown
---
name: auditor
description: Audits changes for compliance, traceability, and deployment safety. Verifies that changes follow project governance and CI/CD requirements.
tools:
  - read
  - search
---

You are a compliance and deployment auditor for the Todo application.

## Audit Checklist
1. Verify all workflow changes include required checks
2. Confirm branch protection rules are not weakened
3. Check that secrets are not exposed in logs or artifacts
4. Verify infrastructure changes have parameter validation
5. Confirm deployment workflows require environment approval
6. Check that dependency changes are reviewed (dependency-review action)
7. Verify CodeQL is enabled for security scanning

## Output Format
Report as:
- **Area**: CI/CD | Security | Infrastructure | Compliance
- **Status**: Pass | Fail | Warning
- **Finding**: Description
- **Evidence**: File path or artifact reference
- **Recommendation**: Required action (if status is not Pass)

Do not edit files. Report audit findings only.
````

### Auditor versus reviewer

Both are low autonomy, and at a glance they overlap. They do not.

The **reviewer** asks *is this code correct?* — logic, validation, tests, conventions.
The **auditor** asks *is this change governed?* — are the gates intact, is approval
required, is CodeQL still enabled.

Splitting them matters because they fail differently. A reviewer distracted by governance
questions produces shallower code review. More importantly, item 2 on the auditor's
checklist — "branch protection rules are not weakened" — is the one nobody thinks to check
while reading a diff, and it is exactly how a safety control quietly disappears.

---

## Step 2 — AGENTS.md

**Create this file:** `AGENTS.md` (repository root)

```markdown
# AGENTS.md

## Project Structure

- `src/api/` - ASP.NET Core 8 Todo API with EF Core.
- `src/frontend/` - React and Vite frontend.
- `infra/` - Azure infrastructure defined with Bicep.
- `.github/workflows/` - Build, test, security, and deployment workflows.

## Testing Commands

- API: `dotnet test src/api/Tests/TodoApi.Tests.csproj`
- Frontend: `cd src/frontend && npm test`
- Frontend build: `cd src/frontend && npm run build`
- E2E: `cd src/frontend && npx playwright test`

## Agent Guidelines

- Work on a branch and submit changes through a pull request.
- Do not push directly to `main`.
- Use read-only tools for reviews.
- Use edit and execute tools only for implementation or testing tasks.
- Keep changes scoped to the requested task.
- Report validation results and unresolved risks.
- Require human review for deployment and infrastructure changes.
```

`AGENTS.md` is an emerging cross-tool convention: a repository-root file that agent tooling
reads regardless of vendor. `copilot-instructions.md` is Copilot-specific; `AGENTS.md` is
portable. Keeping both means the repository stays legible to whatever tool a contributor
brings.

For the exam, know the distinction: **`.github/copilot-instructions.md` is Copilot's
repository instructions; `AGENTS.md` is agent-oriented project documentation.**

---

## Step 3 — Path-specific instructions

Repository instructions apply everywhere. Some rules only make sense in one directory, and
loading them globally wastes context and invites misapplication.

**Create this file:** `.github/instructions/api.instructions.md`

```markdown
---
applyTo: "src/api/**"
---

# API Development Instructions

- All controller actions must be async
- Use dependency injection for services
- Return appropriate HTTP status codes (201 for create, 204 for delete, 404 for not found)
- Validate input in controllers before calling services
- Use Entity Framework Core for data access
- Include XML documentation for public API methods
- All new endpoints require corresponding unit tests
```

**Create this file:** `.github/instructions/workflows.instructions.md`

```markdown
---
applyTo: ".github/workflows/**"
---

# Workflow Instructions

- Always use least-privilege permissions
- Pin action versions to full SHA or major version tag
- Use `--no-ask-user` flag for Copilot CLI in CI
- Set `fail-fast: false` for matrix agent jobs
- Upload artifacts with `if-no-files-found: error` for critical outputs
- Include concurrency control for PR-triggered workflows
- Use environment protection rules for deployment workflows
- Never expose secrets in step summaries or logs
```

### The `applyTo` glob

`applyTo` is the required frontmatter field for instruction files, and it scopes the rules
to matching paths. An agent working in `src/frontend` never sees the API rules.

The workflows file is worth pausing on: every rule in it is something you learned in
Exercises 4–6. Writing them down converts knowledge you happen to have into a constraint the
repository enforces on future changes — including changes made by agents. That is the whole
point of the instruction layer.

---

## Step 4 — Reusable prompts

**Create this file:** `.github/prompts/security-review.prompt.md`

```markdown
# Security Review

Review the selected changes for:
1. **Authentication & Authorization** — missing auth checks, privilege escalation
2. **Secret Exposure** — hardcoded secrets, tokens, keys in code or logs
3. **Injection Vulnerabilities** — SQL injection, command injection, XSS
4. **Dependency Risk** — known CVEs, outdated packages, typosquatting
5. **Workflow Security** — permission escalation, workflow injection, unsafe checkout
6. **Infrastructure Security** — public endpoints without auth, missing HTTPS, overly permissive network rules

Return findings with:
- Severity (Critical/High/Medium/Low)
- File path and line number
- Description of the vulnerability
- Recommended fix with code example where applicable
```

**Create this file:** `.github/prompts/test-analysis.prompt.md`

```markdown
# Test Analysis

Analyze the test results and provide:
1. **Summary** — pass/fail counts for each test suite
2. **Failure Analysis** — root cause for each failing test
3. **Coverage Gaps** — areas of code not covered by tests
4. **Recommendations** — suggested additional tests to improve coverage

Focus on actionable insights. For each failure, provide the most likely fix.
```

### Prompts versus agents

A prompt is a **task** you invoke; an agent is a **role** with capabilities. The security
review prompt can be handed to any agent — including the general one — whereas
`security-scanner.agent.md` defines who is doing the work and what they are permitted to do.

Use prompts for recurring requests you would otherwise retype. Use agents when the *tool
boundary* matters.

---

## Step 5 — Deployment with approvals

Everything so far in this exercise has produced *advice*. This step is about *authority*: what
actually reaches production, and who says so.

**You are not creating a new workflow file.** `ci.yml` already deploys on push to `main`. A
second workflow triggered on the same event would race it — two runs deploying the same Bicep
template to the same resource group, with no ordering between them and no way to predict which
one wins. You extend the pipeline that already exists.

### The shape you are building

```
build-and-test ─┬─> e2e-test ───────┐
                ├─> agent-review ───┤
                ├─> security-scan ──┼─> consolidate ─> publish-summary
                └─> dependency-review ┘       └───────> [human] ─> deploy
```

Four ideas, each independently examinable:

| Property | Mechanism |
|---|---|
| Nothing merges unproven | `e2e-test` runs on every pull request |
| Nothing ships unscanned | `deploy` waits for deterministic checks, E2E, and consolidation |
| A human decides | Required reviewer on the `production` environment |
| The deployed boundary is checked | `deploy` rejects a directly forged identity header |

### Proving the application before merge

The interesting question about an end-to-end suite is not *what* it tests but *when* it runs. A
suite that only runs after deployment can report a fault but cannot prevent one — by the time it
speaks, the bad code is already merged and already live.

So `e2e-test` runs on every pull request, and to do that it must not depend on anything a pull
request lacks: no Azure subscription, no secrets, no deployed environment. It gets there by
hosting the whole application on the runner. The API is started with **no connection string**,
which makes it fall back to an in-memory store and skip migrations entirely:

Add the `e2e-test` job from the completed repository's
[`ci.yml`](https://github.com/sameeraman/gh-600-lab/blob/main/.github/workflows/ci.yml) directly
after `dependency-review`. Preserve its job-level permissions, readiness loops, test harness,
evidence upload, screenshot publishing guard, and update-in-place pull request comment. The
excerpts below explain the parts that are easy to misconfigure; the linked completed workflow
is the canonical full job.

```yaml
      - name: Start the API
        env:
          ASPNETCORE_ENVIRONMENT: Development
          Authentication__RequireSignedTokens: 'false'
        run: dotnet run --project src/api/TodoApi.csproj -c Release --urls http://127.0.0.1:5000 &
```

The remaining obstacle is the front door. The Static Web App requires an interactive Entra
sign-in and a headless browser cannot perform one. Neither can a service principal — it holds a
client secret, not a sign-in — so no credential exists that would let Playwright in.

`src/frontend/e2e/test-harness.mjs` stands in for the Static Web App on the runner, doing the two
things the SWA does for the application:

| Static Web App | Harness |
|---|---|
| Serves the built SPA | Serves `dist/` |
| Answers `/.auth/me` with the signed-in user | Answers with a fixed test principal |
| Injects `x-ms-client-principal` when proxying to the linked backend | Injects the same header |

The suite covers the three things a todo application has to do: create a task, mark it done,
delete it. Evidence — a screenshot of every test, traces, and the HTML report — is uploaded as
the **e2e-evidence** artifact.

### Putting the evidence where the reviewer is

An artifact nobody opens is not evidence. By default a reviewer would have to leave the pull
request, find the workflow run, locate the artifact, download a zip and unpack it — which means
in practice they will read the green tick and move on.

So the job posts the results back to the pull request itself:

```yaml
      - name: Comment the results on the pull request
        if: always() && github.event_name == 'pull_request'
        uses: actions/github-script@v8
```

Three details make this work properly:

- **`permissions: pull-requests: write`** on the job. `GITHUB_TOKEN` is read-only for this scope
  otherwise, and the API call fails.
- **`if: always()`** — a comment that only appears when tests pass is worthless. The failing case
  is the one that needs reporting.
- **The comment is updated, not duplicated.** The script writes a hidden `<!-- e2e-results -->`
  marker, searches existing comments for it, and edits in place. Without that, a ten-commit pull
  request ends up with ten identical comments and reviewers learn to ignore all of them.

The body is built from Playwright's JSON reporter, which is why `playwright.config.cjs` lists it
alongside the HTML one:

```js
  reporter: [
    ['html', { open: 'never' }],
    ['json', { outputFile: 'results.json' }],
    ['list']
  ],
```

> **GitHub cannot display an image it cannot fetch.** Artifact contents have no URL, and
> markdown strips `data:` URIs, so a screenshot inside the zip can never appear in a comment. The
> job works around this by capturing one landing-page shot, pushing it to a side branch called
> `e2e-screenshots`, and embedding its `raw.githubusercontent.com` URL:
>
> ```yaml
>       - name: Capture the landing page
>         if: always()
>         working-directory: src/frontend
>         run: |
>           npx playwright screenshot --viewport-size=1280,800 --wait-for-timeout=1500 \
>             http://127.0.0.1:3000 landing.png
> ```
>
> That branch holds nothing but screenshots, one directory per pull request, and never merges
> anywhere. It needs `contents: write`, only works on a public repository — a private one's raw
> URLs require authentication and render as broken images — and is skipped for pull requests from
> forks, which are issued a read-only token. The per-test screenshots stay in the artifact.

A green tick tells you a job exited zero; the screenshots tell you what the application actually
looked like.

> **Be honest about what this does not test.** Authentication is outside the scope of the e2e
> run: the harness asserts a principal rather than earning one. Nor is the real database
> exercised, since the API runs in-memory. Both are genuine gaps. The correct response is to know
> they are there and say so in the test plan — not to let a green suite imply coverage it does
> not have.

### Verifying the deployed authentication boundary

The completed workflow keeps post-deployment validation inside `deploy`. After publishing the
API and frontend, it reads the App Service Easy Auth configuration and requires all three
platform controls to be enabled: authentication, mandatory authentication for every request,
and the linked Static Web Apps identity provider.

It then calls the API directly with a forged `x-ms-client-principal` header. Only `400`, `401`,
or `403` is acceptable. A success response would prove that a caller can bypass the Static Web
App front door and impersonate a user, so the deployment fails.

The pre-merge E2E suite therefore proves application behaviour, while the post-deployment check
proves this specific production authentication boundary. It does not claim to be a full
functional smoke test of Azure SQL or interactive Entra sign-in.

> **One environment, on purpose.** A real pipeline would deploy to a Test environment, smoke it,
> and only then ask for approval to promote to Production. The lab uses a single environment to
> keep the Azure footprint and the cost down. The mechanisms — the approval gate, the evidence
> artifact, the ordering of `needs:` — are identical either way; what changes is how many
> resource groups you pay for.

### The `environment` key is the approval gate

```yaml
  deploy:
    needs: [build-and-test, security-scan, dependency-review, e2e-test, consolidate]
    if: github.ref == 'refs/heads/main'
    environment:
      name: production
      url: ${{ steps.infra.outputs.apiAppUrl }}
```

You configured a required reviewer on `production` in
[lab preparation](lab-preparation.md), section 10. Because this job carries an `environment`, it
pauses there until a human approves — before a single Azure resource is touched. This is the
answer to "require human approval before production deploy" from Exercise 6.

Because the browser tests now run before the merge, the approver is deciding with the build, the
unit tests, CodeQL **and** the end-to-end evidence already in hand. The gate is a judgement about
timing and blast radius, not a substitute for testing.

It is worth being precise about why this is the right mechanism. Branch protection governs
*merging*. Environment reviewers govern *deploying*. They are different decisions, and code that
is fine to merge is not automatically fine to ship right now.

### OIDC authentication

`permissions: id-token: write` enables OIDC federation with Azure, so `azure/login` obtains a
short-lived token instead of a stored secret. No long-lived credential exists to leak. This is
the same principle as the managed identity used by the application itself.

---

## Step 6 — Complete the single workflow graph

In the same `ci.yml`, add the auditor to the Exercise 5 matrix:

```yaml
matrix:
  agent: [reviewer, security-scanner, auditor]
```

Then add `e2e-test` to `consolidate.needs` and update `deploy.needs` to the final dependency
set:

```yaml
  consolidate:
    needs: [dependency-review, agent-review, security-scan, e2e-test]

  deploy:
    needs: [build-and-test, security-scan, dependency-review, e2e-test, consolidate]
```

These edits progress the Exercise 5 workflow to the completed repository state without adding
a second CI workflow.

---

## Verify

```bash
for f in \
  .github/agents/auditor.agent.md \
  AGENTS.md \
  .github/instructions/api.instructions.md \
  .github/instructions/workflows.instructions.md \
  .github/prompts/security-review.prompt.md \
  .github/prompts/test-analysis.prompt.md \
  .github/workflows/ci.yml
do
  [ -s "$f" ] && echo "OK    $f" || echo "EMPTY $f"
done

grep -q 'applyTo:' .github/instructions/api.instructions.md && echo "PASS: applyTo present"
grep -q 'environment:' .github/workflows/ci.yml && echo "PASS: deployment gated on an environment"
grep -q 'e2e-test' .github/workflows/ci.yml && echo "PASS: e2e stage wired into ci.yml"
grep -q 'required' <(gh api "repos/$(gh repo view --json nameWithOwner --jq .nameWithOwner)/environments" --jq '.environments[].protection_rules[]?.type') \
  && echo "PASS: an environment has protection rules"

ruby -ryaml -e '
d = YAML.safe_load(File.read(".github/workflows/ci.yml"), aliases: true) \
  or abort("FAIL: empty workflow")
jobs = d["jobs"]
expected = %w[build-and-test security-scan dependency-review e2e-test agent-review consolidate publish-summary deploy]
abort("FAIL: jobs are #{jobs.keys}") unless expected.all? { |job| jobs.key?(job) }
abort("FAIL: auditor missing from matrix") unless jobs["agent-review"]["strategy"]["matrix"]["agent"].include?("auditor")
abort("FAIL: consolidate must wait for e2e-test") unless jobs["consolidate"]["needs"].include?("e2e-test")
abort("FAIL: deploy dependency graph is incomplete") unless %w[security-scan dependency-review e2e-test consolidate].all? { |job| jobs["deploy"]["needs"].include?(job) }
puts "PASS: final ci.yml job graph matches the completed lab"'
```

---

## Full implementation checklist

**Agent profiles**

- [ ] `.github/agents/reviewer.agent.md` — low autonomy (`read`, `search`)
- [ ] `.github/agents/auditor.agent.md` — low autonomy (`read`, `search`)
- [ ] `.github/agents/test-runner.agent.md` — medium (`read`, `search`, `edit`, `execute`)
- [ ] `.github/agents/security-scanner.agent.md` — medium (`read`, `search`, `execute`)
- [ ] `.github/agents/orchestrator.agent.md` — coordinator (`read`, `search`, `agent`)

**Instructions**

- [ ] `.github/copilot-instructions.md`
- [ ] `AGENTS.md`
- [ ] `.github/instructions/api.instructions.md`
- [ ] `.github/instructions/workflows.instructions.md`

**Workflows**

- [ ] `.github/workflows/copilot-setup-steps.yml`
- [ ] `.github/workflows/ci.yml` — build, security, dependency review, E2E, agent review, consolidation, and gated deployment

**Guardrails**

- [ ] `.github/hooks/pre-tool-policy.json`
- [ ] `.github/hooks/scripts/pre-tool-policy.sh`
- [ ] `.mcp.json`

Compare against the reference:

```bash
diff -rq completed/.github starter/.github
```

Differences are expected — the guides adapt some commands to this repository's verified
paths. Missing files are not.

---

## Exam notes

### The five agents and their tools

| Agent | Tools | Level |
|-------|-------|-------|
| `reviewer` | `read`, `search` | Low |
| `auditor` | `read`, `search` | Low |
| `test-runner` | `read`, `search`, `edit`, `execute` | Medium |
| `security-scanner` | `read`, `search`, `execute` | Medium |
| `orchestrator` | `read`, `search`, `agent` | Coordinator |

### Complete file path map

```
.github/copilot-instructions.md            Repository instructions
.github/instructions/*.instructions.md     Path-specific (requires applyTo)
AGENTS.md                                  Agent-oriented project docs
.github/agents/*.agent.md                  Agent profiles (requires description)
.github/prompts/*.prompt.md                Reusable prompts
.github/skills/<name>/SKILL.md             Agent skills
.github/hooks/*.json                       Guardrail hooks
.github/workflows/copilot-setup-steps.yml  Cloud agent setup
.mcp.json                                  MCP configuration
~/.copilot/session-state/<id>/events.jsonl Session state
```

### Approval gates

| Gate | Governs |
|------|---------|
| Branch protection | Merging |
| Environment required reviewer | Deploying |
| "Approve and run workflows" | Agent-modified workflow files |

### Deployment security

- `id-token: write` enables OIDC — no stored credentials
- `--allow-tool='shell(curl:*)'` scopes shell access to one command
- `--what-if` previews infrastructure changes before applying

---

## Common pitfalls

**Omitting `applyTo`.** The instruction file will not scope correctly.

**Assuming branch protection covers deployment.** It governs merges only.

**Granting `edit` to the security scanner.** Findings need human triage.

**Storing Azure credentials as long-lived secrets.** Use OIDC.

---

## A word before you take this anywhere near production

What you have built is a **skeleton**. It is shaped correctly, and every concept in it is
real, but the specific contents are teaching material rather than a production configuration.

In a real repository, the agent files, instruction files, and prompts would be written around
*your* architecture, *your* threat model, and *your* team's conventions. The scan list in the
security agent would come from your actual risk register. The review checklist would reflect
the defects your team actually ships. The tool grants would be argued over. None of that can
be generalised into a lab.

So: **use these files to understand the concepts, not as a starting template you copy into a
production repository.** Take the structure — graduated tool permissions, deterministic checks
in the platform and judgement in the agents, artifacts for every handoff, gates a human owns.
Rewrite the contents for the system you are actually protecting.

---

## What you built

A complete agentic pipeline: five specialized agents, four layers of instruction, three
workflows, and guardrails at both the agent and platform boundaries. Every artifact is
reviewable, every high-risk action is gated, and every gate is enforced somewhere the agent
cannot reach.

**Next:** [Exercise 8 — Exam Practice](exercise-08-exam-practice.md)
