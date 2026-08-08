# GH-600 Lab Guides

Self-paced, step-by-step guides for transforming a conventional CI pipeline into an
agentic CI/CD pipeline, built around the sample Todo application in the
[`gh-600-lab-starter`](https://github.com/sameeraman/gh-600-lab-starter) repository.

Each guide is written to be followed independently: it tells you exactly which file to
create, what to put in it, why that file exists, how to verify it, and what the
certification exam expects you to remember.

## What's included in the starter repository

The [`gh-600-lab-starter`](https://github.com/sameeraman/gh-600-lab-starter) repository
provides:

- A React and Vite frontend for the sample Todo application.
- An ASP.NET Core 8 Web API with Entity Framework Core.
- Unit tests and Playwright end-to-end tests.
- Bicep templates for provisioning the Azure environment.
- Configuration for Azure Static Web Apps, App Service, managed identity, and Azure SQL
    Database.
- A conventional GitHub Actions pipeline with the GH-500 security baseline.

## What you will do in the lab

Using the guides in the
[`gh-600-lab-instructions`](https://github.com/sameeraman/gh-600-lab-instructions)
repository, you will:

- Deploy the sample application to Azure using OIDC federation and managed identity.
- Create repository-wide and path-specific instructions that give agents durable context.
- Build specialized custom agents with deliberately limited tool permissions.
- Configure MCP servers and a cloud-agent environment.
- Analyze agent memory and session state, then diagnose and tune agent failures.
- Run unit and Playwright end-to-end tests as evidence in an agentic CI/CD pipeline.
- Orchestrate parallel agent reviews and consolidate their reports through workflow artifacts.
- Add preventive policy hooks, audit trails, and least-privilege controls.
- Gate production deployment on human approval, then run smoke tests and an agent audit.
- Test your GH-600 knowledge with exam-style questions and a final reference guide.

## At a glance

- **Total duration:** approximately 5 hours 15 minutes, including lab preparation.
- **Environment:** live Azure resources in your own subscription and Microsoft Entra tenant.
- **Estimated cost:** approximately US$10–20 per month until you complete the
    [teardown](lab-preparation.md#14-teardown).
- **Repository visibility:** use a public fork unless your account has a GitHub Code Security
    licence. See [why the lab uses a public fork](lab-preparation.md#42-enable-actions-and-the-dependency-graph).

## Before you start

[Lab Preparation](lab-preparation.md) — fork and clone the starter, then complete the one-time
Azure and GitHub setup: resource group, OIDC federated identity, app registrations, repository
settings, and secrets. The exercises assume the sample application is already deployed, which
is what this guide gets you.

### Prerequisites summary

- Azure CLI 2.60 or later, GitHub CLI 2.40 or later, and Git 2.30 or later. See the
    [required tools](lab-preparation.md#2-prerequisites).
- Permission to create Microsoft Entra app registrations.
- Azure **Owner**, or **Contributor + User Access Administrator**, so you can create role
    assignments. Review the full [account and permission requirements](lab-preparation.md#2-prerequisites).
- Access to GitHub Copilot for the IDE and Copilot CLI activities used in the exercises.

## Assumed baseline knowledge

This lab assumes working knowledge of Git and GitHub, including repositories, branches, commits,
pull requests, and GitHub Actions. It also builds on the GitHub Copilot knowledge covered by
GH-300. You should be comfortable using GitHub Copilot in the IDE and understand its core
features before starting the GH-600 exercises.

The starter repository already contains the application source, Bicep infrastructure, and a
conventional GitHub Actions pipeline. It also includes the CodeQL, dependency-review, and secret-
scanning controls normally covered by GH-500. This lab treats those controls as an established
baseline so that the exercises can stay focused on GH-600 topics: custom agents, tools, MCP,
orchestration, memory, evaluation, guardrails, and accountable deployment.

The exercises do not reteach Git, GitHub, GH-300 concepts, or GH-500 controls. On public forks,
CodeQL and dependency review run normally after Actions and the dependency graph are enabled.
On private forks without GitHub Code Security, the baseline workflow clearly skips unsupported
scans so that the lab can remain focused on GH-600.

## Guides

Work through them in order — each builds on the last.

| Guide | Domain | Time | What you build |
|-------|--------|------|----------------|
| [0 — Lab Preparation](lab-preparation.md) | Prerequisites | ~30 min | Azure and GitHub lab environment |
| [1 — Prepare Agent Architecture](exercise-01-agent-architecture.md) | 1 | ~15 min | `.github/copilot-instructions.md` |
| [2 — Tool Use & Environment](exercise-02-tool-use-and-environment.md) | 2 | ~55 min | Custom agents, MCP config, cloud-agent setup |
| [3 — Memory, State & Execution](exercise-03-memory-state-execution.md) | 3 | ~30 min | *(analysis)* Session logs, durable state |
| [4 — Evaluation, Error Analysis & Tuning](exercise-04-evaluation-and-tuning.md) | 4 | ~25 min | *(analysis)* Failure diagnosis, tuning order |
| [5 — Multi-Agent Orchestration](exercise-05-multi-agent-orchestration.md) | 5 | ~45 min | `.github/workflows/agentic-ci.yml` |
| [6 — Guardrails & Accountability](exercise-06-guardrails-and-accountability.md) | 6 | ~35 min | Pre-tool policy hooks |
| [7 — Full Pipeline Integration](exercise-07-full-pipeline-integration.md) | — | ~50 min | Auditor agent, instructions, prompts, deployment |
| [8 — Exam Practice & Cheat Sheet](exercise-08-exam-practice.md) | — | ~30 min | *(revision)* 10 questions + reference |

Exercises 3 and 4 create no files. They cover judgement the exam tests through log reading
and scenario matching, and the concepts they establish are applied directly in 5 and 6.

## What you end up with

```
.github/
├── copilot-instructions.md          Repository-wide conventions
├── agents/                          5 agents, graduated capability
│   ├── reviewer.agent.md            read, search                    (ex 2)
│   ├── test-runner.agent.md         read, search, edit, execute     (ex 2)
│   ├── security-scanner.agent.md    read, search, execute           (ex 2)
│   ├── orchestrator.agent.md        read, search, agent             (ex 2)
│   └── auditor.agent.md             read, search                    (ex 7)
├── instructions/                    Path-scoped rules
├── prompts/                         Reusable task prompts
├── hooks/                           Preventive guardrails
└── workflows/
    ├── copilot-setup-steps.yml      Cloud agent environment                    
    └── ci.yml                       Build, test, e2e, approval, deploy, smoke, agent review pipeline
AGENTS.md                            Agent-oriented project docs
.mcp.json                            External tool servers
```

## The sample application

Fork [`sameeraman/gh-600-lab-starter`](https://github.com/sameeraman/gh-600-lab-starter), then
follow [section 4 of the preparation guide](lab-preparation.md#4-fork-and-clone-the-starter-repository).
All paths in these guides are relative to the root of your cloned fork. The properly deploy application will look like below. 

![app-frontend-preview](images/SCR-20260802-hpem.png)


```
gh-600-lab-starter/
├── src/api/          ASP.NET Core 8 Web API (EF Core, Azure SQL in production)
├── src/frontend/     React + Vite single-page app
├── infra/            Azure Bicep infrastructure
└── .github/          Workflows, and the agent configuration you are about to build
```

### What a fork does not inherit

The fork contains the source and workflow files, but it does **not** inherit repository secrets,
Actions variables, environments, environment protection rules, or OIDC federated credentials.
The preparation guide recreates each of these for your fork and derives the OIDC subject from
that fork's immutable owner and repository IDs.

Enable Actions and the dependency graph before the first workflow run. Keep your lab branches
and pull requests within your own fork: GitHub correctly withholds repository secrets from
workflows triggered by pull requests from untrusted external forks, so those workflows cannot
perform the authenticated Azure deployment used by this lab.

## How to use these guides

Work through them in order. Each file-producing step ends with a verification block; do not
move on until it passes.

Verify with `test -s` rather than `test -f`. Creating a file in an editor does not write it
to disk until you save, and `-f` passes on an empty file — which is the most common way to
lose time in this lab.

If setup or the first deployment fails, use the preparation guide's
[troubleshooting table](lab-preparation.md#13-if-the-first-run-fails). It covers common OIDC,
permissions, private-repository, provider-registration, and deployment-approval failures.

> **Clean up when you finish.** Azure resources continue to incur charges until they are
> removed. Deleting only the resource group does not remove the Microsoft Entra app
> registrations or the GitHub secrets, variables, and environment. Follow the complete
> [teardown procedure](lab-preparation.md#14-teardown).

## Conventions used in these guides

- **Create this file** — a file you must add to the repository.
- **Why this matters** — the reasoning behind the step.
- **Exam notes** — facts that are commonly tested.
- **Verify** — a command that proves the step worked.
