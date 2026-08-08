# Exercise 5 — Multi-Agent Orchestration (Domain 5)

**Goal:** run several specialized agents in parallel from a workflow, pass their output
between jobs through artifacts, and consolidate everything into one reviewable summary.

**You will modify:** `.github/workflows/ci.yml`

**Prerequisite:** [Exercise 4](exercise-04-evaluation-and-tuning.md) complete.

**Time:** ~45 minutes

---

## From one agent to a team

Exercise 2 built agents that run when you invoke them from a chat window. That is useful
but manual, and it does not scale to every pull request.

This exercise moves them into CI, which changes three things:

1. **They run automatically**, on every pull request, without anyone remembering to ask.
2. **They run in parallel**, because code review, agent security review, and deterministic
  security scanning are independent and there is no reason to serialize them.
3. **Their output becomes an artifact**, which — per Exercise 3 — is the only way
   information legitimately travels between jobs.

The starter repository already has one authoritative workflow with build, security,
dependency-review, and deployment jobs. You will extend that same file with agent review and
artifact-based consolidation:

```
build-and-test
     ├──> agent-review (matrix: reviewer, security-scanner) ──┐
     ├──> security-scan ──────────────────────────────────────┤
     └──> dependency-review ──────────────────────────────────┴──> consolidate
                                                                      ├──> publish-summary (PR)
                                                                      └──> deploy (main)
```

The three review jobs depend on `build-and-test` and run concurrently. `consolidate` waits for
all of them. Exercise 7 later adds `e2e-test` to this graph and makes both consolidation and
deployment wait for its evidence.

---

## Step 1 — Prepare the existing workflow

**Open this existing file:** `.github/workflows/ci.yml`

Do not create another workflow. The starter already deploys from `ci.yml`; a second CI file
would duplicate builds and create two independent authorities for the same commit.

Change the workflow name from `Basic CI` to `CI`, replace the top-level permissions block with
an empty default, and add concurrency immediately before it:

```yaml
name: CI

concurrency:
  group: ${{ github.workflow }}-${{ github.head_ref || github.run_id }}
  cancel-in-progress: true

permissions: {}
```

Keep the existing `pull_request` and `push` triggers. `permissions: {}` denies every permission
by default; each job then receives only the scopes it declares. The starter jobs already use
job-level permissions, and the new jobs below do the same.

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

### The permissions default

An empty top-level default prevents a new job from silently inheriting broad authority. For
example, `agent-review` gets `contents: read` and `copilot-requests: write`, while
`publish-summary` gets `actions: read` and `pull-requests: write`. No agent job can push code.

---

## Step 2 — Preserve deterministic test evidence

The starter already contains `build-and-test`; do not duplicate it. Append this step after
`Run Frontend Tests` in that job:

```yaml
      - name: Upload Test Results
        uses: actions/upload-artifact@v7
        if: always()
        with:
          name: test-results
          path: |
            test-results/
            src/frontend/coverage/
          retention-days: 7
```

The deterministic checks still run first and gate everything else. There is no point paying
for agent analysis of a branch that does not compile.

Note `if: always()` on the artifact upload. A failing test run is precisely when you want
the results file; without this, the step is skipped on failure.

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
- **`COPILOT_GITHUB_TOKEN`** supplies authentication from the job-scoped `${{ github.token }}`.

Output is redirected to a per-agent markdown file, then uploaded with
`if-no-files-found: error` so an agent that produces nothing fails the job loudly rather
than passing quietly.

---

## Step 4 — Keep the existing deterministic gates

Do not replace the starter's `security-scan` or `dependency-review` jobs. They already use
the completed example's visibility checks, private-repository fallback, and current action
versions (`github/codeql-action@v4` and `actions/dependency-review-action@v5`). Both declare
`needs: [build-and-test]`, just like `agent-review`, so all three run in parallel.

These are **deterministic** checks alongside the probabilistic agent analysis. CodeQL and
dependency review produce the same answer every run. Agent review does not. A sound
pipeline uses agents to add judgement on top of deterministic scanning — never to replace
it.

Exercise 7 adds the full runner-hosted `e2e-test` job from the completed repository. Keeping
that implementation in one exercise avoids first creating a simplified job and then replacing
it with a different one.

---

## Step 5 — Consolidate and publish

```yaml
  consolidate:
    needs: [dependency-review, agent-review, security-scan]
    runs-on: ubuntu-latest
    permissions:
      actions: read
      copilot-requests: write
    steps:
      - name: Download All Reports
        uses: actions/download-artifact@v8
        with:
          path: reports/

      - name: Normalize Reports Directory Permissions
        run: |
          mkdir -p reports
          chmod -R u+rwX reports

      - uses: actions/setup-node@v7
        with:
          node-version: '22'
          package-manager-cache: false

      - name: Install Copilot CLI
        run: npm install -g @github/copilot

      - name: AI-Consolidated Summary
        env:
          COPILOT_GITHUB_TOKEN: ${{ github.token }}
        run: |
          copilot -p "Read the report files already present under ./reports/ and return only a concise Markdown executive summary for the pull request comment. Begin directly with a level-two heading. Do not wrap the response in a Markdown code fence. Do not create files or run shell commands. Include: overall risk level, blocking issues, advisory items, and a merge recommendation." \
            --allow-tool='read,search' \
            --silent \
            --stream=off \
            --no-ask-user > reports/executive-summary.md

      - name: Upload Consolidated Summary
        uses: actions/upload-artifact@v7
        with:
          name: executive-summary
          path: reports/executive-summary.md
          retention-days: 7
          if-no-files-found: error

      - name: Write Step Summary
        run: |
          {
            echo "## 🤖 Agentic CI Consolidated Report"
            echo ""
            cat reports/executive-summary.md
          } >> "$GITHUB_STEP_SUMMARY"

  publish-summary:
    needs: [consolidate]
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    permissions:
      actions: read
      pull-requests: write
    steps:
      - name: Download Consolidated Summary
        uses: actions/download-artifact@v8
        with:
          name: executive-summary
          path: reports/

      - name: Post Summary to PR
        uses: actions/github-script@v8
        with:
          script: |
            const fs = require('fs');
            const summary = fs.readFileSync('reports/executive-summary.md', 'utf8');
            await github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
              body: `## 🤖 Agentic CI Summary\n\n${summary}`
            });
```

### The handoff

`needs: [dependency-review, agent-review, security-scan]` waits for every review gate. Naming
the matrix job once waits for every matrix leg. Exercise 7 adds `e2e-test` to this list.

`download-artifact` with no `name` fetches every available artifact into `reports/`. This is the artifact
handoff from Exercise 3 in practice: the agent jobs ran on different machines that no longer
exist, and the artifact store is the only thing connecting them.

### `--allow-tool='read,search'`

The consolidation step reads reports and writes a summary. It has no business editing files
or running commands, so its tools are restricted inline. Note that this is a plain `copilot`
invocation without `--agent`, so there is no agent profile constraining it — the flag
supplies the boundary that the profile would otherwise provide.

### Two output destinations

The separate `publish-summary` job receives only the permission needed to post the PR comment.
The comment is durable and lands where reviewers work; `$GITHUB_STEP_SUMMARY` renders on the
workflow run page for anyone debugging CI.

---

## Step 6 — Make deployment wait for the review

Find the existing `deploy` job and add `consolidate` to its dependencies:

```yaml
  deploy:
    needs: [build-and-test, security-scan, dependency-review, consolidate]
```

This keeps one authority graph inside `ci.yml`. On pull requests, the review jobs gate merging.
On a push to `main`, deployment also waits for the same evidence and the production environment
approval configured during lab preparation.

---

## Verify

```bash
test -s .github/workflows/ci.yml || echo "FAIL: file is empty"

ruby -ryaml -e '
d = YAML.safe_load(File.read(".github/workflows/ci.yml"), aliases: true) \
  or abort("FAIL: empty file")
jobs = d["jobs"]
%w[build-and-test security-scan dependency-review agent-review consolidate publish-summary deploy].each do |j|
  abort("FAIL: missing job #{j}") unless jobs.key?(j)
end
abort("FAIL: fail-fast must be false") if jobs["agent-review"]["strategy"]["fail-fast"] != false
abort("FAIL: missing concurrency") unless d["concurrency"]["cancel-in-progress"] == true
abort("FAIL: deploy must wait for consolidate") unless jobs["deploy"]["needs"].include?("consolidate")
puts "PASS: pipeline structure correct"'
```

```bash
grep -q -- "--no-ask-user" .github/workflows/ci.yml && echo "PASS: no-ask-user present"
grep -q "if-no-files-found: error" .github/workflows/ci.yml && echo "PASS: artifact guard"
! grep -q "PERSONAL_ACCESS_TOKEN" .github/workflows/ci.yml && echo "PASS: uses scoped GITHUB_TOKEN"
```

> GitHub expressions and token permissions only resolve on GitHub. Local validation confirms
> structure; behaviour requires opening a pull request.

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
