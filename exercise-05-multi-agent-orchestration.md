# Exercise 5 — Multi-Agent Orchestration (Domain 5)

**Goal:** run several specialized agents in parallel from a workflow, pass their output
between jobs through artifacts, and consolidate everything into one reviewable summary.

**You will create:** `.github/workflows/agentic-ci.yml`

**Prerequisite:** [Exercise 4](exercise-04-evaluation-and-tuning.md) complete.

**Time:** ~45 minutes

---

## From one agent to a team

Exercise 2 built agents that run when you invoke them from a chat window. That is useful
but manual, and it does not scale to every pull request.

This exercise moves them into CI, which changes three things:

1. **They run automatically**, on every pull request, without anyone remembering to ask.
2. **They run in parallel**, because a review, an audit, and a security scan are
   independent and there is no reason to serialize them.
3. **Their output becomes an artifact**, which — per Exercise 3 — is the only way
   information legitimately travels between jobs.

The pipeline you build has five stages:

```
build-and-test
     ├──> agent-review (matrix: reviewer, security-scanner)
     ├──> security-scan (CodeQL + dependency review)
     └──> e2e-tests
                    └──> consolidate
```

Stages 2, 3, and 4 all depend on stage 1 and run concurrently with each other. Stage 5
waits for all of them.

---

## Step 1 — Create the workflow shell

**Create this file:** `.github/workflows/agentic-ci.yml`

```yaml
name: Agentic CI Pipeline

on:
  pull_request:
    branches: [main]
  workflow_dispatch:
    inputs:
      task:
        description: 'Task description for agents'
        required: false
        type: string

concurrency:
  group: ${{ github.workflow }}-${{ github.head_ref || github.run_id }}
  cancel-in-progress: true

permissions:
  contents: read
  pull-requests: write
  security-events: write
```

### The concurrency block

This is the most-tested configuration in Domain 5, so it is worth taking apart term by term.

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.head_ref || github.run_id }}
  cancel-in-progress: true
```

**`github.workflow`** scopes the group to this workflow. Without it, unrelated workflows
would share a group and cancel each other.

**`github.head_ref`** is the pull request's source branch. Grouping by it means all runs for
one PR share a group — which is what lets a new push cancel the previous run.

**`|| github.run_id`** is the fallback. `head_ref` is only set for pull request events; on a
push event it is empty. Without the fallback every push would land in the same group and
cancel unrelated runs. Since `run_id` is unique per run, the fallback effectively disables
grouping for non-PR triggers, which is the desired behaviour.

**`cancel-in-progress: true`** cancels older runs in the group. When agents push several
commits to a PR in quick succession, only the newest state is worth validating.

#### The trap

If **every** run must complete — a production queue where each run has side effects you
cannot skip — you want the opposite:

```yaml
concurrency:
  group: production-agent-work
  queue: max
```

**Never combine `queue: max` with `cancel-in-progress: true`.** They express contradictory
intentions: queue everything versus discard the stale ones. Exam questions are built on
exactly this contradiction.

### The permissions block

Least privilege again, now for the workflow token:

- `contents: read` — check out the code.
- `pull-requests: write` — post the consolidated summary as a comment.
- `security-events: write` — upload CodeQL results.

No `contents: write`, because nothing here should push commits.

---

## Step 2 — Build and test

Append to the same file:

```yaml
jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6

      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'

      - uses: actions/setup-node@v4
        with:
          node-version: '22'

      - name: Build API
        run: dotnet build src/api/TodoApi.csproj --configuration Release

      - name: Run API Tests
        run: |
          dotnet test src/api/Tests/TodoApi.Tests.csproj \
            --configuration Release \
            --logger "trx;LogFileName=api-results.trx" \
            --results-directory ./test-results

      - name: Install Frontend Dependencies
        run: cd src/frontend && npm ci

      - name: Run Frontend Tests
        run: cd src/frontend && npm test

      - name: Upload Test Results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results
          path: |
            test-results/
            src/frontend/coverage/
          retention-days: 7
```

This stage is deliberately conventional — deterministic checks run first and gate
everything else. There is no point paying for agent analysis of a branch that does not
compile.

Note `if: always()` on the artifact upload. A failing test run is precisely when you want
the results file; without this, the step is skipped on failure.

The `--logger "trx;..."` flag produces a structured result file rather than only console
output, which is what makes the artifact useful to a later job.

---

## Step 3 — The agent review matrix

```yaml
  agent-review:
    needs: [build-and-test]
    runs-on: ubuntu-latest
    permissions:
      contents: read
      copilot-requests: write
    strategy:
      fail-fast: false
      matrix:
        agent: [reviewer, security-scanner]
    steps:
      - uses: actions/checkout@v6

      - uses: actions/setup-node@v7
        with:
          node-version: '22'
          package-manager-cache: false

      - name: Install Copilot CLI
        run: npm install -g @github/copilot

      - name: Run ${{ matrix.agent }} Agent
        env:
          COPILOT_GITHUB_TOKEN: ${{ github.token }}
        run: |
          copilot --agent=${{ matrix.agent }} \
            -p "Analyze the changes in this PR. Focus on your specialization area. Report findings with severity and file paths." \
            --no-ask-user > "${{ matrix.agent }}-report.md"

      - name: Upload Agent Report
        uses: actions/upload-artifact@v7
        with:
          name: ${{ matrix.agent }}-report
          path: ${{ matrix.agent }}-report.md
          retention-days: 7
          if-no-files-found: error
```

> **Both of these agents already exist.** You wrote `reviewer` and `security-scanner` in
> [Exercise 2](exercise-02-tool-use-and-environment.md). A third — `auditor` — joins this
> matrix in [Exercise 7](exercise-07-full-pipeline-integration.md). If a profile named in the
> matrix does not exist, that leg fails, which is why the list and the files must stay in step.

### Why a matrix

One job definition, two parallel executions today and three after Exercise 7. Adding an agent
is a one-word change rather than a copied-and-pasted job. Each leg gets its own runner, its
own log, and its own artifact.

### `fail-fast: false` is essential here

The default is `fail-fast: true`, which cancels every remaining leg as soon as one fails.
That is sensible for a build matrix across OS versions, where one broken combination usually
means the change is broken.

It is wrong for agents. These three are asking independent questions. If the security
scanner errors, you still want the reviewer's findings — they were never contingent on it.
Cancelling them destroys useful work and leaves you with an incomplete picture for no
benefit.

**Rule to remember: matrix agent jobs always set `fail-fast: false`.**

### The CLI invocation

Three details, each a likely exam item:

- **`--agent=NAME`** selects one of your `.github/agents/*.agent.md` profiles. The matrix
  variable feeds it, so the same step runs a different agent per leg.
- **`--no-ask-user`** prevents the hang described in Exercise 4. Non-negotiable in CI.
- **`COPILOT_GITHUB_TOKEN`** supplies authentication, from a secret.

Output is redirected to a per-agent markdown file, then uploaded with
`if-no-files-found: error` so an agent that produces nothing fails the job loudly rather
than passing quietly.

---

## Step 4 — Security scanning and E2E, in parallel

```yaml
  security-scan:
    needs: [build-and-test]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6

      - name: Initialize CodeQL
        uses: github/codeql-action/init@v3
        with:
          languages: csharp, javascript

      - name: Build for CodeQL
        run: dotnet build src/api/TodoApi.csproj

      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v3

      - name: Dependency Review
        uses: actions/dependency-review-action@v4
        if: github.event_name == 'pull_request'
        with:
          fail-on-severity: high

  e2e-tests:
    needs: [build-and-test]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6

      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'

      - uses: actions/setup-node@v4
        with:
          node-version: '22'

      - name: Start API
        run: |
          dotnet run --project src/api/TodoApi.csproj &
          sleep 5

      - name: Install Frontend & Playwright
        run: |
          cd src/frontend
          npm ci
          npx playwright install --with-deps chromium

      - name: Run Playwright Tests
        run: cd src/frontend && npx playwright test
        env:
          PLAYWRIGHT_BASE_URL: http://localhost:3000

      - name: Upload Playwright Report
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: playwright-report
          path: src/frontend/playwright-report/
          retention-days: 7
```

Both declare `needs: [build-and-test]` and neither depends on the other, so all three
stage-2 jobs run concurrently. `needs` expresses ordering; anything not named runs in
parallel.

These are **deterministic** checks alongside the probabilistic agent analysis. CodeQL and
dependency review produce the same answer every run. Agent review does not. A sound
pipeline uses agents to add judgement on top of deterministic scanning — never to replace
it.

---

## Step 5 — Consolidate

```yaml
  consolidate:
    needs: [agent-review, security-scan, e2e-tests]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6

      - name: Download All Reports
        uses: actions/download-artifact@v4
        with:
          path: reports/

      - uses: actions/setup-node@v4
        with:
          node-version: '22'

      - name: Install Copilot CLI
        run: npm install -g @github/copilot

      - name: AI-Consolidated Summary
        env:
          COPILOT_GITHUB_TOKEN: ${{ secrets.PERSONAL_ACCESS_TOKEN }}
        run: |
          copilot -p "Consolidate the review, audit, and security reports in ./reports/ into a single executive summary. Include: overall risk level, blocking issues, advisory items, and a merge recommendation." \
            --allow-tool='read,search' \
            --no-ask-user > consolidated-summary.md

      - name: Post Summary to PR
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const summary = fs.readFileSync('consolidated-summary.md', 'utf8');
            await github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
              body: `## 🤖 Agentic CI Summary\n\n${summary}`
            });

      - name: Write Step Summary
        run: |
          {
            echo "## 🤖 Agentic CI Consolidated Report"
            echo ""
            cat consolidated-summary.md
          } >> "$GITHUB_STEP_SUMMARY"
```

### The handoff

`needs: [agent-review, security-scan, e2e-tests]` waits for all three. Naming the matrix job
once waits for every leg.

`download-artifact` with no `name` fetches everything into `reports/`. This is the artifact
handoff from Exercise 3 in practice: the agent jobs ran on different machines that no longer
exist, and the artifact store is the only thing connecting them.

### `--allow-tool='read,search'`

The consolidation step reads reports and writes a summary. It has no business editing files
or running commands, so its tools are restricted inline. Note that this is a plain `copilot`
invocation without `--agent`, so there is no agent profile constraining it — the flag
supplies the boundary that the profile would otherwise provide.

### Two output destinations

The PR comment is durable and lands where reviewers work. `$GITHUB_STEP_SUMMARY` renders on
the workflow run page for anyone debugging CI. Different audiences, both cheap.

---

## Verify

```bash
test -s .github/workflows/agentic-ci.yml || echo "FAIL: file is empty"

ruby -ryaml -e '
d = YAML.safe_load(File.read(".github/workflows/agentic-ci.yml"), aliases: true) \
  or abort("FAIL: empty file")
jobs = d["jobs"]
%w[build-and-test agent-review consolidate].each do |j|
  abort("FAIL: missing job #{j}") unless jobs.key?(j)
end
abort("FAIL: fail-fast must be false") if jobs["agent-review"]["strategy"]["fail-fast"] != false
abort("FAIL: missing concurrency") unless d["concurrency"]["cancel-in-progress"] == true
puts "PASS: pipeline structure correct"'
```

```bash
grep -q -- "--no-ask-user" .github/workflows/agentic-ci.yml && echo "PASS: no-ask-user present"
grep -q "if-no-files-found: error" .github/workflows/agentic-ci.yml && echo "PASS: artifact guard"
```

> `github.head_ref` and `secrets.PERSONAL_ACCESS_TOKEN` only resolve on GitHub. Local
> validation confirms structure; behaviour requires opening a pull request.

---

## Exam notes

### Concurrency

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.head_ref || github.run_id }}
  cancel-in-progress: true
```

| Component | Purpose |
|-----------|---------|
| `github.workflow` | Prevents cancelling other workflows |
| `github.head_ref` | Groups by PR source branch |
| `\|\| github.run_id` | Fallback when `head_ref` is unset (push events) |
| `cancel-in-progress: true` | Discards stale runs |

Opposite case — every run must complete:

```yaml
concurrency:
  group: production-agent-work
  queue: max
```

Never combine `queue: max` with `cancel-in-progress: true`.

### Matrix

```yaml
strategy:
  fail-fast: false
  matrix:
    agent: [reviewer, security-scanner]
```

`fail-fast: false` keeps independent agents running when one fails.

### Job dependencies and handoff

- `needs:` orders jobs; unlisted jobs run in parallel.
- `needs: [matrix-job]` waits for **all** legs.
- Data moves via `upload-artifact` / `download-artifact`, or `$GITHUB_OUTPUT` for small values.
- **Handoff artifacts beat chat memory** — jobs share no filesystem.

### CLI flags

| Flag | Effect |
|------|--------|
| `--agent=NAME` | Selects a custom agent profile |
| `--no-ask-user` | Prevents interactive hang in CI |
| `--allow-tool='read,search'` | Restricts tools for an unprofiled invocation |
| `COPILOT_GITHUB_TOKEN` | Authentication |

---

## Common pitfalls

**Leaving `fail-fast` at its default.** One agent's failure silently cancels the others.

**Omitting the `|| github.run_id` fallback.** Push events collapse into one group and cancel
each other.

**Expecting files to persist between jobs.** They do not. Upload and download.

**Forgetting `--no-ask-user`.** The job hangs until it times out.

**Replacing deterministic scanning with agents.** CodeQL and dependency review are
reproducible; agent output is not. Run both.

---

## What you built

A pipeline where specialized agents run in parallel on every pull request, their findings
are captured as durable artifacts, and a consolidation step turns them into one summary that
lands on the PR. Exercise 6 adds the enforcement layer that constrains what these agents can
do while running.

**Next:** [Exercise 6 — Guardrails & Accountability](exercise-06-guardrails-and-accountability.md)
